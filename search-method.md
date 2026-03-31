# 🔴 Search Method

We use filtered vector search in Elasticsearch:

* First, we extract structured parameters like experience, location, and certification from the user query
* Then we convert the full query into an embedding
* Finally, we perform a kNN search on resume embeddings with filters applied”
