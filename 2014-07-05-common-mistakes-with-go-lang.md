# Common Mistakes Made with Golang, and Resources to Learn From

## Background: My Experience with Golang Thus Far 

I've been keeping tabs on Go for a while now, so I was excited when a couple of 
months ago I had the need to serve millions of hits from a small grouping of 
dedicated servers, which needed to:

1. Cache requests for a pre-determined TTL
2. Reach out to my database (Cassandra with a cluster in the same Datacenter)
3. Return a JSON marshalled response for any given endpoint

In ideally under 10ms. I architected a solution which relied heavily on Go:

## Caching

After mentioning that I was thinking of using Redis (being used in conjunction 
with [twemproxy](https://github.com/twitter/twemproxy),
[smitty](https://github.com/areina/smitty), and
 [Redis Sentinel](http://redis.io/topics/sentinel)) as read-only storage for
  the endpoint data, `#go-nuts` user **kaos\`** on `freenode` suggested that
  [I eat "mashed glass"](https://botbot.me/freenode/go-nuts/msg/14341610/).

With a more fruitful discussion with user **Tv\`**, I'd stumbled across - and 
ended up using - [Groupcache](https://github.com/golang/groupcache), which I'm
 using with a user-defined TTL.

I'll go over the great benefits of using Groupcache soon in another post!

## Web Serving

Initially, since I only wanted to get one endpoint up and running, I'd decided 
on simply using Go's incredible [net/http](http://golang.org/pkg/net/http/) 
core Package.

It's quick and provides all of the functionality a simple API would need.

## Data Access

Quickly and efficiently accessing the data from 
[Groupcache](https://github.com/golang/groupcache) was easy thanks to another 
core Go Package: [net/rpc](http://golang.org/pkg/net/rpc/). 

A [Remote Procedure Call](http://en.wikipedia.org/wiki/Remote_procedure_call) 
is an incredibly powerful technique for using functions made publicly available
 on another server's API. 
 
<p class="center">
  ![RPC Technique Overview](http://i.imgur.com/lYQgi0R.gif)
</p>

<p class="center">
  <small>source: [http://www.cs.cf.ac.uk/Dave/C/node33.html](http://www.cs.cf.ac.uk/Dave/C/node33.html)</small>
</p>

## Production

As I was working on and writing a number of other moving parts for this 
particular project, I'd asked a great colleague of mine, Kurt Froehlich, to put
 together a working Webserver and Data Access backend using the above 
 architecture. Overall, I'd say the process went smooth since I'd found a 
 [full example](https://github.com/capotej/groupcache-db-experiment) which
  (albeit in a little bit of an obfuscated manner) hits all three points needed. 

At a very high level, most of what was needed was achievable by replacing the
 [slowdb](
 https://github.com/capotej/groupcache-db-experiment/blob/master/slowdb/slowdb.go) 
 with [Cassandra](http://cassandra.apache.org/) by way of using the 
 [gocql Cassandra Client](https://github.com/gocql/gocql), and writing a few
  methods to retrieve and format incoming query parameters as we needed them.

Kurt was able to get this running and separated into different packages within
 a week or two with minimal prior knowledge of the language. I think he'd made 
 it about 80 pages into [Programming in Go](http://www.qtrac.eu/gobook.html) 
 when he'd started working on it. Definitely a testament to both Kurt and the 
 ease of using/learning Go.

I've since patched a few performance pitfalls in the code, rewritten the API to 
use the [mux Package](http://www.gorillatoolkit.org/pkg/mux) from the 
[Gorilla web toolkit](http://www.gorillatoolkit.org/), and expanded the API to a
 number of endpoints. But, the core of the original functionality is still much 
 the same, and it runs great.

Additionally, I'd written ~2k LOC in Go for data ingestion, retrieval, and 
reporting in different areas of this application. One of the first (and only 
publicly available) pieces I'd written is a 
[script named solr-fetch](
https://github.com/momer/solr-index-fetch/blob/master/solr-fetch.go) which 
fetches and downloads all the pieces of the latest version of a 
[Apache Solr](http://lucene.apache.org/solr/) index via the 
[Solr REST API](https://wiki.apache.org/solr/SchemaRESTAPI).

 So, Common Mistakes...

Recently, over in [/r/golang](http://www.reddit.com/r/golang), I've seen an 
influx of posts about issues people have when they use constructs that they 
might not fully understand or have not understood what the consequences of using
 them are, like: 

- ["go concurrency bug"s](http://utcc.utoronto.ca/~cks/space/blog/programming/GoRaceBug) or 
- ["the closure pitfall"](http://www.goinggo.net/2014/06/pitfalls-with-closures-in-go.html)

So I figured I'd go over a few resources on where to best fill this 
understanding gap if you want to bypass some of these issues. If you have any 
specific questions, shoot me an email at `mo [at] umkr.com`, and I'll see if I 
can either answer them or find some resources which do.

## (Non)Persistent Connections

One of the easiest pitfalls to stumble into when using any external resource is 
the latency and load associated with creating and closing new connections.

In the above project, the initial implementation created and closed an RPC 
connection on every request. As the number of requests scaled up, this quickly 
caused both the Data Access RPC Server *and* the Webserver to stumble over 
themselves, as we can see here:

![Stress testing via blitz.io]({{ site.url }}/assets/img/rpcPitfallBlitzErrors.png)

So, what's the solution? Well, if we look closely at the relevant areas of the 
[net/rpc Package source code](
http://golang.org/src/pkg/net/rpc/client.go?s=7883:7989#L282), we can see that a 
single connection is already built to handle many concurrent requests with the
 `rpc.Client.Go` method. 

I moved the RPC Client to a global variable, so that every Goroutine can make 
use of the same connection, and added 
[a very basic connection maintenance pattern](
https://gist.github.com/momer/ac20357abd331e23b8ea). This could be improved 
further with connection pools etc. In any case, we're able to see that built-in 
concurrency at work:

![Stress testing via blitz.io](/assets/img/rpcPitfallsBlitzResolved.png)

Performance differences for a stress test ranging from 0 - 1000 concurrent 
users:

- Average response time *decreased* from **<span class="red">110ms</span>** to
    **<span class="green">32ms</span>**
- Maximum response time *decreased* from **<span class="red">438ms</span>** to 
    **<span class="green">&lt; 33ms</span>**
- Total number of timeouts *decreased* from **<span class="red">5401</span>** to 
    **<span class="green">0</span>**
- Total number of errors *decreased* from **<span class="red">2</span>** to 
    **<span class="green">0</span>**
- Total number of hits processed *increased* from 
    **<span class="red">2713</span>** to **<span class="green">13,991</span>**

## Growing fat from leftovers

Goroutines are cheap. Their [minimum stack size is only 8kb](
http://golang.org/doc/go1.2#stack_size). This is why we can create thousands of 
them and not sweat it.

Unless.

Unless you're making use of functions that make use -- and return values from --
 slices. I'd run into this just a few weeks ago, as I had written a couple of 
 functions without bearing in mind the inner workings of the `Slice`.

As described [on this Go Blog post about slices](
http://blog.golang.org/go-slices-usage-and-internals#TOC_6.), reslicing an 
array doesn't make a copy of the original, underlying, array for a given slice 
- it just continues to use the original array until nothing references it.

If you find yourself creating a large slice within a function used by a 
Goroutine, but only returning a single item from the slice 
(ie: `return mySlice[1:4]`), bear in mind that the entire array that your 
original slice references is still in memory!

Do this `n` number of times, and you could quickly find yourself depleting the 
memory available to new goroutine stacks. Try to create one more under these 
conditions, and your program will start spitting up errors.

## Loops, Closures, and Local Variables 

The Go documentation has a [great article about this common mistake in 
particular](https://code.google.com/p/go-wiki/wiki/CommonMistakes), so I won't 
reproduce the entire explanation here.

The premise is that when iterating over a loop, if you fire off a goroutine for
 each item in the loop and pass it a reference to the variable which is 
 changing in the loop, each goroutine will be referencing the **same** variable.

**<span class="red">Mistake</span>**

```go
// Source: https://code.google.com/p/go-wiki/wiki/CommonMistakes
for val := range values {
    go func() {
        fmt.Println(val)
    }()
}
```

Directly from the referenced article, "By adding val as a parameter to the 
closure, val is evaluated at each iteration and placed on the stack for the 
goroutine, so each slice element is available to the goroutine when it is 
eventually executed." We can see that in action here:

**<span class="green">Corrected</span>**
```go
// Source: https://code.google.com/p/go-wiki/wiki/CommonMistakes
for val := range values {
    go func(val interface{}) {
        fmt.Println(val)
    }(val)
}
```

## Concurrency, Channels, Completion

At the beginning of the article I'd linked to a blog post about a "go 
concurrency bug" that a user had "inflicted upon [himself]."

My major gripe with this article was that using [Channels](
http://golang.org/ref/spec#Channel_types) and [Goroutines](
http://golang.org/ref/spec#Go_statements) can be very easy and enjoyable if you
 understand how they work. The solution presented in the article does *work*, 
 but it doesn't address a very basic principle: Goroutines run on their own; 
 unless you explicitly wait for them to finish, your main thread will continue
  executing as normal.

Take that thought and keep it in the back of your mind whenever you use 
Channels and Goroutines.

There are many patterns to productively work with - and ensure the 
completion of work from - Channels and Goroutines, or just Goroutines on their 
own.

Many are described in this [Go blog post about Pipelines](
http://blog.golang.org/pipelines) and this 
[Google I/O 2013 presentation about Advanced Go Concurrency Patterns](
http://blog.golang.org/advanced-go-concurrency-patterns).

Dealing with race conditions doesn't have to be painful, either. Go's 1.1 
release included the [Go Race Detector](
http://golang.org/doc/articles/race_detector.html), also described in the 
[Go Blog Post: Introducing the Race Detector](
http://blog.golang.org/race-detector). 
