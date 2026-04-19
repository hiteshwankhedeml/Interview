# 🔴 Search Method

<mark style="color:$danger;background-color:purple;">**We use filtered vector search in Elasticsearch:**</mark>

* <mark style="color:$danger;background-color:purple;">**First, we extract structured parameters like experience, location, and certification from the user query**</mark>
* <mark style="color:$danger;background-color:purple;">**Then we convert the full query into an embedding**</mark>
* <mark style="color:$danger;background-color:purple;">**Finally, we perform a kNN search on resume embeddings with filters applied”**</mark>

