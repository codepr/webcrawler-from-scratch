# Fetching and parsing HTML contents

Now that we have a top-down picture of the behavior we expect our crawler have
to implement we can move on to lower levels and focus on every little brick we
need to create in order to build our final product.

The first component we're going to design and implement is the HTTP fetching
object, in the simplest form a wrapper around an HTTP client that navigate
through the HTML tree of each fetched page and extracts every link found.

Let's start with a breakdown of what we're going to need to implement our
fetcher:

- HTTP client
- HTML parser

## Parsing HTML documents

So we move on writing some basic unit tests to define the behavior we expect
from these 2 parts, starting with the HTML parser.

*Note: After a brief search I found out that GoQuery by PuerkitoBio is the
easiest and most handy library to parse HTML contents offering a jquery-like
DSL to navigate through the entire tree. The alternative was the navigation by
hand and probably some regex, not worth the hassle given the purpose of the
project*

**fetcher/parser\_test.go**

```go
package fetcher

import (
	"bytes"
	"net/url"
	"reflect"
	"testing"
)

func TestGoqueryParsePage(t *testing.T) {
	parser := NewGoqueryParser()
	firstLink, _ := url.Parse("https://example-page.com/sample-page/")
	secondLink, _ := url.Parse("http://localhost:8787/sample-page/")
	thirdLink, _ := url.Parse("http://localhost:8787/foo/bar")
	expected := []*url.URL{firstLink, secondLink, thirdLink}
	content := bytes.NewBufferString(
		`<head>
			<link rel="canonical" href="https://example-page.com/sample-page/" />
			<link rel="canonical" href="http://localhost:8787/sample-page/" />
		 </head>
		 <body>
			<a href="foo/bar"><img src="/baz.png"></a>
			<img src="/stonk">
			<a href="foo/bar">
		</body>`,
	)
	res, err := parser.Parse("http://localhost:8787", content)
	if err != nil {
		t.Errorf("GoqueryParser#ParsePage failed: expected %v got %v", expected, err)
	}
	if !reflect.DeepEqual(res, expected) {
		t.Errorf("GoqueryParser#ParsePage failed: expected %v got %v", expected, res)
	}
}
```

Let's now move on with the implementation, we're going to define a parser
interface exposing a single method `Parse(string, *io.Reader)([]*url.URL,
error)`.

One of the strongest features of Go is that there's no need to explicitly
declare when we want to implement an interface, we just need to implement the
methods that it defines and we're good to go. This makes possible to build
abstractions that we foresee as useful, like in this case (classic OOP style),
but also to adapt abstractions as needed after we already worked a bit on the
problems we're trying to solve:

> *Go is an attempt to combine the safety and performance of statically typed
> languages with the convenience and fun of dynamically typed interpretative
> languages.*<br>
> ***Rob Pike***

Let's say we're about to implement an object `ImageWriter` that writes binary
formatted images to disk, we just need to write the method `Write` of the
`io.Writer` interface without explicitly declare that we're implementing it. At
the same time let's say we have an object from a third-party library that
exposes a method `ReadLine` we can easily declare an interface `ReadLiner` with
only a method `ReadLine` inside and use either the third-party object (or
whatever object with a `ReadLine` method) or a newly defined object with the
`ReadLine` method defined into a simple function `ReadByLine(r ReadLiner)`.
This is really akin to a duck-typing behavior at compile time, and it's enabled
by this feature of go.

**fetcher/parser.go**

