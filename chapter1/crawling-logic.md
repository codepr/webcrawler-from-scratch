# The crawling logic

We're at the core logic of the web crawler, the algorithm is simple but we have
to define some rules and settings to manage the crawling process and also to
cover up some corner cases as well. For example what to do if we're making too
many calls to the server? What if the response time gets higher
at every call? How to decide the user-agent to adopt for each call? These are
only some of the questions that arises during the design of our crawler.

## Crawling settings

Let's start simple by creating a struct carrying a state in the form of
crawling settings, defining the behavior of the crawler.

What we want to be able to configure is:

* `fetchingTimeout`, the number of (m)s to wait before declaring the link
  unreachable if it's hanging
* `crawlingTimeout`, the number of (m)s to wait after the last link we found
  in the last crawled page, after which we declare the crawling done
* `politenessDelay`, the number of (m)s to wait after an HTTP request before
  making another under the same domain
* `maxDepth`, the number of level to crawl through a domain tree, some sites
  can be several levels deep, it comes handy set a limit in this terms
* `maxConcurrency`, the number of concurrent worker goroutines that may run
  during the crawling process of a domain, this is utmost important
* `userAgent`, the user-agent that we declare on every HTTP request header,
  this is also an important setting, being polite when crawling a domain allow
  the server to know who's visiting every link and also define some
  "rules of the house" to be applied in order to avoid being banned

### Limiting the concurrency

Setting an upper limit on the number of concurrent goroutines is vital in this
applications and generally that limit isn't even that high, some `robots.txt`
(those "rules of the house" defined by the domain) often set a politeness delay
of over 1 min, or you start receive lot of `429: Too many requests` responses
or `503: Service unavailable` with a `Retry-After` header set. This makes
harder than it seems to write a good crawling algorithm, in this case we start
simple by setting a fixed politeness delay and a concurrency limit, going
incremental we'll try to implement some sort of heuristic to take into account
the response time of each call to adjust the delay and also the `robots.txt`
rules if present.

The other concerns about letting unlimited concurrency are the most known
regarding the resources of the host machine:

* goroutines are cheap and can be spawned in millions, each goroutine has a
  memory cost around 2-5 Kbs, clearly those millions would put a toll in terms
  of memory usage
* the number of opened state file descriptors grows really fast, under Linux
  without increasing it with `ulimit n` the limit is reached pretty fast as
  every TCP connection relies on a socket; this is especially true if the
  crawler needs also to maintain a pool of connections to a DB or store data
  on disk

**crawler.go**

