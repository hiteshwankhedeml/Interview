# 🟢 State Management

* <mark style="color:purple;background-color:purple;">**Single shared state type:**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**`JDGenerationState`**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**(**</mark><mark style="color:purple;background-color:purple;">**`TypedDict`**</mark><mark style="color:purple;background-color:purple;">**)**</mark>
* **Reducer fields:** `stream_messages`, `execution_log` — **append** across nodes (not overwrite)

```python
class JDGenerationState(TypedDict):
    user_inputs: dict           # Form inputs from the user
    retrieved_jds: list         # JDs found by Research Agent
    analysis_results: dict      # Market patterns from Analysis Agent
    generated_jd: dict          # The generated JD (full_content + sections)
    validation_results: dict    # Quality score, issues, suggestions
    validation_feedback: dict   # Stored feedback for regeneration cycles
    generation_reflection: dict # Generation Agent's self-assessment
    validation_reflection: dict # Validation Agent's self-assessment
    agent_discussion: dict      # Communication Layer output
    reasoning_output: dict      # Reasoning Agent decision + reasoning text
    iteration_count: int        # Current regeneration iteration (0-based)
    stream_messages: list       # Real-time status messages (uses reducer to append)
    execution_log: list         # Full workflow trace with timestamps
```

* **Reducer:** `reduce_messages` — lists **grow** as the run progresses
