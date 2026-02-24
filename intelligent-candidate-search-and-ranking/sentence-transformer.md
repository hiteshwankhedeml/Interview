# Sentence Transformer

* Start with pretrained BERT encoder
* Add pooling layer to convert token embeddings → single sentence embedding
* Train using sentence pairs (similar / not similar)
* Compute cosine similarity between sentence vectors
* Apply contrastive/triplet loss to pull similar sentences closer and push dissimilar ones apart

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
