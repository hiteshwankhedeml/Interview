# 🟢 JD Generation Agent

<mark style="color:purple;background-color:purple;">**System prompt (summary):**</mark>

* <mark style="color:purple;background-color:purple;">**Persona: Expert JD writer**</mark>
* Inputs baked in: user reqs, company context, analysis patterns, sample JDs
* Output: Exactly 4 sections — Education, Experience, Roles & Responsibilities, Skills
* No intro/outro / meta commentary

<mark style="color:purple;background-color:purple;">**Input (first generation):**</mark>

* <mark style="color:purple;background-color:purple;">**User requirements (job title, education, experience, functional area, skills, additional input)**</mark>
* <mark style="color:purple;background-color:purple;">**Company context (defaults to Reliance Jio profile)**</mark>
* <mark style="color:purple;background-color:purple;">**Market analysis insights from the Analysis Agent**</mark>
* Top 5 retrieved JDs as reference (1000 chars each)

<mark style="color:purple;background-color:purple;">**Input (regeneration):**</mark>

* <mark style="color:purple;background-color:purple;">**Same as first generation plus validation feedback: quality score, issues, suggestions (from the last validation pass)**</mark>

Output

* `full_content` — full markdown JD
* `sections` — Education / Experience / Roles / Skills
* `job_title`

Streaming

* Tokens to UI in batches of 3
* \~50ms delay → typewriter feel

<mark style="color:purple;background-color:purple;">**Self-reflection:**</mark>

* <mark style="color:purple;background-color:purple;">**Prompt asks: meet requirements? obvious issues? improvements? professional / complete?**</mark>
* Returns:
  * `self_assessment` (text)
  * <mark style="color:purple;background-color:purple;">**`should_revise`**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**(keyword score: e.g. revise/fix/problem vs good/complete/satisfactory)**</mark>
