# Batching

* Batching = processing multiple inputs in one LLM call
* Reduces API calls → improves latency
* Saves prompt overhead (instructions repeated once)
* Core token usage remains mostly same
* Use batch size of 3–5 inputs
* Always use clear separators and unique IDs
* Enforce structured output (JSON array)
* Avoid batching very large inputs (token limits)
* Improves consistency, not accuracy
