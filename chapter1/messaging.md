# Introducing a middleware

Middlewares are basically every piece of software that is used as a medium, a
communication channel between different and decoupled components in an
architecture, message queues, message brokers, buses are effectively all
middlewares.

Many times they're used in microservices architectures, think about asynchronous
communication between different services, where you just want to schedule some
kind of jobs without worrying of the immediate response, `RabbitMQ` or `SQS` is
often used in these scenarios.<br>
In our case, we're going to add a really simple message queue interface to
decouple the crawling logic of the web crawler, this, along with decoupling
responsibilities, comes with the benefit of a simpler testing of the entire
application.

## Producer and consumer

The most famous and simple abstraction is the producer-consumer pattern, where
two actors are involved:

* The producer, generates traffic
* The consumer, consumes the traffic

It's a simplification of the pub-sub pattern, but can be easily extended and
behave just like it, with multiple subscribers consuming the same source.

> *The bigger the interface, the weaker the abstraction*<br>
> ***Rob Pike***

Go is conceptually a minimalist-oriented language, or at least that's how I
mostly perceive it, so the best practice is to maintain the interfaces as
little as possible, and it really makes sense as every interface defines an
enforcement, a contract that must be fulfilled, the less you have to implement
to be compliant the better.

We've seen how Go best approach to abstractions is to not abstract at all until
needed, good news is that, given the inherent pragmatism of the language, we're
allowed to break the rules sometimes, according to common sense.<br>
In this case, we're reasonably sure that for our application we're going to
need a simple communication channel that enables two operations:

* Production
* Consumption

But for the sake of readability and extensibility for future additions we'll
stick to Rob Pike's quote by adding three generic communication interfaces:

**messaging/queue.go**

```go
// Package messaging contains middleware for communication with decoupled
// services, could be RabbitMQ drivers as well as kafka or redis
package messaging

// Producer defines a producer behavior, exposes a single `Produce` method
// meant to enqueue an array of bytes
type Producer interface {
	Produce([]byte) error
}

// Consumer defines a consumer behavior, exposes a single `Consume` method
// meant to connect to a queue blocking while consuming incoming arrays of
// bytes forwarding them into a channel
type Consumer interface {
	Consume(chan<- []byte) error
}

// ProducerConsumer defines the behavior of a simple message queue, it's
// expected to provide a `Produce` function a `Consume` one
type ProducerConsumer interface {
	Producer
	Consumer
}

// ProducerConsumerCloser defines the behavior of a simple mssage queue
// that requires some kidn of external connection to be managed
type ProducerConsumerCloser interface {
	ProducerConsumer
	Close()
}
```

{% hint style="info" %}
Channels as function arguments can be specified also with the direction that
they're meant to be used, be it send-only, receive-only or both.
{% endhint %}

Producer and consumer are meant to work with the most generic type beside
`interface{}`, the `[]byte` is usually an exchange format made out of
encoding of data, be it binary or JSON it is possible to obtain a byte array
representation of the data.<br>
The consumer exported method `Consume(chan <- []byte) error` accepts a
send-only channel, it's the channel that we're going to use to obtain our
items, the method is in-fact thought as a blocking call whereas another
goroutine is receiving on the other end of the channel.

From now on these are our gates and pipes to communicate, it'll be possible to
create multiple different structs, like `RabbitMQ` or `Redis` backed to pass
around bits from the crawling logic to other clients.

And that's exactly what we're going to do to make it testable, we'll introduce
a fool-proof struct encapsulating a simple channel as communication backend
directly inside our test files:

