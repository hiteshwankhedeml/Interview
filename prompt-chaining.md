# Prompt Chaining

* <mark style="color:purple;background-color:purple;">**Breaking a complex task into multiple sequential prompts.**</mark>
* <mark style="color:purple;background-color:purple;">**Output of one prompt becomes input to the next.**</mark>
* **Simple example**
  * Prompt 1: Extract key points from a document.
  * Prompt 2: Summarize the extracted points.
  * Prompt 3: Generate final answer from the summary.
* **How to implement**
  * Step 1: Define each task as a separate prompt.
  * Step 2: Execute prompts in order.
  * Step 3: Pass output of step _n_ as input to step _n+1_.
  * Step 4: Validate or format output at each step.
* **Implementation styles**
  * Sequential function calls.
  * Pipeline or chain abstraction (e.g., LangChain).
  * Orchestrated via code or workflow engine.
* **One-line summary**
  * Prompt chaining = **divide, prompt, pass, repeat**.