```go
// Package crawler containing the crawling logics and utilities to scrape
// remote resources on the web
package crawler

import (
	"context"
	"encoding/json"
	"log"
	"net/url"
	"os"
	"os/signal"
	"sync"
	"sync/atomic"
	"syscall"
	"time"

	"github.com/codepr/webcrawler/fetcher"
)

const (
	// Default fetcher timeout before giving up an URL
	defaultFetchTimeout time.Duration = 10 * time.Second
	// Default crawling timeout, time to wait to stop the crawl after no links are
	// found
	defaultCrawlingTimeout time.Duration = 30 * time.Second
    // Default politeness delay, fixed delay to calculate a randomized wait time
	// for subsequent HTTP calls to a domain
	defaultPolitenessDelay time.Duration = 500 * time.Millisecond
	// Default depth to crawl for each domain
	defaultDepth int = 16
	// Default number of concurrent goroutines to crawl
	defaultConcurrency int = 8
	// Default user agent to use
	defaultUserAgent string = "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"
)

// ParsedResult contains the URL crawled and an array of links found, json
// serializable to be sent on message queues
type ParsedResult struct {
	URL   string   `json:"url"`
	Links []string `json:"links"`
}

// CrawlerSettings represents general settings for the crawler and his
// dependencies
type CrawlerSettings struct {
	// FetchingTimeout is the time to wait before closing a connection that does not
	// respond
	FetchingTimeout time.Duration
	// CrawlingTimeout is the number of second to wait before exiting the crawling
	// in case of no links found
	CrawlingTimeout time.Duration
	// Concurrency is the number of concurrent goroutine to run while fetching
	// a page. 0 means unbounded
	Concurrency int
	// Parser is a `fetcher.Parser` instance object used to parse fetched pages
	Parser fetcher.Parser
	// MaxDepth represents a limit on the number of pages recursively fetched.
	// 0 means unlimited
	MaxDepth int
	// UserAgent is the user-agent header set in each GET request, most of the
	// times it also defines which robots.txt rules to follow while crawling a
	// domain, depending on the directives specified by the site admin
	UserAgent string
    // PolitenessFixedDelay represents the delay to wait between subsequent
	// calls to the same domain, it'll taken into consideration against a
	// robots.txt if present and against the last response time, taking always
	// the major between these last two. Robots.txt has the precedence.
	PolitenessFixedDelay time.Duration
}

// WebCrawler is the main object representing a crawler
type WebCrawler struct {
	// logger is a private logger instance
	logger *log.Logger
	// settings is a pointer to `CrawlerSettings` containing some crawler
	// specifications
	settings *CrawlerSettings
}

// New create a new Crawler instance, accepting a maximum level of depth during
// crawling all the anchor links inside each page, a concurrency limiter that
// defines how many goroutine to run in parallel while fetching links and a
// timeout for each HTTP call.
func New(userAgent string) *WebCrawler {
	// Default crawler settings
	settings := &CrawlerSettings{
		FetchingTimeout:      defaultFetchTimeout,
		Parser:               fetcher.NewGoqueryParser(),
		UserAgent:            userAgent,
		CrawlingTimeout:      defaultCrawlingTimeout,
		PolitenessFixedDelay: defaultPolitenessDelay,
		Concurrency:          defaultConcurrency,
	}
	crawler := &WebCrawler{
		logger:   log.New(os.Stderr, "crawler: ", log.LstdFlags),
		settings: settings,
	}
	return crawler
}

// NewFromSettings create a new webCrawler with the settings passed in
func NewFromSettings(settings *CrawlerSettings) *WebCrawler {
	return &WebCrawler{
		logger:   log.New(os.Stderr, "crawler: ", log.LstdFlags),
		settings: settings,
	}
}
```

As we can see, the number of settings is high and a little uncomfortable to
manage with constructor functions, this is a good case to adopt an opt pattern,
a common build pattern in Go, let's modify the `New` function:

**crawler.go**
```go
// CrawlerOpt is a type definition for option pattern while creating a new
// crawler
type CrawlerOpt func(*CrawlerSettings)

// New create a new Crawler instance, accepting a maximum level of depth during
// crawling all the anchor links inside each page, a concurrency limiter that
// defines how many goroutine to run in parallel while fetching links and a
// timeout for each HTTP call.
func New(userAgent string, opts ...CrawlerOpt) *WebCrawler {
	// Default crawler settings
	settings := &CrawlerSettings{
		FetchingTimeout:      defaultFetchTimeout,
		Parser:               fetcher.NewGoqueryParser(),
		UserAgent:            userAgent,
		CrawlingTimeout:      defaultCrawlingTimeout,
		PolitenessFixedDelay: defaultPolitenessDelay,
		Concurrency:          defaultConcurrency,
	}
	// Mix in all optionals
	for _, opt := range opts {
		opt(settings)
	}
	crawler := &WebCrawler{
		logger:   log.New(os.Stderr, "crawler: ", log.LstdFlags),
		settings: settings,
	}
	return crawler
}
```

Now the constructor accepts optional parameters in the form of a factory
function, this makes possible to customize the creation of the `WebCrawler`
object:

