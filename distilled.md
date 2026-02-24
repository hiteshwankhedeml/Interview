# 🟢 Distilled

* Knowledge distillation = <mark style="color:purple;background-color:purple;">**training a small model (student) to mimic a large model (teacher)**</mark>
* Teacher model is large, accurate, slow
* Student model is smaller, faster
* <mark style="color:purple;background-color:purple;">**Teacher generates soft outputs (probabilities, hidden states, attention patterns)**</mark>
* <mark style="color:purple;background-color:purple;">**Student is trained to match those outputs**</mark>
* <mark style="color:purple;background-color:purple;">**Loss function measures difference between teacher and student outputs**</mark>
* Student learns generalization from teacher’s “soft knowledge”
* Final student model is lightweight but retains much of teacher performance
* **KL Divergence Loss**
  * Measures difference between teacher and student probability distributions
  * Most common distillation loss