```go
// Package fetcher defines and implement the fetching and parsing utilities
// for remote resources
package fetcher

import (
	"io"
	"net/url"
	"path/filepath"
	"sync"

	"github.com/PuerkitoBio/goquery"
)

// Parser is an interface exposing a single method `Parse`, to be used on
// raw results of a fetch call
type Parser interface {
	Parse(string, io.Reader) ([]*url.URL, error)
}

// GoqueryParser is just an algorithm `Parser` definition that uses
// `github.com/PuerkitoBio/goquery` as a backend library
type GoqueryParser struct {
	excludedExts map[string]bool
	seen         *sync.Map
}

// NewGoqueryParser create a new parser with goquery as backend
func NewGoqueryParser() GoqueryParser {
	return GoqueryParser{
		excludedExts: make(map[string]bool),
		seen:         new(sync.Map),
	}
}

// ExcludeExtensions add extensions to be excluded to the default exclusion
// pool
func (p *GoqueryParser) ExcludeExtensions(exts ...string) {
	for _, ext := range exts {
		p.excludedExts[ext] = true
	}
}

// Parse is the implementation of the `Parser` interface for the
// `GoqueryParser` struct, read the content of an `io.Reader` (e.g.
// any file-like streamable object) and extracts all anchor links.
// It returns a `ParserResult` object or any error that arises from the goquery
// call on the data read.
func (p GoqueryParser) Parse(baseURL string, reader io.Reader) ([]*url.URL, error) {
	doc, err := goquery.NewDocumentFromReader(reader)
	if err != nil {
		return nil, err
	}
	links := p.extractLinks(doc, baseURL)
	return links, nil
}

// extractLinks retrieves all anchor links inside a `goquery.Document`
// representing an HTML content.
// It returns a slice of string containing all the extracted links or `nil` if\
// the passed document is a `nil` pointer.
func (p *GoqueryParser) extractLinks(doc *goquery.Document, baseURL string) []*url.URL {
	if doc == nil {
		return nil
	}
	foundURLs := []*url.URL{}
	doc.Find("a,link").FilterFunction(func(i int, element *goquery.Selection) bool {
		hrefLink, hrefExists := element.Attr("href")
		linkType, linkExists := element.Attr("rel")
		anchorOk := hrefExists && !p.excludedExts[filepath.Ext(hrefLink)]
		linkOk := linkExists && linkType == "canonical" && !p.excludedExts[filepath.Ext(linkType)]
		return anchorOk || linkOk
	}).Each(func(i int, element *goquery.Selection) {
		res, _ := element.Attr("href")
		if link, ok := resolveRelativeURL(baseURL, res); ok {
			if present, _ := p.seen.LoadOrStore(link.String(), false); !present.(bool) {
				foundURLs = append(foundURLs, link)
				p.seen.Store(link.String(), true)
			}
		}
	})
	return foundURLs
}

// resolveRelativeURL just correctly join a base domain to a relative path
// to produce an absolute path to fetch on.
// It returns a tuple, a string representing the absolute path with resolved
// paths and a boolean representing the success or failure of the process.
func resolveRelativeURL(baseURL string, relative string) (*url.URL, bool) {
	u, err := url.Parse(relative)
	if err != nil {
		return nil, false
	}
	if u.Hostname() != "" {
		return u, true
	}
	base, err := url.Parse(baseURL)
	if err != nil {
		return nil, false
	}

	return base.ResolveReference(u), true
}
```

Well, let's try running those tests, hopefully they'll give a positive outcome:

```sh
go test -v ./...
```

## Fetching HTML documents

The next step is the definition of fetching unit tests, what we expect here is
the possibility to simply fetch a single link, ignoring its content and
fetching a link extracting all contained links.<br>We're probably going to
defines 2 interfaces for these tasks, `Fetcher` and `LinkFetcher`.

**fetcher/fetcher\_test.go**

