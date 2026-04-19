# 🟢 Infinite Loop

| Guard                                   | What it prevents                                                                                                                                               |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`relax_count` ≤ 2**                   | Endless **shortfall → relax → search** for the same wording; after two widens we **terminal** (message + `new_search` / `finalize` only).                      |
| **Per-page low fit**                    | If **every** score on **that page** is below threshold, **stop** the run for this query (no automatic “keep digging”); human can still start **`new_search`**. |
| **`seen_candidate_ids` + ES exclusion** | Relax / new fetch cannot keep re-scoring the **same** people in a circle.                                                                                      |
| **Prepare is the only “more” entry**    | Recruiter never jumps straight to relax; **③** decides buffer vs widen, so you do not get alternating widen / shrink with no progress rule.                    |

Together: **finite relax budget**, **semantic stop on useless pages**, **dedupe across fetches**, **single choke point for “more”**.
