# Crawling rules

One of the most important traits that a web crawler must fulfill is the
politeness towards every domain it will crawl. Politeness is achieved by
applying some rules regarding:

* Number of requests per unit of time to the website
* Respecting the will of the domain about allowed sub-domains and links to
  follow
* Respect the website behavior, expressed by HTTP responses to each call

Conceptually it's a set of well-manner rules, would you ever enter in a
stranger's house and start opening all his rooms, fridges and wardrobes or
hell, use their bathroom without asking? The least that can happen is that
you'll end up banned with a restraining injunction towards that place. And it's
exactly what happens with web crawlers that do not respect well-manner rules.

Generally most domains put a file named `robots.txt` in the root of their
website which contains these rules, meant exactly for web crawlers and bots,
sometimes general rules, sometimes targeted rules based on the user-agent of
the crawler, sometimes both:

```
User-agent: badbot
Disallow: /             # Disallow all

User-agent: *
Allow: */bar/*
Disallow: */baz/*
Crawl-delay: 2          # 2 seconds of delay between each call
```

In this example we can see the webadmin specified a targeted rule for "badbot"
which disallow the crawling for the entire domain, and a general rule for
everyone which apply a crawling delay of two seconds between HTTP calls.

Our crawler already supports a simple directive regarding the number of
requests per unit of time, in the form of a `politenessDelay`, but we can do
better, we're going to add a new object specifically responsible of the
managing of these rules, we expect it to:

* Be able to parse `robots.txt` files
* Be able to calculate a good delay between calls, taking into consideration
  robots rules, the politeness delay and the response time of each call
* Tell us if a domain is allowed to be crawled

Parsing `robots.txt` is a simple but a tedious job,
[github.com/temoto/robotstxt](github.com/temoto/robotstxt) offers nice APIs to
manage these rules efficiently, thus our struct will carry a `robotstxt.Group`
pointer, the other member will be the `politenessDelay` we previously used as
delay between calls on **crawler.go**.<br>
The test suite should be straight-forward to write, let's start simple, just a
simple server mock with a fake `robots.txt` path to be parsed.

**crawlingrules\_test.go**

