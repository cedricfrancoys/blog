# One API to rule them All

Writing an API from scratch to fit the needs of an Application is often a challenging and thrilling experience: development teams may have defined a logic of their own that seems the best suited for the product, or maybe they even tried an exiting GraphQL implementation.

![](//i.imgur.com/f3MUsmzl.png)

That being said, when the design phase of an API has been completed, even if being able to merge data from different sources has been planned from the very beginning,  it is not uncommon - for many good reasons (change of provider, documentation not provided before contracting a subscription, etc.) - to be confronted with data formats which are not compatible with the structures defined internally.


Besides - sadly -, not all data-providers support graphQL (or even JSON !).

In such cases, there is no choice but to adapt the data by processing it with a multi-pass strategy (convert XML to JSON, then extract targeted data from JSON) before being able to use it.




Nevertheless, for all we know, it is very likely that this kind of situation will become more and more common. Many companies are indeed developing services around Big Data, AI analysis and real-time data processing.



To be able to handle this kind of situation in the best possible way - which makes even more sense as such services are spreading - the best approach would be: 

* to consider the internal Interface(s) as one API among others (i.e. not the "main" one);
* to create a meta-API service dedicated to the aggregation of all data sources used;
* to query that service using a unified format.



There are many possibilities to achieve this but it is undeniably advantageous to use a standard that will support backward compatibility. Besides, in this context it really makes sense to plan a long term portability.

Therefore for this kind of service, implementing a ReSTful API is probably the wisest choice.





### Further readings

* Anoter article I wrote about [most popular ReST JSON specifications](https://cedricfrancoys.be/article/API/ReSTful-JSON)
* The Wikipedia entry on [Representational state transfer](https://fr.wikipedia.org/wiki/Representational_state_transfer)
* [Roy Thomas Fielding's original dissertation about Representational State Transfer](https://www.ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation.pdf)