```go
package fetcher

import (
	"fmt"
	"net/http"
	"net/http/httptest"
	"net/url"
	"reflect"
	"testing"
	"time"
)

func serverMock() *httptest.Server {
	handler := http.NewServeMux()
	handler.HandleFunc("/foo/bar", resourceMock)

	server := httptest.NewServer(handler)
	return server
}

func resourceMock(w http.ResponseWriter, r *http.Request) {
	_, _ = w.Write([]byte(
		`<head>
			<link rel="canonical" href="https://example.com/sample-page/" />
			<link rel="canonical" href="/sample-page/" />
		 </head>
		 <body>
			<a href="foo/bar"><img src="/baz.png"></a>
			<img src="/stonk">
			<a href="foo/bar">
		 </body>`,
	))
}

func TestStdHttpFetcherFetch(t *testing.T) {
	server := serverMock()
	defer server.Close()
	f := New("test-agent", nil, 10*time.Second)
	target := fmt.Sprintf("%s/foo/bar", server.URL)
	_, res, err := f.Fetch(target)
	if err != nil {
		t.Errorf("StdHttpFetcher#Fetch failed: %v", err)
	}
	if res.StatusCode != 200 {
		t.Errorf("StdHttpFetcher#Fetch failed: %#v", res)
	}
	_, res, err = f.Fetch("testUrl")
	if err == nil {
		t.Errorf("StdHttpFetcher#Fetch failed: %v", err)
	}
}

func TestStdHttpFetcherFetchLinks(t *testing.T) {
	server := serverMock()
	defer server.Close()
	f := New("test-agent", NewGoqueryParser(), 10*time.Second)
	target := fmt.Sprintf("%s/foo/bar", server.URL)
	firstLink, _ := url.Parse("https://example.com/sample-page/")
	secondLink, _ := url.Parse(server.URL + "/sample-page/")
	thirdLink, _ := url.Parse(server.URL + "/foo/bar")
	expected := []*url.URL{firstLink, secondLink, thirdLink}
	_, res, err := f.FetchLinks(target)
	if err != nil {
		t.Errorf("StdHttpFetcher#FetchLinks failed: expected %v got %v", expected, err)
	}
	if !reflect.DeepEqual(res, expected) {
		t.Errorf("StdHttpFetcher#FetchLinks failed: expected %v got %v", expected, res)
	}
}
```

Ok, we just need to write some logic now to make those tests pass again.