```go
// withConcurrency is a simple constructor option to pass into the
// crawler.New function call to set the concurrency level
func withConcurrency(concurrency int) crawler.CrawlerOpt {
	return func(s *crawler.CrawlerSettings) {
		s.Concurrency = concurrency
	}
}
c := crawler.New("user-agent", withConcurrency(4))
```

## Crawling a domain

The core of the crawler component, we want to export just one simple function
that allows us to start the crawling process on one or more domains,
practically we can see the application flow as a coordinator-helpers hierarchy
consisting of two simple functions:

* `Crawl` the only exported function, accepts a variadic number of URL strings
  and spawn a `crawlPage` goroutine for each of them
* `crawlPage` private function, contains the core logic, for each URL in the
  queue extracts every link found and put them into the same queue, starting
  from the upper-most link passed in by the `Crawl` function

As we already seen, the problem is easily solved recursively, but Go provides
us tools to avoid the use of recursion which is generally a prerogative of
functional languages and those that provide tail-recursion optimization (see
Scala, Haskell or Erlang for example).

We want to use an unbuffered channel as our queue for every new URL we want to
crawl, this also allows to spawn a worker goroutine for each URL and push all
extracted URLs in each page directly into the channel queue, governed by the
main routine

**crawler.go**
```go
// Crawl a single page by fetching the starting URL, extracting all anchors
// and exploring each one of them applying the same steps. Every image link
// found is forwarded into a dedicated channel, as well as errors.
//
// A waitgroup is used to synchronize it's execution, enabling the caller to
// wait for completion.
func (c *WebCrawler) crawlPage(rootURL *url.URL, wg *sync.WaitGroup, ctx context.Context) {
	// First we wanna make sure we decrease the waitgroup counter at the end of
	// the crawling
	defer wg.Done()
	fetchClient := fetcher.New(c.settings.UserAgent,
		c.settings.Parser, c.settings.FetchingTimeout)
	var (
		// semaphore is just a value-less channel used to limit the number of
		// concurrent goroutine workers fetching links
		semaphore chan struct{}
		// New found links channel
		linksCh chan []*url.URL
		stop    bool
		depth   int
		// A map is used to track all visited links, in order to avoid multiple
		// fetches on the previous visited links
		seen    map[string]bool = make(map[string]bool)
		fetchWg sync.WaitGroup  = sync.WaitGroup{}
		// An atomic counter to make sure that we've already crawled all remaining
		// links if a timeout occur
		linkCounter int32 = 1
	)
	// Set the concurrency level by using a buffered channel as semaphore
	if c.settings.Concurrency > 0 {
		semaphore = make(chan struct{}, c.settings.Concurrency)
		linksCh = make(chan []*url.URL, c.settings.Concurrency)
	} else {
		// we want to disallow the unlimited concurrency, to avoid being banned from
		// the ccurrent crawled domain and also to avoid running OOM or running out
		// of unix file descriptors, as each HTTP call is built upon a  socket
		// connection, which is in-fact an opened descriptor.
		semaphore = make(chan struct{}, 1)
		linksCh = make(chan []*url.URL, 1)
	}
	// Just a kickstart for the first URL to scrape
	linksCh <- []*url.URL{rootURL}
	// Every cycle represents a single page crawling, when new anchors are
	// found, the counter is increased, making the loop continue till the
	// end of links
	for !stop {
		select {
		case links := <-linksCh:
			for _, link := range links {
				// Skip already visited links
				if seen[link.String()] {
					atomic.AddInt32(&linkCounter, -1)
					continue
				}
				seen[link.String()] = true
				// Spawn a goroutine to fetch the link, throttling by
				// concurrency argument on the semaphore will take care of the
				// concurrent number of goroutine.
				fetchWg.Add(1)
				go func(link *url.URL, stopSentinel bool, w *sync.WaitGroup) {
					defer w.Done()
					defer atomic.AddInt32(&linkCounter, -1)
					// 0 concurrency level means we serialize calls as
					// goroutines are cheap but not that cheap (around 2-5 kb
					// each, 1 million links = ~4/5 GB ram), by allowing for
					// unlimited number of workers, potentially we could run
					// OOM (or banned from the website) really fast
					semaphore <- struct{}{}
					defer func() {
						time.Sleep(c.settings.PolitenessFixedDelay)
						<-semaphore
					}()
					// We fetch the current link here and parse HTML for children links
					responseTime, foundLinks, err := fetchClient.FetchLinks(link.String())
					if err != nil {
						c.logger.Println(err)
						return
					}
					// No errors occured, we want to enqueue all scraped links
					// to the link queue
					if stopSentinel || foundLinks == nil || len(foundLinks) == 0 {
						return
					}
					atomic.AddInt32(&linkCounter, int32(len(foundLinks)))
					// Enqueue found links for the next cycle
					linksCh <- foundLinks
				}(link, stop, &fetchWg)
				// We want to check if a level limit is set and in case, check if
				// it's reached as every explored link count as a level
				if c.settings.MaxDepth == 0 || !stop {
					depth++
					stop = c.settings.MaxDepth > 0 && depth >= c.settings.MaxDepth
				}
			}
		case <-time.After(c.settings.CrawlingTimeout):
			// c.settings.CrawlingTimeout seconds without any new link found, check
			// that the remaining links have been processed and stop the iteration
			if atomic.LoadInt32(&linkCounter) <= 0 {
				stop = true
			}
		case <-ctx.Done():
			return
		}
	}
	fetchWg.Wait()
}

// Crawl will walk through a list of URLs spawning a goroutine for each one of
// them
func (c *WebCrawler) Crawl(URLs ...string) {
	wg := sync.WaitGroup{}
	ctx, cancel := context.WithCancel(context.Background())
	// Sanity check for URLs passed, check that they're in the form
	// scheme://host:port/path, adding missing fields
	for _, href := range URLs {
		url, err := url.Parse(href)
		if err != nil {
			log.Fatal(err)
		}
		if url.Scheme == "" {
			url.Scheme = "https"
		}
		// Spawn a goroutine for each URLs to crawl, a waitgroup is used to wait
		// for completion
		wg.Add(1)
		go c.crawlPage(url, &wg, ctx)
	}
	// Graceful shutdown of workers
	signalCh := make(chan os.Signal, 1)
	signal.Notify(signalCh, os.Interrupt, syscall.SIGTERM)
	go func() {
		<-signalCh
		cancel()
		os.Exit(1)
	}()
	wg.Wait()
	c.logger.Println("Crawling done")
}
```

