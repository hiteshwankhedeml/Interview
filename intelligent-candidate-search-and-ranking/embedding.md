# Embedding

For each resume JSON:

1. **Generate embedding** — Use Sentence Transformers (`all-MiniLM-L6-v2`) to embed `resume_text` → 384-dim vector
2. **Build document** — Copy all fields from JSON + add `resume_embedding`
3. **Use `candidate_id` as document ID** — For upserts and deduplication

```python
# Pseudocode
embedding = model.encode(resume["resume_text"]).tolist()
doc = {
    **resume,
    "resume_embedding": embedding
}
```