Our first move will be the definition of a standard HTTP client wrapper,
encapsulating parsing capabilities through a parser interface dependency, its
first feature will be the simple fetching of a link. We also want to add some
kind of re-try mechanism in case of failure, maintaining a degree of politeness
toward the target, a simple exponential backoff between calls is enough for
now, [rehttp](https://github.com/PuerkitoBio/rehttp) library allow us to do it
gracefully, once again courtesy of PuerkitoBio work.

*Note: we just ignore any possible invalid TLS certificates, in a real
production case this could represent a security issue, and we'd strongly prefer
to trace when something like that happens.*

**fetcher/fetcher.go**

```go
// Package fetcher defines and implement the downloading and parsing utilities
// for remote resources
package fetcher

import (
	"crypto/tls"
	"fmt"
	"net/http"
	"net/url"
	"time"

	"github.com/PuerkitoBio/rehttp"
)

// Fetcher is an interface exposing a method to fetch resources, Fetch enable
// raw contents download.
type Fetcher interface {
	// Fetch makes an HTTP GET request to an URL returning a `*http.Response` or
	// any error occured
	Fetch(string) (time.Duration, *http.Response, error)
}

// stdHttpFetcher is a simple Fetcher with std library http.Client as a
// backend for HTTP requests.
type stdHttpFetcher struct {
	userAgent string
	parser    Parser
	client    *http.Client
}

// New create a new Fetcher specifying an user-agent to set on each call,
// a parser interface to parse HTML contents and a timeout.
// By default it retries when a temporary error occurs (most temporary
// errors are HTTP ones) for a specified number of times by applying an
// exponential backoff strategy.
func New(userAgent string, parser Parser, timeout time.Duration) *stdHttpFetcher {
	transport := rehttp.NewTransport(
		&http.Transport{
			TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
		},
		rehttp.RetryAll(rehttp.RetryMaxRetries(3), rehttp.RetryTemporaryErr()),
		rehttp.ExpJitterDelay(1, 10*time.Second),
	)
	client := &http.Client{Timeout: timeout, Transport: transport}
	return &stdHttpFetcher{userAgent, parser, client}
}

// Fetch is a private function used to make a single HTTP GET request
// toward an URL.
// It returns an `*http.Response` or any error occured during the call.
func (f stdHttpFetcher) Fetch(url string) (time.Duration, *http.Response, error) {

	req, err := http.NewRequest("GET", url, nil)
	if err != nil {
		return time.Duration(0), nil, err
	}
	req.Header.Set("User-Agent", f.userAgent)
	// We want to time the request
	start := time.Now()
	res, err := f.client.Do(req)
	elapsed := time.Since(start)
	if err != nil {
		return elapsed, nil, err
	}

	return elapsed, res, nil
}
```

Now that we have a simple `Fetcher` interface, we can easily extend his
behavior to fetch the content page and extract all the contained links, finally
using that `parser` interface we have inserted into the `stdHttpFetcher`.

We want to maintain the separation between `Fetcher` and `LinkFetcher`
interfaces in order to maintain single responsibilities to each component and
because we're going to need the simple HTTP request capability later.

*Note: in the `Fetcher` interface and in the `LinkFetcher` one, both exported
methods return also a `time.Duration` as first value, that is the wall time of
the call, telling us the time elapsed to make the HTTP call. It'll return
useful later for calculating a politeness delay during the crawling process*

**fetcher/fetcher.go**

```go
// LinkFetcher is an interface exposing a method to download raw contents and
// parse them extracting all outgoing links.
type LinkFetcher interface {
	// FetchLinks makes an HTTP GET request to an URL, parse the HTML in the
	// response and returns an array of URLs or any error occured
	FetchLinks(string) (time.Duration, []*url.URL, error)
}

// Parse an URL extracting the protion <scheme>://<host>:<port>
// Returns a string with the base domain of the URL
func parseStartURL(u string) string {
	parsed, _ := url.Parse(u)
	return fmt.Sprintf("%s://%s", parsed.Scheme, parsed.Host)
}

// Fetch contact and download raw data from a specified URL and parse the
// content into a `ParserResult` struct.
// It returns a `*ParserResult` or any error occuring during the call or the
// parsing of the results.
func (f stdHttpFetcher) FetchLinks(targetURL string) (time.Duration, []*url.URL, error) {
	if f.parser == nil {
		return time.Duration(0), nil, fmt.Errorf("fetching links from %s failed: no parser set", targetURL)
	}
	// Extract base domain from the url
	baseDomain := parseStartURL(targetURL)

	elapsed, resp, err := f.Fetch(targetURL)
	if err != nil || resp.StatusCode >= http.StatusBadRequest {
		return elapsed, nil, fmt.Errorf("fetching links from %s failed: %w", targetURL, err)
	}
	defer resp.Body.Close()

	links, err := f.parser.Parse(baseDomain, resp.Body)
	if err != nil {
		return elapsed, nil, fmt.Errorf("fetching links from %s failed: %w", targetURL, err)
	}
	return elapsed, links, nil
}
```

Running the simple unit tests we written should result in a success outcome

```sh
go test -v ./...
=== RUN   TestStdHttpFetcherFetch
--- PASS: TestStdHttpFetcherFetch (0.00s)
=== RUN   TestStdHttpFetcherFetchLinks
--- PASS: TestStdHttpFetcherFetchLinks (0.00s)
=== RUN   TestGoqueryParsePage
--- PASS: TestGoqueryParsePage (0.00s)
PASS
ok  	webcrawler/fetcher	0.006s
```

The project structure should be the following

```sh
tree
.
├── fetcher
│   ├── fetcher.go
│   ├── fetcher_test.go
│   ├── parser.go
│   └── parser_test.go
├── go.mod
└── go.sum
```
