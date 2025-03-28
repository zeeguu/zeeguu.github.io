# Topic Classification

Our main goal with Zeeguu is to provide language learners with an easy way to find online resources to improve their language skills. There is an assumption that the users can already read, but need an extra help with the vocabulary they can encounter in the "wild".

To do this, users can select topics during onboarding to customize their homepage based on personal interests, hopefully finding articles they like and are more motivated to engaged with the content.

## The problem

In the past, Zeeguu used a solution of associating specific keywords with topics. Each of these keywords would be checked against the URL and the title of an article, and in case it was found, the associated topic would be added.

This method, while simple, contains some problems, such as:

- **Misclassifications:** Certain words could show up in the title, but not necessarily define the topic of the article. For example, let's consider the following title: "The Government is considering introducing new tech in Schools". If we have the word 'tech' associated with **Technology**, this article would be tagged in the Technology topic, though the article simply discusses introducing new software in classrooms, without any details on what **Politics**.
- **Time consuming and manual definitions:** It requires us to manually add topics for each language, otherwise the filter functionality doesn't work. This process is time consuming, and requires both investigating the sources we are crawling and finding good, representative words for different topics.
- **Consistency across languages:** Some languages could have 10 topics, while another might have 6, and out of those, only 3 overlap. This creates inconsistencies for the user experience, especially when changing language where some of their topics wouldn't be available in the new language.
- **A lot of articles aren't assigned topics**: We also noted that there was a high number of articles that are never given a topic. For example, in Week 45 (2024) we crawled 7528 articles, but only **36.49%** of those were given topics. This means that if a user just browses their homepage without searching, they see only a fraction of what's available in that day.

## Current Solution

We considered various methods - but ultimately we wanted a solution that would be explainable to us, and where we can still keep some control of what is associated with different topics. What we want to solve is that would still allow us to infer topics in new articles, especially for new languages.