```go
// Package crawler containing the crawling logics and utilities to scrape
// remote resources on the web
package crawler

import (
	"net/http"
	"net/http/httptest"
	"net/url"
	"testing"
	"time"
)

func serverMock() *httptest.Server {
	handler := http.NewServeMux()
	handler.HandleFunc("/robots.txt", func(w http.ResponseWriter, r *http.Request) {
		_, _ = w.Write([]byte(
			`User-agent: *
	Disallow: */baz/*
	Crawl-delay: 2`,
		))
	})
	server := httptest.NewServer(handler)
	return server
}

func serverWithoutCrawlingRules() *httptest.Server {
	handler := http.NewServeMux()
	handler.HandleFunc("/foo", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	})
	server := httptest.NewServer(handler)
	return server
}

func TestCrawlingRules(t *testing.T) {
	server := serverMock()
	defer server.Close()
	serverURL, _ := url.Parse(server.URL)
	r := NewCrawlingRules(serverURL, 100*time.Millisecond)
	testLink, _ := url.Parse(server.URL + "/foo/baz/bar")
	if !r.Allowed(testLink) {
		t.Errorf("CrawlingRules#IsAllowed failed: expected true got false")
	}
	r.GetRobotsTxtGroup("test-agent", serverURL)
	if r.Allowed(testLink) {
		t.Errorf("CrawlingRules#IsAllowed failed: expected false got true")
	}
	if r.CrawlDelay() != 2*time.Second {
		t.Errorf("CrawlingRules#CrawlDelay failed: expected 2 got %d", r.CrawlDelay())
	}
}

func TestCrawlingRulesNotFound(t *testing.T) {
	server := serverWithoutCrawlingRules()
	defer server.Close()
	serverURL, _ := url.Parse(server.URL)
	r := NewCrawlingRules(serverURL, 100*time.Millisecond)
	if r.GetRobotsTxtGroup("test-agent", serverURL) {
		t.Errorf("CrawlingRules#GetRobotsTxtGroup failed")
	}
}
```

Our `CrawlingRules`, for now, will handle just a single domain, so it'll be
logically instantiated each time we want to crawl a domain, in the `crawlPage`
method. This implies that the object will be shared between multiple concurrent
workers, we need to make it thread-safe with a mutex, Go offers two mutex types:

* `sync.Mutex` the classical mutex lock, mutual exclusion of read and write
  operations, once hold the lock, no one can actually access the critical part
  of the guarded code
* `sync.RWMutex` this is a little more relaxed, offers the possibility to lock
  either for read or for write operations, based on the rationale that only
  writes bring changes, it makes possible to have unlimited read-lock, but just
  only one write-lock, and no one can access the critical part if a write lock
  is hold, till the release

We choose the `sync.RWMutex`, as we mentioned earlier, we want also to take
into account the server reactions towards our requests, beside the status code,
the first metric that we want to mix-in in the crawling rules is the response
time, and that's exactly where we want to guard against concurrent access,
after each call we'd like to update the last call delay, but also let other
workers access it if needed to delay their next call, so we're going to need
a full lock on update (write) and just a read lock on read.

The `CrawlingRules` object as we designed it will expose four methods:

* `GetRobotsTxtGroup(userAgent string, domain *url.URL) bool`, tries to
  retrieve the `robots.txt` file and parse it from the domain root, return
  `true` if a valid file is found
* `Allowed(url *url.URL) bool`, return the validity of an URL according to the
  rules, `true` means that it can be crawled
* `UpdateLastDelay(lastResponseTime time.Duration)` just update the last delay
  according with the latest response time
* `CrawlDelay() time.Duration` return the crawling delay to be applied for the
  next HTTP call

**crawlingrules.go**
```go
// Package crawler containing the crawling logics and utilities to scrape
// remote resources on the web
package crawler

import (
	"math"
	"math/rand"
	"net/http"
	"net/url"
	"sync"
	"time"

	"github.com/codepr/webcrawler/fetcher"

	"github.com/temoto/robotstxt"
)

// Default /robots.txt path on server
const robotsTxtPath string = "/robots.txt"

// CrawlingRules contains the rules to be obeyed during the crawling of a single
// domain, including allowances and delays to respect.
//
// There are a total of 3 different delays for each domain, the robots.txt has
// always the precedence over the fixedDelay and the lastDelay.
// If no robots.txt is found during the crawl, a random delay will be calculated
// based on the response time of the last request, if a fixedDelay is set, the
// major between a random value between 1.5 * fixedDelay and 0.5 * fixedDelay
// and the lastDelay will be chosen.
type CrawlingRules struct {
	// temoto/robotstxt backend is used to fetch the robotsGroup from the
	// robots.txt file
	robotsGroup *robotstxt.Group
	// A fixed delay to respect on each request if no valid robots.txt is found
	fixedDelay time.Duration
	// The delay of the last request, useful to calculate a new delay for the
	// next request
	lastDelay time.Duration
	// A RWmutex is needed to make the delya calculation threadsafe as this
	// struct will be shared among multiple goroutines
	rwMutex sync.RWMutex
}

// NewCrawlingRules creates a new CrawlingRules struct
func NewCrawlingRules(fixedDelay time.Duration) *CrawlingRules {
	return &CrawlingRules{fixedDelay: fixedDelay}
}

// Allowed tests for eligibility of an URL to be crawled, based on the rules
// of the robots.txt file on the server. If no valid robots.txt is found all
// URLs in the domain are assumed to be allowed, returning true.
func (r *CrawlingRules) Allowed(url *url.URL) bool {
	if r.robotsGroup != nil {
		return r.robotsGroup.Test(url.RequestURI())
	}
	return True
}

// GetRobotsTxtGroup tryes to fetch the robots.txt from the domain and parse
// it. Returns a boolean based on the success of the process.
func (r *CrawlingRules) GetRobotsTxtGroup(userAgent string, domain *url.URL) bool {
	f := fetcher.New(userAgent, nil, 10*time.Second)
	u, _ := url.Parse(robotsTxtPath)
	targetURL := domain.ResolveReference(u)
	// Try to fetch the robots.txt file
	_, res, err := f.Fetch(targetURL.String())
	if err != nil || res.StatusCode == http.StatusNotFound {
		return false
	}
	body, err := robotstxt.FromResponse(res)
	// If robots data cannot be parsed, will return nil, which will allow access by default.
	// Reasonable, since by default no robots.txt means full access, so invalid
	// robots.txt is similar behavior.
	if err != nil {
		return false
	}
	r.robotsGroup = body.FindGroup(userAgent)
	return r.robotsGroup != nil
}
```

Here is where we need the `Fetcher` once again, in `GetRobotsTxtGroup` it's
used to retrieve the `robots.txt`, generally located in the root at
domain:port/robots.txt.

The crawling delay can be designed in a moltitude of ways, we opt for a way
that should favor the server reaction time over the `politenessDelay` that's
configured at the start of the application, basically following this formula:

```python
delay = max(random.randrange(politeness_delay*0.5, politeness_delay*1.5), robotstxt_delay, last_response**2)
```

The maximum value between

* `robots.txt` delay
* power of 2 of last response delay in seconds
* random value x for `politenessDelay` \* 0.5 <= x <= `politenessDelay` \* 1.5

**crawlingrules.go**

```go
// CrawlDelay return the delay to be respected for the next request on a same
// domain. It chooses from 3 different possible delays, the most important one
// is the one defined by the robots.txt of the domain, then it proceeds
// generating a random delay based on the last request response time and a
// fixed delay set by configuration of the crawler.
//
// It follows these steps:
//
// - robots.txt delay
// - delay = random 0.5*fixedDelay and 1.5*fixedDelay
// - max(lastResponseTime^2, delay, robots.txt delay)
func (r *CrawlingRules) CrawlDelay() time.Duration {
	r.rwMutex.RLock()
	defer r.rwMutex.RUnlock()
	var delay time.Duration
	if r.robotsGroup != nil {
		delay = r.robotsGroup.CrawlDelay
	}
	// We calculate a random value: 0.5*fixedDelay < value < 1.5*fixedDelay
	randomDelay := randDelay(int64(r.fixedDelay.Milliseconds())) * time.Millisecond
	baseDelay := time.Duration(
		math.Max(float64(randomDelay.Milliseconds()), float64(delay.Milliseconds())),
	) * time.Millisecond
	// We return the max between the random value calculated and the lastDelay
	return time.Duration(
		math.Max(float64(r.lastDelay.Milliseconds()), float64(baseDelay.Milliseconds())),
	) * time.Millisecond
}

// SetDelay just pow(2) the lastTime response in seconds and set it as the
// lastDelay value
func (r *CrawlingRules) UpdateLastDelay(lastResponseTime time.Duration) {
	r.rwMutex.Lock()
	r.lastDelay = time.Duration(
		math.Pow(float64(lastResponseTime.Seconds()), 2.0),
	) * time.Second
	r.rwMutex.Unlock()
}

// Return a random value between 1.5*value and 0.5*value
func randDelay(value int64) time.Duration {
	if value == 0 {
		return 0
	}
	max, min := 1.5*float64(value), 0.5*float64(value)
	return time.Duration(rand.Int63n(int64(max-min)) + int64(max))
}
```

We should update **crawler.go** file with the crawling rules to be applied
during the crawling process of a domain, in the `crawlPage` private function

```diff
func (c *WebCrawler) crawlPage(rootURL *url.URL, wg *sync.WaitGroup, ctx context.Context) {
	...
    // Just a kickstart for the first URL to scrape
	linksCh <- []*url.URL{rootURL}
+   // We try to fetch a robots.txt rule to follow, being polite to the
+	// domain
+	crawlingRules := NewCrawlingRules(rootURL, c.settings.PolitenessFixedDelay)
+	if crawlingRules.GetRobotsTxtGroup(c.settings.UserAgent, rootURL) {
+		log.Printf("Found a valid %s/robots.txt", rootURL.Host)
+	} else {
+		c.logger.Printf("No valid %s/robots.txt found", rootURL.Host)
+	}

	// Every cycle represents a single page crawling, when new anchors are
	// found, the counter is increased, making the loop continue till the
	// end of links
	for !stop {
		select {
		case links := <-linksCh:
			for _, link := range links {
				// Skip already visited links or disallowed ones by the robots.txt rules
-				if seen[link.String()] {
+				if seen[link.String()] || !crawlingRules.Allowed(link) {
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
						time.Sleep(crawlingRules.CrawlDelay())
						<-semaphore
					}()
					// We fetch the current link here and parse HTML for children links
					responseTime, foundLinks, err := fetchClient.FetchLinks(link.String())
+					crawlingRules.UpdateLastDelay(responseTime)
					if err != nil {
						c.logger.Println(err)
						return
					}
					...
			    }
		        ...
            }
        }
        ...
}
```

Before giving the usual check on unit tests, hoping that nothing broke, we can
update also **crawler\_test.go** file with some more test cases, to make sure
that our `CrawlingRules` object do his job correctly:

```go
func serverMockWithRobotsTxt() *httptest.Server {
	handler := http.NewServeMux()
	handler.HandleFunc("/robots.txt", resourceMock(
		`User-agent: *
	Disallow: */test
	Crawl-delay: 1`,
	))
	handler.HandleFunc("/", resourceMock(
		`<head>
			<link rel="canonical" href="https://example-page.com/sample-page/" />
		 </head>
		 <body>
			<img src="/baz.png">
			<img src="/stonk">
			<a href="foo/bar/baz">
		</body>`,
	))
	handler.HandleFunc("/foo/bar/baz", resourceMock(
		`<head>
			<link rel="canonical" href="https://example-page.com/sample-page/" />
			<link rel="canonical" href="/foo/bar/test" />
		 </head>
		 <body>
			<img src="/baz.png">
			<img src="/stonk">
		</body>`,
	))
	handler.HandleFunc("/foo/bar/test", resourceMock(
		`<head>
			<link rel="canonical" href="https://example-page.com/sample-page/" />
		 </head>
		 <body>
			<img src="/stonk">
		</body>`,
	))

	server := httptest.NewServer(handler)
	return server
}

func TestCrawlPagesRespectingRobotsTxt(t *testing.T) {
	server := serverMockWithRobotsTxt()
	defer server.Close()
	testbus := testQueue{make(chan []byte)}
	results := make(chan []ParsedResult)
	go func() { results <- consumeEvents(&testbus) }()
	crawler := New("test-agent", &testbus, withCrawlingTimeout(100*time.Millisecond))
	crawler.Crawl(server.URL)
	testbus.Close()
	res := <-results
	expected := []ParsedResult{
		{
			server.URL,
			[]string{"https://example-page.com/sample-page/", server.URL + "/foo/bar/baz"},
		},
		{
			server.URL + "/foo/bar/baz",
			[]string{server.URL + "/foo/bar/test"},
		},
	}
	if !reflect.DeepEqual(res, expected) {
		t.Errorf("Crawler#Crawl failed: expected %v got %v", expected, res)
	}
}
```

