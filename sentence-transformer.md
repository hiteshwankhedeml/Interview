# 🟢 Sentence Transformer

* <mark style="color:purple;background-color:purple;">**Start with pretrained BERT encoder**</mark>
* <mark style="color:purple;background-color:purple;">**Add pooling layer to convert token embeddings → single sentence embedding**</mark>
* <mark style="color:purple;background-color:purple;">**Train using sentence pairs (similar / not similar)**</mark>
* <mark style="color:purple;background-color:purple;">**Compute cosine similarity between sentence vectors**</mark>
* <mark style="color:purple;background-color:purple;">**Apply contrastive/triplet loss to pull similar sentences closer and push dissimilar ones apart**</mark>

```
      Sentence A                         Sentence B
"Vacation rules"                    "Leave policy"

            │                                  │
            ▼                                  ▼
    ┌────────────────┐                ┌────────────────┐
    │   Shared BERT  │                │   Shared BERT  │
    │   Encoder      │                │   Encoder      │
    └────────────────┘                └────────────────┘
            │                                  │
            ▼                                  ▼
    Token Embeddings                    Token Embeddings
            │                                  │
            ▼                                  ▼
       Mean Pooling                         Mean Pooling
            │                                  │
            ▼                                  ▼
     Sentence Vector A                  Sentence Vector B
            │                                  │
            └──────────────┬───────────────────┘
                           ▼
                 Cosine Similarity
                           │
                           ▼
                       Loss Function
          (Contrastive / Triplet / Ranking Loss)
                           │
                           ▼
                  Backpropagation Updates
```
