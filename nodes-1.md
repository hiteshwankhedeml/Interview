# 🟢 Nodes

| Node                        | What it reads (idea)                          | What it writes (idea)                                                                        |
| --------------------------- | --------------------------------------------- | -------------------------------------------------------------------------------------------- |
| **① `parse_and_plan`**      | Recruiter NL string                           | Structured requirement, ES / hybrid query spec, resets session counters for a **new search** |
| **② `execute_search`**      | Query spec, `seen_candidate_ids`              | New hits merged in order; never re-append excluded IDs                                       |
| **③ `prepare_eval_window`** | Ordered IDs, cursor, `pending_tail`           | Next batch of up to **10** IDs to score, or signals **shortfall** (need relax)               |
| **④ `relax_search`**        | Current query + gap / shortfall context       | Looser ES query; increments **`relax_count`** (cap **2** per query)                          |
| **⑤ `evaluate_page`**       | That page’s CV payloads + requirement summary | Per-candidate scores, strengths/gaps; append to evaluated history                            |
| **⑥ `human_checkpoint`**    | Latest page result                            | **Interrupt** — UI shows page; graph pauses with full state saved                            |
| **⑦ `rank_and_report`**     | Evaluated pool (when user finalizes)          | Shortlist / narrative for export or UI                                                       |

Router after **⑥** (not a separate “LLM node”): **`next_10`**, **`new_search`**, **`finalize`** — only those human intents; **no** direct “relax” button (widening is decided inside **③**).
