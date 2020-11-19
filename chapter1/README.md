How does the crawler service work
=================================

In its simplest form a web crawler is a program that explores the web starting
from an URL and tries to build a sitemap following every link it finds. Link by
link it stores informations like site content and URL domain to a storage
layer, making it possible to create, for example, reverse indexes and content
signatures in order to provide efficient searching capabilities through APIs.
The main components at the core of a search engine are:

* A **crawling service**, walks across a domain extracting links it finds and
  downloading every pages' body
* A **reverse index service**, generates a mapping of words to pages containing
  the search term
* A **document service**, generates a static title and snippet for each fetched
  link
* A **REST interface** to query the application and obtain results

On next chapters we're going to explore each one of these services more in
depth and at the end of every sub-section we're going to add a `main` entry
point to try out the application. We'll try to design every component to be
self-contained, independent and scalable.

Starting from the first and most important component, the very beating heart of
the application, the crawler service is in its essence a recursive algorithm
applied to a queue of URLs:

1. get the next URL from the fetching queue
2. fetch the URL page content
3. extract all links from the page
4. enqueue all found links into the fetching queue
5. repeat 1

Refinements could be implemented, like de-duplication of links (we don't want
to crawl multiple times the same link) based on some heuristics like URL domain
and content signatures, also politeness and robots.txt directives have to be
considered, but the business logic at the core is summarized into those 5
steps.
