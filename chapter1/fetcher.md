# Fetching and parsing HTML contents

Now that we have a full picture of the behavior we expect our crawler should
have, we can move on to lower levels and focus on every little brick we need to
build in order to reach our goal. In this cases, application of TDD comes
naturally by adopting a bottom-up approach to the design of the system,
independent units linked together to form a bigger system, piece by piece.

The first component we're going to design and implement is the HTTP fetching
object, basically a wrapper around an HTTP client that navigates through the
HTML tree of each fetched page and extracts every link found.

Let's start with a breakdown of what we're going to need to implement our
fetcher:

- HTTP client
    * Retry mechanism
- HTML parser
    * Link extractor

## Parsing HTML documents

So we move on writing some basic unit tests to define the behavior we expect
from these two parts, starting with the HTML parser. The core feature we want
to implement is a function that ingests an HTML document and return all the
links it finds.

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

Let's now move on with the implementation, we could very well define a parser
interface exposing a single method `Parse(string, *io.Reader)([]*url.URL,
error)`.<br>
This definition of an interface foreseeing the implementation, is closer to a
classical OOP style, think about Java usage of interfaces, we're going to
define a contract to enforce a behavior; but it's not really the best and only
usage of this language feature. More on this later.

### Little dissertation over interfaces in Go

One of the strongest features of Go is that there's no need to explicitly
declare when we want to implement an interface, we just need to implement the
methods that it defines and we're good. This makes possible to build
abstractions that we foresee as useful, like in this case (classic OOP style),
but also to adapt abstractions as needed after we already worked a bit on the
problems we're trying to solve:

> *Go is an attempt to combine the safety and performance of statically typed
> languages with the convenience and fun of dynamically typed interpretative
> languages.*<br>
> ***Rob Pike***

Let's say we're about to design an object `ImageWriter` that writes binary
formatted images to disk, we just need to implement the method `Write` of the
`io.Writer` interface without explicitly declare that we're implementing it,
this way our `ImageWriter` object can be used anywhere an `io.Writer` is
accepted. At the same time let's say we have an object from a third-party
library that exposes a method `ReadLine`, we can easily declare an interface
`ReadLiner` with only a method `ReadLine` inside and use either the third-party
object (or whatever object with a `ReadLine` method) or a newly defined object
with the `ReadLine` method defined into a simple function `ReadByLine(r
ReadLiner)`.<br>
It's the principle of **accepts interfaces, return structs**[^2], in other
words if a function signature accepts an interface, then callers have the
option to pass in any concrete type, just as long as it implements that
interface. The implication is that interfaces should be declared close to where
they're used.<br> This is really akin to a duck-typing behavior at compile
time, and it's enabled by this feature of Go, making it, for some aspects,
really similar to dynamic languages like Python or Ruby.

So to start grasping the problem we prefer to just start with a concrete
implementation, we're going to decide later where and when (and if) we're
going to need a general abstraction.

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
```

{% hint style="info" %}
Go does not natively support the set data structure, the idiomatic way to
implement it is to use a map with `bool` as value type
{% endhint %}

`GoqueryParser` could very well been an empty struct, as long as it implements
the parse method we're good, but the object is meant to be shared to multiple
workers and it's advisable to filter out repeated links already. For a simple
list of exclusion we use a set (a map with `bool` value), representing the file
extensions that we don't want to extract from each HTML document we fetch. The
`seen` map is a concurrent implementation of a map from the `sync` package,
its an optimized version of a simple map guarded by mutex.

Let's implement the parse method with the goquery library:

**fetcher/parser.go**

```go

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
// It returns a slice of string containing all the extracted links or `nil` if
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

```
go test -v ./...
```

{% hint style="info" %}
Command go test, just like go run and go build, takes care of downloading
dependencies and updating go.mod file automatically.
{% endhint %}

## Fetching HTML documents

A web crawler operates on the 7th layer, as simple as that, the main
communication protocol used to fetch outside contents from websites is HTTP,
so the core component of the `fetcher` package will be an HTTP client.

The next step is the definition of fetching unit tests, what we expect here is
the possibility to simply fetch a single link, ignoring its content and
fetching a link extracting all contained links.

We're going to mock a server probably, luckily Go provides us everything we
need in the `net/http/httptest`.

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
    "io"
	"net/http"
	"net/url"
	"time"

	"github.com/PuerkitoBio/rehttp"
)

// Parser is an interface exposing a single method `Parse`, to be used on
// raw results of a fetch call
type Parser interface {
	Parse(string, io.Reader) ([]*url.URL, error)
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

Now that we have a simple `Fetch` method, we can easily extend that
behavior to fetch the page content and extract all the contained links, finally
using that `Parser` interface we have inserted into the `stdHttpFetcher`.

Go's approach to abstractions through interfaces is immediately highlighted in
the snippet above: we declared a `Parser` interface **after** the concrete
implementation of `GoqueryParser`, as we previously discussed, this is somewhat
confusing for ordinary OOP patterns of pre-declaring interfaces as contract,
enforcing a behavior for every concrete type (like in Java for example).<br>
Languages that requires explicit implementation of interfaces promote that
style, where you want to predict the abstraction you're going to implement. Go
instead allows to implement interfaces implicitly by just implementing their
methods. This unlocks a more flexible style, which promotes a better
understanding of the problem before abstracting away details; thus in Go
interfaces is generally declared closer to their clients. This way it's the
client that dictates the abstraction needed and not the other way around,
making the entire development process more flixible.

*Note: the `Fetch` and `FetchLinks` methods, return also a `time.Duration` as
first value, that is the wall time of the call, telling us the time elapsed to
make the HTTP call. It'll return useful later for calculating a politeness
delay during the crawling process*

**fetcher/fetcher.go**

```go
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
	if err != nil {
		return elapsed, nil, fmt.Errorf("fetching links from %s failed: %w", targetURL, err)
	}
	defer resp.Body.Close()
    if resp.StatusCode >= http.StatusBadRequest {
		return elapsed, nil, fmt.Errorf("fetching links from %s failed: %s", targetURL, resp.Status)
    }
	links, err := f.parser.Parse(baseDomain, resp.Body)
	if err != nil {
		return elapsed, nil, fmt.Errorf("fetching links from %s failed: %w", targetURL, err)
	}
	return elapsed, links, nil
}
```

Running the simple unit tests we written should result in a success outcome

```
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

```
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

[^2]: [https://commandercoriander.net/blog/2018/03/30/go-interfaces/](https://commandercoriander.net/blog/2018/03/30/go-interfaces/)
