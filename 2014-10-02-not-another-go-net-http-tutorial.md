# Not Another Go/Golang net/http Tutorial
---
Before we get into it, I'd like to provide a little bit of a preface for the 
motivation behind writing this tutorial. A few months back, Brent Anderson 
created the meetup.com group for [Minnesota's own Go User Meetup](
http://www.meetup.com/golangmn/); and, a couple months later, I set about 
organizing the group and events. One of our own members, Craig Erdmann, and his 
trusty sidekick Jen Rajkumar offered to host and cater the meetup.

So I set about writing the presentation *I* wanted to see -- not what the group 
needed. We were going to walk through a pretty obfuscated example from 
[Hoare's extremely influential 'Communicating Sequential Processes'](
http://www.cs.cmu.edu/~crary/819-f09/Hoare78.pdf), which [someone had 
implemented in Go](
https://github.com/thomas11/csp/blob/master/csp.go#L236-L290). After asking for 
feedback though, it quickly became apparent that we had many new entrants; and, 
as a result of quickly rewriting the presentation, it was a pretty poor 
experience for the newcomers. 

The idea with this post is to create interesting content for those who have 
used go net/http in the past, but keep it easy-to-follow for beginners.

Side note: For those of you who have contacted me asking to write about 
specific topics (i.e.: Setting up a hadoop cluster, using Mahout's machine 
learning algorithms, using Groupcache as a replacement for Redis, etc.), I 
really appreciate your requests and still plan on writing them - they're 
somewhere in the heap.

## The Ubiquitous Example
---
We've all seen this example, either from the [Golang wiki](
https://golang.org/doc/articles/wiki/#tmp_3), or from blog posts regurgitating
 the same (wonder who does that?):

```go
	package main
	
	import (
		"fmt"
		"net/http"
	)
	
	func handler(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
	}
	
	func main() {
		http.HandleFunc("/", handler)
		http.ListenAndServe(":8080", nil)
	}
```

For those of us that haven't, what's going on here is:

- First, we import the ["net/http"](http://golang.org/pkg/net/http/) package.
- Then, we see a function definition for a function called `handler`.
    - As arguments, `handler` takes an http.ResponseWriter and a pointer to an 
    `http.Request`.
- Next, in our `main()` function, we use `http.HandleFunc(<path>, <function>)` 
to direct all requests to `"/"` to our `handler` function.
- Finally, we start a server listening on `localhost:8080`, with no specified 
handler (this results in using the built-in `DefaultServeMux` as our 
multiplexer).

The wiki goes on to talk about how this works at a high level, but let's take a 
deeper look.

#### http.ResponseWriter && http.Request
---
Our `handler` function takes two arguments: One of type [http.ResponseWriter](
http://golang.org/pkg/net/http/#ResponseWriter), and the other a pointer to a 
[http.Request](http://golang.org/pkg/net/http/#Request) type.

The [source](http://golang.org/src/pkg/net/http/server.go?s=1255:2131#L41) for 
the `http.ResponseWriter` shows that it's an interface type defined as:

```go
    // Commentary:
	// A ResponseWriter interface is used by an HTTP handler to
	// construct an HTTP response.
	type ResponseWriter interface {
    // End Commentary
            // Commentary:
	        // Header returns the header map that will be sent by WriteHeader.
	        // Changing the header after a call to WriteHeader (or Write) has
	        // no effect.
	        Header() Header
            // End Commentary
	
            // Commentary:
	        // Write writes the data to the connection as part of an HTTP reply.
	        // If WriteHeader has not yet been called, Write calls WriteHeader(http.StatusOK)
	        // before writing the data.  If the Header does not contain a
	        // Content-Type line, Write adds a Content-Type set to the result of passing
	        // the initial 512 bytes of written data to DetectContentType.
	        Write([]byte) (int, error)
            // End Commentary
	
            // Commentary:
	        // WriteHeader sends an HTTP response header with status code.
	        // If WriteHeader is not called explicitly, the first call to Write
	        // will trigger an implicit WriteHeader(http.StatusOK).
	        // Thus explicit calls to WriteHeader are mainly used to
	        // send error codes.
	        WriteHeader(int)
            // End Commentary
	}
```

So, we know that `w` (our `http.ResponseWriter` variable in our `handler` 
function arguments) is pretty simple - it really only *needs* to conform to a 
role of doing three things: 

- Being able to send a response header with status code back to the connection
- The ability to write data (received by `Write()` as a `[]byte`) back to the 
connection held
- The ability to return the header map to be used by WriteHeader

That last set of bullets may seem unnecessary, as it's sort of in the comments, 
but it helps us distill the source down to the idea that instead of:

```go
	func handler(w http.ResponseWriter, r *http.Request) {
	    fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
	}
```

we could potentially do this in our `handler` function:

```go
	func handler(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json; charset=utf-8") 

		myItems := []string{"item1", "item2", "item3"}
		a, _ := json.Marshal(myItems)

		w.Write(a)
		return
	}
```

Pretty neat - we're setting an HTTP Header, marshalling a string slice to JSON,
 and writing that out to our http connection.

If `http.ResponseWriter` is for handling our response on the connection, the 
`http.Request` is for the other half of the equation - the initial request. 
It's type is actually `struct`, and it has a [definition a bit lengthier](
http://golang.org/src/pkg/net/http/request.go?s=2132:8067#L66) than that of 
`http.ResponseWriter`, so I've reproduced it without most comments here:

```go
    // Commentary:
	// A Request represents an HTTP request received by a server
	// or to be sent by a client.
	//
	// The field semantics differ slightly between client and server
	// usage. In addition to the notes on the fields below, see the
	// documentation for Request.Write and RoundTripper.
	type Request struct {
	        Method string
	        URL *url.URL
	        Proto      string // "HTTP/1.0"
	        ProtoMajor int    // 1
	        ProtoMinor int    // 0
	        Header Header
	        Body io.ReadCloser
	        ContentLength int64
	        TransferEncoding []string
	        Close bool
	        Host string
	        Form url.Values
	        PostForm url.Values
	        MultipartForm *multipart.Form
	        Trailer Header
	        RemoteAddr string
	        RequestURI string
	        TLS *tls.ConnectionState
	}
    // End Commentary
```

Suffice it to say that if any request parameters, data, etc. had come 
through, we could retrieve and parse them from that `http.Request` pointer.

#### ServeMux
---

In our example, we're not making use of the `http.Request` pointer argument at 
all -- why then, do we need to include it as an argument to *our* `handler` 
function? To simplify the answer, we're conforming to the requirements 
defined by the `DefaultServeMux` multiplexer. You might be wondering where the 
hell the code for that came from - we haven't seen any declaration of this 
`DefaultServeMux`, or anything; but, when we passed in `nil` to our

{% highlight go %}
	http.ListenAndServe(":8080", nil)
{% endhighlight %}

call, we implicitly asked ListenAndServe to multiplex incoming connections over 
the Default Serve Multiplexer, which will take care of routing requests to the 
correct `handler` functions, as long as we've *"registered"* them appropriately. 

We've only registered a handler function for the `"/"` path with the 
`DefaultServeMux`. In order to do that, though, we needed to call 
`http.HandleFunc`, which is [defined as](
http://golang.org/src/pkg/net/http/server.go?s=45785:45856#L1556):

```go
    // Commentary:
	// HandleFunc registers the handler function for the given pattern
	// in the DefaultServeMux.
	// The documentation for ServeMux explains how patterns are matched.
	func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
		DefaultServeMux.HandleFunc(pattern, handler)
	}
    // End Commentary
```

As an aside: I'd like to point out how easy it is to reason about the code in 
the built-in packages. Comparing that to the actual call to `http.HandleFunc` 
in our example code:

```go
	func handler(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
	}

	func main() {
		http.HandleFunc("/", handler)
		...
	}
```

We can see that we satisfy the requirements, as we pass `http.HandleFunc` a 
string, and a function which takes both a `ResponseWriter` and a `*Request` as 
arguments.

So, what *is* ServeMux? From [the docs](golang.org/pkg/net/http/#ServeMux):

> ServeMux is an HTTP request multiplexer. It matches the URL of each incoming 
> request against a list of registered patterns and calls the handler for the
> pattern that most closely matches the URL.
> 
> Patterns name fixed, rooted paths, 
>like "/favicon.ico", or rooted subtrees, like "/images/" (note the trailing 
>slash). Longer patterns take precedence over shorter ones, so that if there 
>are handlers registered for both "/images/" and "/images/thumbnails/", the 
>latter handler will be called for paths beginning "/images/thumbnails/" and 
>the former will receive requests for any other paths in the "/images/" subtree.
>  
> Note that since a pattern ending in a slash names a rooted subtree, the 
>pattern "/" matches all paths not matched by other registered patterns, 
>not just the URL with Path == "/".
>  
> Patterns may optionally begin with a host name, restricting matches to URLs
> on that host only. Host-specific patterns take precedence over general 
>patterns, so that a handler might register for the two patterns "/codesearch"
> and "codesearch.google.com/" without also taking over requests for 
>"http://www.google.com/".
>  
> ServeMux also takes care of sanitizing the URL request path, redirecting any 
>request containing . or .. elements to an equivalent .- and ..-free URL

## ListenAndServe
---

Hopefully some of the 'magic' behind the ubiquitous Go/Golang net/http example
 has been demystified. Or you're thoroughly confused, and should send me 
 feedback.

Assuming the former, let's look under the curtain of our final function call, 
`http.ListenAndServe(":8080", nil)`. Once again, from [the source](
http://golang.org/src/pkg/net/http/server.go?s=52405:52460#L1766): 

```go
    // Commentary:
	// ListenAndServe listens on the TCP network address addr
	// and then calls Serve with handler to handle requests
	// on incoming connections.  Handler is typically nil,
	// in which case the DefaultServeMux is used.
	func ListenAndServe(addr string, handler Handler) error {
		server := &Server{Addr: addr, Handler: handler}
		return server.ListenAndServe()
	}
    // End Commentary
```

We get a new `Server` struct as `server`, and then call the 
`http.ListenAndServe` method on that server, which if [we follow the source](
http://golang.org/src/pkg/net/http/server.go?s=49983:50024#L1669) will lead us 
to calling the `Serve` method on our `Server` struct, passing it a 
`tcpKeepAliveListener`, which 

> sets TCP keep-alive timeouts on accepted
> connections. It's used by ListenAndServe and ListenAndServeTLS so
> dead TCP connections (e.g. closing laptop mid-download) eventually
> go away.

For continuity, the source of the `Server.ListenAndServe` method, where we
 can see us calling `Server.Serve` is:

```go
    // Commentary:
	// ListenAndServe listens on the TCP network address srv.Addr and then
	// calls Serve to handle requests on incoming connections.  If
	// srv.Addr is blank, ":http" is used.
	func (srv *Server) ListenAndServe() error {
		addr := srv.Addr
		if addr == "" {
			addr = ":http"
		}
		ln, err := net.Listen("tcp", addr)
		if err != nil {
			return err
		}
		return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
	}
    // End Commentary
```

Returning from `Server.ListenAndServe` is the result of the `Server.Serve` 
call. The final call to our `server`'s `Serve` method is defined as: 

```go
    // Commentary:
	// Serve accepts incoming connections on the Listener l, creating a
	// new service goroutine for each.  The service goroutines read requests and
	// then call srv.Handler to reply to them.
	func (srv *Server) Serve(l net.Listener) error {
		defer l.Close()
        // Commentary:
        // how long to sleep on accept failure
		var tempDelay time.Duration
		for {
			rw, e := l.Accept()
			if e != nil {
				if ne, ok := e.(net.Error); ok && ne.Temporary() {
					if tempDelay == 0 {
						tempDelay = 5 * time.Millisecond
					} else {
						tempDelay *= 2
					}
					if max := 1 * time.Second; tempDelay > max {
						tempDelay = max
					}
					srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
					time.Sleep(tempDelay)
					continue
				}
				return e
			}
			tempDelay = 0
			c, err := srv.newConn(rw)
			if err != nil {
				continue
			}
            // Commentary:
            // before Serve can return
			c.setState(c.rwc, StateNew)
			go c.serve()
		}
	}
    // End Commentary
```

We can see that `Server.Serve` is busy waiting on calls to `l.Accept` (the 
`tcpKeepAliveListener` that was passed to Serve), and if we ignore the 
`net.Error` handling made up of nested if statements, it's clear that we create
 a `newConn` (new connection) with what we received from `l.Accept`, and 
 then set some state on that connection.

For our discussion, we'll hand-wave that stuff, and go straight to the final 
piece of magic: 

```go
	...
	go c.Serve()
	...
```

Now, it is plain that no matter what happened with the hand-wavy stuff, 
we've called the `Serve` method on our new accepted connection with its own 
Goroutine. This, along with other Goroutines running (like our current 
`main()` Goroutine), will be multiplexed in the Go Runtime. 

#### That's it
---

In under 300 lines of markdown, we've uncovered the magic behind the 
fantastically simple net/http server. Granted, I skipped some source, 
comments, and tons of options (TLS anyone?); but, you now have the tools and 
links to source to learn more. 

Also, this isn't a stab at Rails, since I'm an active and avid user of it 
still, but you just can't delve down into its innards as you can with Go. 
Sure, you might argue that Webrick, Puma, Unicorn, etc. aren't part of Rails, 
that's beside the point.

If you're interested in further demystifying what's going on, I'd recommend 
trying to understand the concept of the Go Runtime. Check out the 
[Analysis of the Go runtime scheduler](
http://www.cs.columbia.edu/~aho/cs6998/reports/12-12-11_DeshpandeSponslerWeiss_GO.pdf)
 by Neil Deshpande, Erica Sponsler, and Nathaniel Weiss @ Columbia University - 
 which, although somewhat dated, is still very relevant. The only note I 
 would make while you read that, though, is that the changes discussed 
 (proposed by Dmitry Vyukov) are partially in the current Go Runtime Scheduler.

Last note: If you liked this, you should check out my 
[Common Mistakes Made with Golang, and Resources to Learn From](
/blog/2014-07-05-common-mistakes-with-go-lang) post.

