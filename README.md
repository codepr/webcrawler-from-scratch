Web crawler from scratch in Go
==============================

Ever wondered how google.com works? What's under the hood that enables any user
to insert a string and obtain related results from the web, given it's inherent
complexity and vastity? How does the search engine indexes all those websites
and correlate their contents with the input string?

We'll try to answer some of these questions by building a simplified version of
the main component that power every search engine at his simplest: a web
crawler. We won't cover all sofistications and ranking algorithms at the core
of the google engine, they're the result of years of research and improvements
and it would require a book on its own just to scratch the surface on those
topics.

This will be a tutorial on how to build something akin to a raw search engine
starting from its inner-most component and extending it by adding features
chapter by chapter.<br>
The repository containing the code is
[https://github.com/codepr/webcrawler](https://github.com/codepr/webcrawler).
During the journey we'll touch many system design concepts:

- Microservices
- Middlewares
- Network unreliability
- Concurrency
- Scalability
    - Consistency patterns
    - Availability patterns

And more in depth on the topic:

- Web crawler main characteristics
    - Politeness
    - Crawling rules
- Reverse indexing services
- Content signatures
