# top\_p

* <mark style="color:purple;background-color:purple;">**Limits**</mark><mark style="color:purple;background-color:purple;">**&#x20;**</mark>_<mark style="color:purple;background-color:purple;">**which tokens**</mark>_<mark style="color:purple;background-color:purple;">**&#x20;**</mark><mark style="color:purple;background-color:purple;">**the model can choose from.**</mark>
* **Example**
  * Prompt: _“The weather today is”_
  * Token probabilities:
    * `sunny` → 0.60
    * `cloudy` → 0.25
    * `rainy` → 0.10
    * `apocalyptic` → 0.05
* **top\_p = 0.6**
  * Take smallest set with cumulative probability ≥ 0.6.
  * Allowed tokens: `sunny`
  * Output: always `sunny`
* **top\_p = 0.85**
  * Cumulative: `sunny + cloudy = 0.85`
  * Allowed tokens: `sunny`, `cloudy`
* **top\_p = 1.0**
  * All tokens allowed.
  * Behaves like no restriction.
* **Summary**
  * Low `top_p` = safer, limited choices.
  * High `top_p` = more diversity.
  * `top_p` **filters tokens**, it does not reshape probabilities.
