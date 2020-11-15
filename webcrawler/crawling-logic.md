# The crawling logic

We're at the core logic of the web crawler, the algorithm is simple but we have
to define some rules and settings to manage the crawling process and also to
cover up some corner cases as well. For example what to do if we're making too
many calls to the server? What if the response time become more and more higher
at every call? How to decide the user-agent to adopt for each call? These are
only some of the questions that arises during the design of our crawler.

Let's start simple by creating a struct carrying a state in the form of
crawling settings, defining the behavior of the crawler, like the timeout to
apply on each call, the maximum depth to reach on every domain we crawl
(potentially a domain tree could go down several levels)

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
}

// WebCrawler is the main object representing a crawler
type WebCrawler struct {
	// logger is a private logger instance
	logger *log.Logger
	// settings is a pointer to `CrawlerSettings` containing some crawler
	// specifications
	settings *CrawlerSettings
}
```

As we already seen, the problem is easily solved recursively, but go provide us
with instruments to avoid the use of recursion which is generally a prerogative
of functional languages and those that provide tail-recursion optimization (see
Scala, Haskell or Erlang for example).

We want to use an unbuffered channel as our queue for every new URL we want to
crawl, this also allows to spawn a worker go routine for each URL and push all
extracted URLs in each page directly into the channel queue, governed by the
main routine

**crawler.go**
```go
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
				// Skip already visited links or disallowed ones by the robots.txt rules
				if seen[link.String()] || !crawlingRules.Allowed(link) {
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
						time.Sleep(1*time.Second)
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
