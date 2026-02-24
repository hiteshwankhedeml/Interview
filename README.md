# Temperature

* <mark style="color:purple;background-color:purple;">**Controls randomness in token selection.**</mark>
* **Example**
  * Prompt: _“The weather today is”_
  * Possible tokens: `sunny`, `cloudy`, `rainy`
* **Temperature = 0**
  * Always picks the most likely token.
  * Output: `sunny`
* **Temperature = 0.7**
  * Mostly `sunny`, sometimes `cloudy`.
* **Temperature = 1.2**
  * More randomness.
  * `rainy` can appear more often.
* **Summary**
  * Low temperature = deterministic.
  * High temperature = more variation.
