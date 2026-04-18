---
layout:
  width: default
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: false
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
---

# 🟢 Nodes

gpt-4o used throughout

| Graph id / step              | LLM? | Calls                                | Updates state (main keys)                           |
| ---------------------------- | ---- | ------------------------------------ | --------------------------------------------------- |
| `ingestion`                  | No   | `IngestionAgent.ingest()`            | `ingestion_result`                                  |
| `description`                | Yes  | `DescriptionAgent.generate()`        | `description_result`                                |
| `compliance`                 | Yes  | `ComplianceAgent.check_compliance()` | `compliance_result`                                 |
| `quality`                    | Yes  | `QualityAgent.score()`               | `quality_result`                                    |
| `decision`                   | Yes  | `DecisionAgent.decide()`             | `decision_result`, `all_attempts`, `final_decision` |
| `increment_attempt`          | No   | (pure Python)                        | `attempt_number`, clears prior attempt outputs      |
| `should_regenerate` (router) | No   | `graph_pipeline.should_regenerate()` | N/A (routing only)                                  |

***

### 1. `ingestion` (`ingestion_node`)

* **File:** `workflow/nodes.py` → `agents/ingestion_agent.py`
* **Usage:**
  * Runs at **graph entry** (every full `invoke`).
  * Reads `state["uploaded_images"]`, `state["form_data"]`.
  * Encodes images as **PNG base64** strings; **strips** string fields into a fixed attribute dict.
* **Prompt:** **None** (no LLM). Pillow + base64 only.

***

### 2. `description` (`description_node`)

* **File:** `workflow/nodes.py` → `agents/description_agent.py`
* **Usage:**
  * Builds catalog copy from `ingestion_result` (attributes + images).
  * Uses `state["temperature"]` for `ChatOpenAI`.
  * Loads **brand/category guidelines** via `get_brand_guidelines` / `get_category_guidelines` and injects them into the human prompt.
* **System message (fixed):**
  * _`You are a professional product description writer. Always respond with valid JSON only.`_
* **Human message (dynamic):** `_build_prompt(...)` concatenates:
  * Role: _expert product description writer for e-commerce_.
  * **Required attributes:** category, brand, material, gender.
  * **Optional block:** color (or instruction to analyze image), size, style, season, care, additional features.
  * **Brand guidelines:** tone, voice, do's, don'ts (from rule engine).
  * **Category guidelines:** tone, common features.
  * If images exist: _analyze product images and incorporate visual details_.
  * **JSON schema** requested: `title`, `bullet_points` (4 items), `description` (150–300 words, bullets for expectations).
  * **Constraints:** SEO-friendly title, no medical claims / false promises, match brand/category.
* **Multimodal:** If images exist, `HumanMessage` `content` is a list: `[{type: text, text: prompt}, {type: image_url, image_url: {url: data:image/png;base64,...}}, ...]`.

***

### 3. `compliance` (`compliance_node`)

* **File:** `workflow/nodes.py` → `agents/compliance_agent.py`
* **Usage:**
  * Input: `description_result` + `ingestion_result["attributes"]`.
  * Reloads brand/category guidelines for **forbidden words** and **compliance\_rules** list.
* **System message (fixed):**
  * _`You are a compliance checker. Always respond with valid JSON only.`_
* **Human message (dynamic):** `_build_compliance_prompt(...)` includes:
  * Role: _compliance checker for product descriptions_.
  * **Attributes:** material, gender (required); color/size with "optional / not specified" wording.
  * **Brand:** forbidden words, don'ts.
  * **Category:** bullet list of `compliance_rules`.
  * **Copy under test:** title, bullets, full description.
  * **Checklist:** forbidden phrases, medical/misleading claims, contradictions vs attributes, missing **required** material/gender in copy, tone mismatch, other violations.
  * **IMPORTANT block:** color/size optional; no violation for missing them; contradiction only if specified vs copy conflicts.
  * **JSON schema:** `is_compliant`, `violations[]` (type, severity, description, location), `feedback`.

