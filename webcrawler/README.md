What is a web crawler
=====================

In its simplest form a web crawler is a program that explore the web starting
from an URL and tries to build a sitemap following every link it finds, link by
link it stores informations like site content and URL domain to a storage
layer, making it possible to create, for example, reverse indexes and content
signatures in order to provide efficient searching capabilities through APIs.
In its essence, it's a recursive algorithm applied to a queue of URLs:

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
