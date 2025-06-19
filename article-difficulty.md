# Estimating Article Difficulty

One of the main features of Zeeguu is that it scouts the internet daily for the most recent articles that might be relevant for a user. Besides the relevance, it also aims to estimate the difficulty of the articles and classify them into CEFR levels. 

The way it does this is that it first computes the Flesch Reading Ease -- a popular measure of difficulty that takes into account sentence complexity and word complexity. The formula for FRI is:

  **FRI Difficulty** = start_constant - sentence_factor × (words/sentences) - word_factor × (syllables/words)

The measure was originally developed for English so the start constant, sentence factor, and word factor are specific to english. However other researchers have adapted the constants for other languages. The Zeeguu team has adapted the formula for the languages that we support but for which we do not have published research. Examples of language specific constants are: 

  **Language-specific constants**:
  
  - **English**: start: 206.835, sentence: 1.015, word: 84.6
  - **German/Danish**: start: 180, sentence: 1, word: 58.5
  - **Spanish/Portuguese**: start: 206.84, sentence: 1.02, word: 60

Finally, based on the formula, we have developed our own [mapping](https://github.com/zeeguu/api/blob/master/zeeguu/core/language/fk_to_cefr.py) from FRI to the CEFR levels. 