***

### 4. `quality` (`quality_node`)

* **File:** `workflow/nodes.py` → `agents/quality_agent.py`
* **Usage:**
  * Input: same `description_result` + attributes; brand/category **tone** from guidelines for alignment scoring.
* **System message (fixed):**
  * _`You are a quality assessor. Always respond with valid JSON only.`_
* **Human message (dynamic):** `_build_quality_prompt(...)` includes:
  * Role: _quality assessor for product descriptions_.
  * **Attributes:** category, brand, material, size, color, gender.
  * **Brand / category:** tone (and voice for brand).
  * **Copy under test:** title, bullets, description.
  * **Dimensions (0–100):** clarity, completeness, readability, SEO fit, brand alignment, accuracy.
  * **JSON schema:** `scores{...}`, `overall_score`, `feedback{strengths, weaknesses, recommendations}`.
  * **Note:** overall score described as weighted average (clarity, completeness, accuracy emphasized).

***

### 5. `decision` (`decision_node`)

* **File:** `workflow/nodes.py` → `agents/decision_agent.py`
* **Usage:**
  * Input: `quality_result`, `compliance_result`, `attempt_number`, `max_attempts` from state.
  * Appends one **attempt bundle** to `all_attempts` and sets `final_decision` to the model's `decision` string (used by router).
* **System message (fixed):**
  * _`You are a decision agent. Always respond with valid JSON only.`_
* **Human message (dynamic):** `_build_decision_prompt(...)` includes:
  * Role: _decision agent for product description approval_.
  * **Attempt:** current vs max.
  * **Quality:** overall score, strengths, weaknesses (from quality agent output).
  * **Compliance:** compliant flag, violation count, violation lines (type, description, severity).
  * **Options:** APPROVE (high quality, typically >75, compliant); REGENERATE (improvable / fixable issues, attempts left); FLAG\_FOR\_REVIEW (critical violations, very low quality <50, or max attempts).
  * **Guidance bullets:** severity, score bands, prioritize compliance over quality, after max attempts favor flag.
  * **JSON schema:** `decision` (`APPROVE|REGENERATE|FLAG_FOR_REVIEW`), `reasoning`, `confidence` 0–1, `recommendations[]`.

***

### 6. `increment_attempt` (`increment_attempt_node`)

* **File:** `workflow/nodes.py`
* **Usage:**
  * Runs only when router returns **`regenerate`** (see below).
  * Increments `attempt_number`; sets `description_result`, `compliance_result`, `quality_result`, `decision_result` to **`None`** for the next pass.
* **Prompt:** **None** (no LLM).

***

### 7. Router: `should_regenerate` (not a graph node with state merge)

* **File:** `workflow/graph_pipeline.py`
* **Usage:** `add_conditional_edges("decision", should_regenerate, {...})`.
* **Logic (Python only):**
  * If `state["final_decision"] == "REGENERATE"` **and** `state["attempt_number"] < state["max_attempts"]` → return **`"regenerate"`** → edge to `increment_attempt`.
  * Else → **`"end"`** → `END`.
* **Prompt:** **None**.

***

### JSON parsing (all LLM agents)

* **Pattern:** `response.content` → regex `\{.*\}` (DOTALL) → `json.loads`.
* **Implication:** Model must emit a JSON object in the reply; stray text outside braces can still match greedily—see code in each agent for edge cases.

***

### Model / temperature (defaults in code)

* **DescriptionAgent:** configurable model string (default in class), `temperature` from **workflow state** (UI).
* **ComplianceAgent, QualityAgent, DecisionAgent:** same default model family in class, **`temperature=0.3`** when constructed from nodes.

***

### Related docs

* **High-level story:** `Project Explain.md`
* **Graph wiring:** `workflow/graph_pipeline.py`, `workflow/state.py`