Now this should be the result of every unit test written so far:

```sh
go test -v ./...
=== RUN   TestCrawlPages
crawler: 2020/11/16 19:21:30 No valid 127.0.0.1:32855/robots.txt found
crawler: 2020/11/16 19:21:31 Crawling done
--- PASS: TestCrawlPages (1.20s)
=== RUN   TestCrawlPagesRespectingRobotsTxt
crawler: 2020/11/16 19:21:32 Crawling done
--- PASS: TestCrawlPagesRespectingRobotsTxt (1.21s)
=== RUN   TestCrawlPagesRespectingMaxDepth
crawler: 2020/11/16 19:21:32 No valid 127.0.0.1:42011/robots.txt found
crawler: 2020/11/16 19:21:33 Crawling done
--- PASS: TestCrawlPagesRespectingMaxDepth (1.07s)
=== RUN   TestCrawlingRules
--- PASS: TestCrawlingRules (0.00s)
=== RUN   TestCrawlingRulesNotFound
--- PASS: TestCrawlingRulesNotFound (0.00s)
PASS
ok  	github.com/codepr/webcrawler	(cached)
=== RUN   TestStdHttpFetcherFetch
--- PASS: TestStdHttpFetcherFetch (0.00s)
=== RUN   TestStdHttpFetcherFetchLinks
--- PASS: TestStdHttpFetcherFetchLinks (0.00s)
=== RUN   TestGoqueryParsePage
--- PASS: TestGoqueryParsePage (0.00s)
PASS
ok  	github.com/codepr/webcrawler/fetcher	(cached)
?   	github.com/codepr/webcrawler/messaging	[no test files]
```
