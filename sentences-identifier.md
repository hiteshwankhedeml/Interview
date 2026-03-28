# 🟢 Sentences Identifier

* <mark style="color:purple;background-color:purple;">**Numbered clauses –**</mark> Looks for patterns like 9.1, 15.1.1 at the start of lines and treats each as the start of a new clause.
* <mark style="color:purple;background-color:purple;">**Section headers –**</mark> Detects section titles (e.g. LIMITATION OF LIABILITY) and uses them as context for the following clauses.
* <mark style="color:purple;background-color:purple;">**Line grouping –**</mark> Lines that are not a new number or section are appended to the current clause until the next number or section.
* <mark style="color:purple;background-color:purple;">**Fallback –**</mark> If fewer than 3 clauses are found, uses spaCy to split the text into sentences and treats each sentence as a clause
* The main logic splits on numbered clauses (e.g. 9.1, 15.1.1). <mark style="color:purple;background-color:purple;">**Some documents don’t use the numbered clause format. spaCy is used as a fallback**</mark> to still get clauses when the numbered pattern finds fewer than 3 clauses.
*

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         RAW DOCUMENT TEXT                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                        ┌───────────────────────┐
                        │  Split by newlines    │
                        │  text.split("\n")     │
                        └───────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FOR EACH LINE                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
            ▼                       ▼                       ▼
┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐
│ Section header?   │   │ Numbered clause?   │   │ Continuation      │
│ (e.g. "9. LIMIT   │   │ (e.g. "9.1 " or   │   │ of current block  │
│  OF LIABILITY")   │   │  "15.1.1 ")       │   │                   │
└─────────┬─────────┘   └─────────┬─────────┘   └─────────┬─────────┘
          │                       │                       │
          │ YES                   │ YES                   │
          ▼                       ▼                       ▼
┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐
│ Flush block       │   │ Flush block       │   │ Append line to    │
│ Update section    │   │ Start new block   │   │ current_block     │
│ Skip this line    │   │ with clause_id    │   │                   │
└───────────────────┘   └───────────────────┘   └───────────────────┘
          │                       │                       │
          └───────────────────────┴───────────────────────┘
                                    │
                                    ▼
                        ┌───────────────────────┐
                        │  Flush final block    │
                        │  → List of Clauses    │
                        └───────────────────────┘
                                    │
                                    ▼
                        ┌───────────────────────┐
                        │  len(clauses) < 3?    │
                        └───────────┬───────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │ YES                           │ NO
                    ▼                               ▼
        ┌───────────────────────┐       ┌───────────────────────┐
        │ SPAcy FALLBACK        │       │ Return clauses        │
        │ en_core_web_sm        │       │ (from numbered split)  │
        │ doc.sents             │       └───────────────────────┘
        │ Skip if len(sent)≤20  │
        └───────────┬───────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │ Return clauses        │
        │ (from sentence split) │
        └───────────────────────┘
```