Our current method relies on Text Embeddings to recreate a numerical representation of what the text is about. Currently, we are using the [Sentence Transformers](https://sbert.net/), specifically [distiluse-base-multilingual-cased-v2](https://huggingface.co/sentence-transformers/distiluse-base-multilingual-cased-v2), which transforms sentences/paragraphs into a 512D vector. This vector is derived by using a pooling function which by default is the Mean of all output vectors [# Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks](https://arxiv.org/pdf/1908.10084).

In our particular case, we embed each sentence and then take the average of all the sentences in an article to take the final representation. We started by using the smallest model to ensure the lowest footprint in our server and for our purposes the model performs. We are continuing to monitor the performance of the model, and we might upgrade to a better model (i.e. [paraphrase-multilingual-mpnet-base-v2](https://huggingface.co/sentence-transformers/paraphrase-multilingual-mpnet-base-v2)).

With embeddings we can now find similar articles in the database, however, these aren't grouped by topics, so we need a way to map articles to existing topics.

### Topic Assignment

To create the topics we now see in Zeeguu, we wanted to emulate topics that users might be familiar with when navigating in a news/magazine website. We wanted to separate them into topics that are clearly distinguishable, but not too precise to simplify the task for our classification method.

We ended up with the following topics:

- Sports
- Culture & Art
- Technology & Science
- Travel & Tourism
- Health & Society
- Business
- Politics
- Satire

To assign these topics, we resort to the feeds we crawl these articles from, which often contain this information embedded in the URL. For example:

`https://www.dr.dk/nyheder/penge/penge-noerder-deler-ud-her-er-6-tjekpunkter-til-dig-der-overvejer-investere`

In the the above URL, DR states that this article belongs to "penge" (money, in danish), which can be easily associated with the topic "Business". The idea here is to rely on the journalists and sources to be able to infer articles which might not be given a word, or that the word is ambiguous to the type of content included in the article. For example, 'world' news. World doesn't specify the topic of the article's content.

To select the best candidates for Keywords, we extract the URL topic keywords (e.g. "penge", "nyheder", "techologi") and count the number of articles they contain. We then go through them and identify good candidates to map them to the Topics we created (e.g. "football" -> Sports). We give them these mappings and we now will have a subset of articles which have been given topics that we can be fairly confident that they are correct.

These keywords that can be assigned a topic are called **url_keyword_topics** and is now one of the three ways we can assign topics to a specific article.

The two other ways are **hardset_topic** and **inferred_topic**.

#### Hardset topics

We have some sources, for example, The Onion, where all articles are Satire, and are not supposed to be considered as any other topic. For this reason, we can specifically set some feed items to be given a topic no matter what keywords they might contain.

This could also be used to very specific curated sources, where we know for a fact that they only post about a specific topic.

#### Inferred Topics

Now, as stated before, our major issue with the previous version was that a lot of the articles were never assigned topics, and as such a lot of the data was unreachable, if not using the search functionality in Zeeguu.

With the current setup, if an article does not fall into the other two categories, we run topic inference to assign an **inferred_topic**. To do this, we first calculate the articles average embedding as described before, using the _sentence-bert_ model. This representation is then compared to all the other articles that we have indexed through _ElasticSearch_, and take the 9 top neighbors by vector similarity. If there is a majority topic out of these 9 neighbors, rounded down (in this case **4**), then that topic is used as the **inferred_topic** for this new article.

One of the main benefits of this approach, is that we can use all labeled articles from all language to classify a new article, as the embeddings are multilingual. This solves one of the main issues we had when introducing a new language.

## Evaluation

No method is without its issues, and when running inference at this scale we were sure we would misclassifications. However, we argue that our misclassifications aren't as problematic in other domains. The worst that can happen is that a user will see some articles that don't correspond their preferences, but if this doesn't happen too often, overall we consider the fact that they were shown more articles as a net positive.

To allow users to provide feedback and the possibility to better tune the model users are given a chance to "remove" a topic from their feed. This can then be used in the future to better calibrate the model if needed. The interaction can be seen below:
![[Pasted image 20250206124318.png]]**Example of a Inferred topic in the platform**

To evaluate the model, we run a script that classifies topics that we parsed the topic from the URL keywords. When doing this, there could be articles that have more than one topic assigned to them, for example:
(https://atlasmag.dk/kultur/sport/den-sk%C3%B8nneste-grimhed)
![[./img/multiple-topic-article.png]]**Example of URL keyword topics in the platform**

The article covers the celebrations after a major football match in front of one of the oldest theaters in Napoli, making a comparison of the history of grand celebrations of the theater to the present celebrations of the match.

In these cases, if the KNN model gets at least on of the topics correct, then it's considered as a correct label.

We obtain an Weighted F1-Score of 0.74 (P: 0.77, R:0.71), we could reduce the threshold (4 out of 9, e.g. by taking the majority, independently of the number of votes) and obtain higher recall, at the cost of precision, but we are happy with labeling about 2/3 of our articles with topics, whilst almost getting 3/4 articles correctly identified. One major outlier from our results is sports, where P:0.93 and R:0.95, as they tend to have more distinct subject matter and language than the other topics, while being one of the majority of articles we have.

To help visualize our data we resort to tSNE of 5000 random articles, with a perplexity of 15 and here we can see the clusters we form. This can also help us identify topics that might be close together, such as Politics and Health & Society, which makes sense as often articles will detail government measures to improve the health sector, for example.

Culture & Art and Health & Society can also be close, namely Society and Culture, as often articles will cover celebrities and these could either be in one or the other category.

![[tsne-plot-topics.png]]**Visualization of 5000 articles using t-SNE to reduce the 512d to a 2D plot.**

## Future Improvements

In the future, we could re-evaluate the topics we are offering. This initial selection was made by looking at what newspapers usually provide and attempting to strike a balance between distinct enough classes that would provide users a simple, yet powerful way of narrowing down their interests. From the visualization above, we can see that there is room for improvement on the classes themselves, but one of our goals is to improve it based on the user feedback.

Currently, we use the _sentence-bert_ embeddings as it is easily available and it's rather a small model, though in the recent years the field of embeddings and language models has quickly advanced. It's very likely we could pick a more recent and advanced model which would help separate the classes better without adding much overhead during calculation time.

As we collect user data, we might also be able to identify what misclassifications are most common and from there target the weaknesses of the model.

## Conclusion

In this blog post, we covered how topic classifications are done on Zeeguu. We show how embeddings and KNN can be a powerful tool to automatically classify documents in a multi-lingual setting by leveraging the labeled data from existing news sources.

This method also benefits from being more explainable than utilizing fine-tuned models, since we can look into the Neighboring articles that contributed for the classification. Furthermore, it allows us to monitor what time of keywords are used to describe the online articles we crawl and how they are used by different media. For example, we could see a rise of a specific topic such as COVID or AI by it's more frequent use in the URLs.

To conclude, we have one of the latest crawls (Week 45, 2024) where we performed the topic classification in parallel, where we can see that out of the 7528 articles we crawled, 93.42% were assigned a topic given the methodology describe whereas the old method only assigned topics to 36.49% of all articles. This also has the benefit of scaling better across languages, since it doesn't require us to map keywords to topics, and can utilize the embeddings to classify new languages.

![[Pasted image 20250206131551.png]]**Number of Crawled Articles and total topic coverage using the different classification methods. On the right, the keyword based method and on the left, the KNN method described.**

## To-do

- Add visualization of how the indexing works.
