# Running the crawler

At long last we can give a try to the application with a real test. We're going
to add a `main` file inside the `cmd` folder.

To test the application we define a simple `ProducerConsumer` queue channel
based, which will print all fetched links, using a tab to list all children
at every level.

**cmd/webcrawler/main.go**

```go
package main

import (
	"encoding/json"
	"flag"
	"log"

	"webcrawler"
	"webcrawler/fetcher"
	"webcrawler/messaging"
)

const (
	// Default depth to crawl for each domain
	defaultDepth int = 16
	// Default number of concurrent goroutines to crawl
	defaultConcurrency int = 8
)

// printEvents is a simple ChannelQueue consumer, just print received results
// from the crawler on stdout, simulate a decoupled process meant to process
// incoming events from the crawler
func printEvents(queue *messaging.ChannelQueue) {
	events := make(chan []byte)
	go func(ch <-chan []byte) {
		var res crawler.ParsedResult
		for e := range ch {
			if err := json.Unmarshal(e, &res); err == nil {
				log.Println(res.URL)
				for _, link := range res.Links {
					log.Println("\t", link)
				}
			}
		}
	}(events)
	if err := queue.Consume(events); err != nil {
		log.Fatal(err)
	}
}

// withMaxDepth is a simple constructor option to pass into the
// crawler.New function call to set the number of levels to crawl
// for each page
func withMaxDepth(depth int) crawler.CrawlerOpt {
	return func(s *crawler.CrawlerSettings) {
		s.MaxDepth = depth
	}
}

// withConcurrency is a simple constructor option to pass into the
// crawler.New function call to set the concurrency level
func withConcurrency(concurrency int) crawler.CrawlerOpt {
	return func(s *crawler.CrawlerSettings) {
		s.Concurrency = concurrency
	}
}

func main() {
	var (
		targetURL   string
		maxDepth    int
		concurrency int
	)
	flag.StringVar(&targetURL, "target", "", "URL to crawl")
	flag.IntVar(&maxDepth, "depth", defaultDepth, "Maximum depth of crawling")
	flag.IntVar(&concurrency, "concurrency", defaultConcurrency, "Number of concurrent goroutine to run")
	flag.Parse()
	// We create a ChannelQueue instance here, ideally it could be a
	// RabbitMQ/AWS SQS task queue
	bus := messaging.NewChannelQueue()
    userAgent string = "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"
	go func() { printEvents(&bus) }()
	c := crawler.New(userAgent, &bus,
		withMaxDepth(maxDepth),
		withConcurrency(concurrency),
	)
	c.Crawl(targetURL)
}
```

```
go run cmd/webcrawler/main.go -target golang.org
```