**crawler_test.go**
```go
// Package crawler containing the crawling logics and utilities to scrape
// remote resources
package crawler

import (
	"encoding/json"
	"io/ioutil"
	"log"
	"net/http"
	"net/http/httptest"
	"os"
	"reflect"
	"sync"
	"testing"
	"time"
)

type testQueue struct {
	bus chan []byte
}

func (t testQueue) Produce(data []byte) error {
	t.bus <- data
	return nil
}

func (t testQueue) Consume(events chan<- []byte) error {
	for event := range t.bus {
		events <- event
	}
	return nil
}

func (t testQueue) Close() {
	close(t.bus)
}

func consumeEvents(queue *testQueue) []ParsedResult {
	wg := sync.WaitGroup{}
	events := make(chan []byte)
	results := []ParsedResult{}
	wg.Add(1)
	go func() {
		defer wg.Done()
		for e := range events {
			var res ParsedResult
			if err := json.Unmarshal(e, &res); err == nil {
				results = append(results, res)
			}
		}
	}()
	_ = queue.Consume(events)
	close(events)
	wg.Wait()
	return results
}

func serverMockWithoutRobotsTxt() *httptest.Server {
	handler := http.NewServeMux()
	handler.HandleFunc("/foo", resourceMock(
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

func resourceMock(content string) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		_, _ = w.Write([]byte(content))
	}
}

func TestMain(m *testing.M) {
	log.SetOutput(ioutil.Discard)
	os.Exit(m.Run())
}

func withMaxDepth(depth int) CrawlerOpt {
	return func(s *CrawlerSettings) {
		s.MaxDepth = depth
	}
}

func withCrawlingTimeout(timeout time.Duration) CrawlerOpt {
	return func(s *CrawlerSettings) {
		s.CrawlingTimeout = timeout
	}
}

func TestCrawlPages(t *testing.T) {
	server := serverMockWithoutRobotsTxt()
	defer server.Close()
	testbus := testQueue{make(chan []byte)}
	results := make(chan []ParsedResult)
	go func() { results <- consumeEvents(&testbus) }()
	crawler := New("test-agent", &testbus, withCrawlingTimeout(100*time.Millisecond))
	crawler.Crawl(server.URL + "/foo")
	testbus.Close()
	res := <-results
	close(results)
	expected := []ParsedResult{
		{
			server.URL + "/foo",
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

func TestCrawlPagesRespectingMaxDepth(t *testing.T) {
	server := serverMockWithoutRobotsTxt()
	defer server.Close()
	testbus := testQueue{make(chan []byte)}
	results := make(chan []ParsedResult)
	go func() { results <- consumeEvents(&testbus) }()
	crawler := New("test-agent", &testbus, withCrawlingTimeout(100*time.Millisecond), withMaxDepth(3))
	crawler.Crawl(server.URL + "/foo")
	testbus.Close()
	res := <-results
	expected := []ParsedResult{
		{
			server.URL + "/foo",
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

*Note: the only function mocking resources server side is called
`serverMockWithoutRobotsTxt()`, this is because right now we're not
considering the existence of a `robots.txt` file on the root of each domain,
but in the next chapter we'll handle that set of rules as well, discussing
crawling politeness*

In order to make them pass we need to adapt the `crawlPage` method to use a
`Producer` implementation to send out every crawled URL, so let's open
**crawler.go** file and update the `WebCrawler` struct, its constructor and
add the forwarding code into the main `crawlPage` loop:

```diff
type WebCrawler struct {
	// logger is a private logger instance
	logger *log.Logger
+	// queue is a simple message queue to forward crawling results to other
+	// components of the architecture, decoupling business logic from processing,
+	// storage or presentation layers
+	queue messaging.Producer
    // linkFetcher is a LinkFetcher object, must expose Fetch and FetchLinks methods
	linkFetcher LinkFetcher
	// settings is a pointer to `CrawlerSettings` containing some crawler
	// specifications
	settings *CrawlerSettings
}
```

The constructor will be updated as well

```diff
-func New(userAgent string, opts ...CrawlerOpt) *WebCrawler {
+func New(userAgent string, queue messaging.Producer, opts ...CrawlerOpt) *WebCrawler {
    ...
	crawler := &WebCrawler{
		logger:   log.New(os.Stderr, "crawler: ", log.LstdFlags),
+		queue:    queue,
		linkFetcher: fetcher.New(userAgent, settings.Parser, settings.FetchTimeout),
		settings: settings,
	}
	return crawler
}
```

Finally the `crawlPage` private method, we want that after every link has been
extracted, it goes into the queue by using the `Producer` interface, for the
format we're just encoding them into JSON by using the standard library:

**crawler.go**

```diff
func (c *WebCrawler) crawlPage(rootURL *url.URL, wg *sync.WaitGroup, ctx context.Context) {
    ...
    for !stop {
        ...
        go func(link *url.URL, stopSentinel bool, w *sync.WaitGroup) {
            ...
            // No errors occured, we want to enqueue all scraped links
            // to the link queue
            if stopSentinel || foundLinks == nil || len(foundLinks) == 0 {
                return
            }
            atomic.AddInt32(&linkCounter, int32(len(foundLinks)))
+			// Send results from fetch process to the processing queue
+			c.enqueueResults(link, foundLinks)
            // Enqueue found links for the next cycle
            linksCh <- foundLinks
        }(link, stop, &fetchWg)
    }
	fetchWg.Wait()
}

+// enqueueResults enqueue fetched links through the ProducerConsumer queue in
+// order to be processed (in this case, printe to stdout)
+func (c *WebCrawler) enqueueResults(link *url.URL, foundLinks []*url.URL) {
+	foundLinksStr := []string{}
+	for _, l := range foundLinks {
+		foundLinksStr = append(foundLinksStr, l.String())
+	}
+	payload, _ := json.Marshal(ParsedResult{link.String(), foundLinksStr})
+	if err := c.queue.Produce(payload); err != nil {
+		c.logger.Println("Unable to communicate with message queue:", err)
+	}
+}
```
And we're good to go, we should be all green running a go test now:

```
go test -v ./...
=== RUN   TestCrawlPages
--- PASS: TestCrawlPages (1.20s)
=== RUN   TestCrawlPagesRespectingMaxDepth
--- PASS: TestCrawlPagesRespectingMaxDepth (1.07s)
PASS
ok  	webcrawler	3.495s
=== RUN   TestStdHttpFetcherFetch
--- PASS: TestStdHttpFetcherFetch (0.00s)
=== RUN   TestStdHttpFetcherFetchLinks
--- PASS: TestStdHttpFetcherFetchLinks (0.00s)
=== RUN   TestGoqueryParsePage
--- PASS: TestGoqueryParsePage (0.00s)
PASS
ok  	webcrawler/fetcher	(cached)
?   	webcrawler/messaging	[no test files]
```

In the next chapter we're going to explore politeness concept, or how a good
web crawler should behave while visiting a domain and `robots.txt` rules that
most of the time are inserted on the root of the site.
