# 🟢 Nodes

#### <mark style="color:purple;background-color:purple;">1.</mark> <mark style="color:purple;background-color:purple;"></mark><mark style="color:purple;background-color:purple;">`parse_and_plan`</mark> <a href="#id-1-parse_and_plan" id="id-1-parse_and_plan"></a>

* Reads: `user_requirement`.
* Does:
  * <mark style="color:purple;background-color:purple;">**Calls the agent LLM with**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**`PARSE_REQUIREMENT_PROMPT`**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**to get structured fields**</mark> (skills, location, experience, role, ranges, etc.). On JSON failure, falls back to a minimal parse.
  * Optionally corrects location via `correct_location` and records changes in `corrections`.
  * Resets iteration-related fields: `iteration` → `0`, clears `candidates_pool` / `evaluated_candidates` / `strong_count`, initializes `search_queries_used` if needed.
  * Calls the LLM with `GENERATE_QUERIES_PROMPT` to produce initial search queries; appends them to `search_queries_used`.
* Writes: `parsed_requirements`, `corrections`, `iteration`, `candidates_pool`, `evaluated_candidates`, `strong_count`, `search_queries_used`, `agent_trace`.

***

#### &#x20;<mark style="color:purple;background-color:purple;">2.</mark> <mark style="color:purple;background-color:purple;"></mark><mark style="color:purple;background-color:purple;">**`execute_search`**</mark> <a href="#id-2-execute_search" id="id-2-execute_search"></a>

* <mark style="color:purple;background-color:purple;">**Reads:**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**`parsed_requirements`**</mark><mark style="color:purple;background-color:purple;">**,**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**`search_queries_used`**</mark><mark style="color:purple;background-color:purple;">**, existing**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**`candidates_pool`**</mark><mark style="color:purple;background-color:purple;">**.**</mark>
* <mark style="color:purple;background-color:purple;">**Does: For each query in**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**`search_queries_used`**</mark><mark style="color:purple;background-color:purple;">**, calls**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**`search_candidates`**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**(Chroma + filters)**</mark><mark style="color:purple;background-color:purple;">.</mark> Dedupes by `candidate_id` against IDs already in the pool; appends only new hits.
* Writes: Extends `candidates_pool`, appends to `agent_trace`.

***

#### <mark style="color:purple;background-color:purple;">**3.**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**`evaluate_candidates`**</mark> <a href="#id-3-evaluate_candidates" id="id-3-evaluate_candidates"></a>

* Reads: `parsed_requirements`, `candidates_pool`, `evaluated_candidates`.
* Does: Invokes `evaluate_all_candidates` with the agent LLM, only evaluating candidates not already in `evaluated_candidates`. Updates strong match count: `fit_score >= STRONG_MATCH_THRESHOLD`.
* Writes: Extends `evaluated_candidates`, sets `strong_count`, `agent_trace`.

***

#### <mark style="color:purple;background-color:purple;">4.</mark> <mark style="color:purple;background-color:purple;"></mark><mark style="color:purple;background-color:purple;">`quality_check`</mark> <a href="#id-4-quality_check" id="id-4-quality_check"></a>

* <mark style="color:purple;background-color:purple;">**We not passed the entire resume as it would have become too lengthy, we kept only the keywords from the search string and removed the remaining, passed it as keywords only so that the prompt does not become too long**</mark>
* <mark style="color:purple;background-color:purple;">**Made it a batch call**</mark>
* Reads: `iteration`, `strong_count`, `evaluated_candidates`, parsed requirement context.
* Does:
  * If `strong_count >= MIN_STRONG_MATCHES` → sets `_route = "sufficient"`.
  * Else if next iteration would exceed `MAX_AGENT_ITERATIONS` → sets `_route = "sufficient"` (must finalize).
  * Else calls the agent LLM with `QUALITY_CHECK_PROMPT` (score distribution, counts, etc.) and parses JSON for `decision` and `gap_analysis`; stores `_route` and `_gap_analysis`.
* Writes: `_route`, optionally `_gap_analysis`, `agent_trace`.

***

#### <mark style="color:purple;background-color:purple;">**5.**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**`refine_search`**</mark> <a href="#id-5-refine_search" id="id-5-refine_search"></a>

* <mark style="color:purple;background-color:purple;">**We used LLM for this, we set it as a recruiter persona, we told that that it needs to work on this query, you have already tried this queries, and now we need to increase horizon of the search**</mark>
* Runs only when routing is `need_more` after `quality_check`.
* Reads: `parsed_requirements`, `_gap_analysis`, `search_queries_used`, `strong_count`.
* Does: Increments `iteration`. Calls the agent LLM with `REFINE_QUERIES_PROMPT` (previous queries + gap text) to produce new queries. On parse failure, uses a small broadened fallback list.
* Writes: Replaces `search_queries_used` with the new query list (next `execute_search` uses only these), `iteration`, `agent_trace`.

***

#### <mark style="color:purple;background-color:purple;">**6.**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**`rank_and_report`**</mark> <a href="#id-6-rank_and_report" id="id-6-rank_and_report"></a>

* Reads: `evaluated_candidates`, `parsed_requirements`, `iteration`, `search_queries_used`, `strong_count`.
* Does: If no evaluated candidates, sets an empty `final_output` and returns. Otherwise sorts by `fit_score`, takes up to 20 for the prompt, builds blocks from `RANKED_CANDIDATE_BLOCK_TEMPLATE`, calls the ranking LLM with `RANK_CANDIDATES_PROMPT`. Parses JSON rankings; on failure, builds a simple top-10 from sort order. Enriches rows with metadata from the evaluation pool.
* Writes: `final_output` (`rankings`, `total_evaluated`, `strong_matches`, `iterations_used`, `queries_used`), `agent_trace`.