Although `crawlPage` is admittedly a little complex and there's room for
improvements, the flow is indeed simple and came out as a variation of the
previously thought steps:

1. enqueue start URL in the channel as a list of 1 element
2. dequeue the link-list from the queue
3. for each link in the list
    * check the ephemeral state of the crawling to see if we already have
      visited the URL
    * fetch HTML contents and extracts all links from the page
    * enqueue every found link into the channel queue
4. goto 2

The `select` makes it a little harder to follow as it introduces asynchronicity
in the flow, but the low number of branches, just two, makes it simple to
understand: In the second branch it just starts a timer everytime "nothing
happens" in the main channel queue, in other word if no links are found for a
determined amount of time (`crawlingTimeout`) the loop gets interrupted and the
crawling for that domain ends.

As of now however, it's difficult to unit-test, we have to find out a way to
make the business logic less tied to the application and above all make it
possible to forward crawling results to outside, so that external clients may
use them without being tightly coupled with the crawler.
That way it'd be possible to perform an opaque-box testing, without worrying
of what's happening inside the crawling loop.

We've just added two additional `.go` files for the `crawler` package, one for
unit tests and one for source, this the current project structure so far:

```sh
tree
.
├── crawler.go
├── crawler_test.go
├── fetcher
│   ├── fetcher.go
│   ├── fetcher_test.go
│   ├── parser.go
│   └── parser_test.go
├── go.mod
└── go.sum
```
