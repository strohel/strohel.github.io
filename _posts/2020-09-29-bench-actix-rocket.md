---
title: "Benchmarking vol. 2: Pitting Actix against Rocket v0.4 and v0.5-dev"
tags:
- Google Cloud Platform
#- Elasticsearch
- Rust
image: /images/2020-09-29-bench-actix-rocket/cover.png
#reddit: TODO /r/rust/comments/TODO/TODO/
#hn: TODO /item?id=TODO
---

![illustration](/images/2020-09-29-bench-actix-rocket/cover.png)
I present a Rust-specific sequel to my [previous benchmark of 2 Kotlin and a Rust microservice](/bench-rust-kotlin-microservices/)
--- it's hard to resist one's own curiosity and popular demand, especially when you've been
[nerd](https://www.reddit.com/r/rust/comments/is9onc/what_i_learnt_from_benchmarking_http4k_ktor/g57n93y/?utm_source=share&utm_medium=web2x&context=3)-[sniped](https://xkcd.com/356/).
Let's stress-test the two prominent web frameworks: Actix Web and Rocket.
In addition to stable threads & blocking calls Rocket v0.4,
I have included a development snapshot of in-the-works Rocket v0.5,
which is async and [no longer requires nightly Rust](https://github.com/SergioBenitez/Rocket/commit/56a61726).

*Impatient? [Jump to the results](#results).*

## Preamble

I'll take advantage of the previous article to fully describe
aspects that apply equally well to this round:
- [How this is different than TechEmpower benchmarks](/bench-rust-kotlin-microservices/#on-techempower-framework-benchmarks).
  *TL;DR: we want to capture finer nuances and test idiomatic implementations with error reporting,
  logging, etc. --- rather than highly optimised ones.*
- [The microservice we've benchmarking](/bench-rust-kotlin-microservices/#the-service).
  *TL;DR: a simple endpoint that does one call to Elasticsearch server.*
- [The testing methodology](/bench-rust-kotlin-microservices/#recipe).
  *TL;DR: repeated runs of a Python script that spins microservice Docker container
  and exposes it to an increasing number of concurrent connections using [wrk](https://github.com/wg/wrk).*
- [Runtime Environment](/bench-rust-kotlin-microservices/#runtime-environment).
  *TL;DR: we limit the microservice to 1.5 CPU cores and 512 MiB memory using Docker.*
- [Hardware](/bench-rust-kotlin-microservices/#hardware).
  *TL;DR: Google Cloud Platform VM with 4-core AMD Epyc Rome CPU for the microservice + 12-core machine for the Elasticsearch server.*

What changed is Rust version, we have to use **nightly** because of Rocket v0.4.
More specifically, all implementations are compiled using
`rustc 1.47.0-nightly (2d8a3b918 2020-08-26)`.[^nightly]

[^nightly]: On my Gentoo development workstation,
    I use the latest official Rust release (currently 1.46.0) but [compiled as nightly](https://packages.gentoo.org/useflags/nightly).
    This combines the rigidity of a well-tested release with the ability to use nightly features.
    I wish this existed in form of a Docker image.

## Actix v3.0

Code as benchmarked: [locations-rs tag `actix-v30`](https://github.com/strohel/locations-rs/tree/actix-v30).
Uses **Actix Web 3.0.2**.

Just some small things have changed
since the [version described in the last post](/bench-rust-kotlin-microservices/#the-story-of-locations-rs):
- We now use version 3.0, but note that [its performance matches v2.0 in our case](https://storage.googleapis.com/strohel-pub/bench-actix-versions/bench-results.html).
- OpenAPI (Swagger) support is reintroduced as I was able to [make Paperclip support v3.0, too](https://github.com/wafflespeanut/paperclip/pull/218).

## Rocket v0.4

Code as benchmarked: [locations-rs-rocket tag `rocket-v04`](https://github.com/strohel/locations-rs-rocket/tree/rocket-v04).
Uses stable **Rocket 0.4.5**.

Porting from Actix to Rocket v0.4 was a matter of
[one +175 -150 lines commit](https://github.com/strohel/locations-rs-rocket/commit/f37de2fe).[^lines]
It looks a bit scary but was mostly mechanical:
converting Actix type to Rocketry ones, and then fixing all compiler errors ---
I love how `rustc` essentially works as your to-do list.
There was only one major hurdle:

[^lines]: Not counting indentation changes and auto-generated Cargo.lock.

### Calling Async Functions from Blocking Handlers

Rocket v0.4 handlers are classic blocking (sync) functions, but Reqwest-based
[elasticsearch-rs](https://github.com/elastic/elasticsearch-rs) only provides an async API.
Whoops.

The general advice is to propagate the asynchronicity up in caller stack
instead of trying to call async functions from sync code.
But what if we really want to? These are our options:

1. `global-rt`: launch a global [threaded Tokio runtime](https://docs.rs/tokio/0.2/tokio/runtime/index.html#threaded-scheduler)
   alongside Rocket workers.
   Then call runtime's [Handle::block_on()](https://docs.rs/tokio/0.2/tokio/runtime/struct.Handle.html#method.block_on)
   in endpoint handlers to delegate the work to the global async runtime,
   pausing the Rocket worker until the future resolves.
2. `per-worker-rt`: create a [basic single-threaded Tokio runtime](https://docs.rs/tokio/0.2/tokio/runtime/index.html#basic-scheduler)
   *per each Rocket worker*.[^per-worker]
   In endpoint handlers then call [Runtime::block_on()](https://docs.rs/tokio/0.2/tokio/runtime/struct.Runtime.html#method.block_on),
   which here has different semantics (!) than the `Handle::block_on()` above:
   the future, and any other spawned async tasks, actually run within this Rocket worker thread.[^worker-run]
   This comes with a caveat.
   Reqwest::client() seems to *attach* itself to the async runtime it is first used in.
   I had to make Elasticsearch client also local to each Rocket worker.
   Otherwise, I got deadlocks or [problems described in hyper issue #2112](https://github.com/hyperium/hyper/issues/2112).
3. `per-req-rt`: launch a fresh basic Tokio runtime per each request.
   It feels wrong, and it is wrong. I've tried and benchmarked this so that we know *how much* wrong.
4. Patch elasticsearch-rs to provide a blocking API --- by employing Request's
   [optional blocking API](https://docs.rs/reqwest/0.10/reqwest/blocking/index.html).
   That would be futile, and essentially a sophisticated variant of 2.,
   given [the implementation details of the blocking client](https://github.com/seanmonstar/reqwest/blob/v0.10.8/src/blocking/client.rs#L715-L783).

[^per-worker]: I haven't found a way to attach user data to each Rocket worker thread,
    so the solution uses [stdlib's thread_local! macro](https://doc.rust-lang.org/std/macro.thread_local.html).

[^worker-run]: I.e. any possible spawned timers don't advance
    when a given thread is not processing any request.
    We don't mind.

[**Here are the results of benchmarking the first three approaches.**](https://storage.googleapis.com/strohel-pub/rocket-async-approach/bench-results.html)
Source code of each variant is available under the respective [tag in the locations-rs-rocket repository](https://github.com/strohel/locations-rs-rocket/tags).

You can almost hear the server crying as it tries to cope with the inefficiency of `per-req-rt`:
it is more than 7✕ less efficient than the best performing variant.

The other two more realistic variants are close to each other.
`per-worker-rt` has a slight edge in peak performance and a clear edge in efficiency,
especially for low connection counts.
It is therefore proclaimed a winner of this qualification round
and represents Rocket v0.4 in later benchmarks.

It is not a surprise that highly-optimised Actix uses a similar approach:
independent single-threaded Tokio runtimes per each worker
instead of a global work-stealing threaded one.

### Keep-Alive Connections

[Keep-Alive (persistent) connections](https://developer.mozilla.org/en-US/docs/Web/HTTP/Connection_management_in_HTTP_1.x#Persistent_connections)
save latency and resources when a client makes more than one request to a given server,
especially when the connection is secured by TLS.
Unfortunately, Rocket v0.4 is not their friend.

**First**, to the best of my knowledge,
[persistent connections don't work at all in Rocket v0.4](https://github.com/SergioBenitez/Rocket/issues/580#issuecomment-698286249)
--- the server closes the connection before reading a second request.
I've [traced the problem down to a bug in BufReader in old hyper 0.10.x and submitted a fix](https://github.com/hyperium/hyper/pull/2288/commits/3109103).
In the 0.11.x branch, the same bug was [fixed a long ago](https://github.com/hyperium/hyper/commit/d35992d0198d733c251e133ecc35f2bca8540d96#diff-078f367374debbc894f7cc1c4084e467R64-R66) and released with 0.11.0 in June 2017.
My pull request was [closed without merging](https://github.com/hyperium/hyper/pull/2288#issuecomment-698492487),
as the maintainers were (understandably) not keen on releasing a new version of a legacy branch that was superseded 3 years ago.
In other words, *Rocket v0.4 depends on unmaintained hyper* for its HTTP handling.

**Second,** even if the bug in hyper is patched,
keep-alive connections in hyper 0.10.x are [implemented naïvely](https://github.com/hyperium/hyper/blob/0.10.x/src/server/mod.rs#L282-L351):
the worker thread is kept busy waiting for the client on the persistent connection,
unable to process other requests.
It is therefore easy to (accidentally) trigger denial-of-service
by opening more persistent connections than available workers,
even with the default keep-alive timeout of 5 s.[^browsers]
Note that the first problem prevents this second problem from happening. ;-)

[^browsers]: Modern web browsers may open multiple, up to 6, concurrent connections to a single host.

Both issues were long resolved in more modern hyper 0.11+,
and therefore in Rocket v0.5-dev which I happen to benchmark too.

If you run Rocket v0.4 in production,
I recommend you to turn off persistent connections in Rocket config (set `keep-alive` to `0`)
--- while most clients retry the failed second request on a persistent connection gracefully,
at least some versions of Python `requests` and `urllib3` were raising exceptions instead.
If you care about latency, I suggest you put an HTTP load-balancer in front of Rocket v0.4 server
to reintroduce persistent connections at least to the client <-> load-balancer hop.

[**The benchmarks show that disabling keep-alive causes only a mild performance hit in our case.**](https://storage.googleapis.com/strohel-pub/rocket-keep-alive/bench-results.html)

The red `oneshot-*` line is Rocket with keep-alive disabled and 16 workers,
while other `persistent-*` lines represent Rocket with patched hyper, enabled keep-alive, and 16, 32 and 64 workers.
It can be seen that disabling keep-alive hurts latency, but not necessarily throughput
--- if the number of connections can be increased.
Note that the effect will be more pronounced in reality,
real network latencies are much more significant than that of our loopback interface.

Unfortunately, `wrk` does not indicate that some of its concurrent connections have failed to connect to the server at all.
But another demonstration of the denial-of-service behaviour is present when keep-alive is enabled:
Notice how the latencies of the 16-, 32-, and 64-worker instances of Rocket *cut-off* at
16, 32, respectively 64 concurrent connections.
When such saturation happens,
it is indeed impossible to make a new connection using e.g. `curl` to the Rocket instance.

Because of these 2 problems, Rocket v0.4 has keep-alive disabled in all other benchmarks.

### Tuning The Number of Workers

If you want to squeeze the highest possible efficiency from a Rocket v0.4 instance,
you should tweak the number of its worker threads.
The optimal count will depend mainly on the number of available CPU cores,
and the ratio of time your endpoints spend CPU-crunching and waiting for I/O.

[**Here I have benchmarked worker counts from 8 to 256.**](https://storage.googleapis.com/strohel-pub/rocket-worker-count/bench-results.html)

Instance with 16 workers is the most efficient, although the differences are small.
Most notable variance is, as expected, in memory consumption.
Having in mind that 1.5 CPUs is available the microservice, we arrive at around 10 workers per core.
Rocket's default is more conservative 2 workers per core.

## Rocket v0.5-dev

***Big fat warning**: Rocket v0.5 is still under development.
Version tested in this post is its [1369dc4 commit](https://github.com/SergioBenitez/Rocket/commit/1369dc4).
A lot of things may change in the final release, including all observations here.
You can track Rocket v0.5 progress in its [async migration tracking issue](https://github.com/SergioBenitez/Rocket/issues/1065)
and related GitHub milestone.*

Code as benchmarked: [locations-rs-rocket tag `rocket-v05-dev`](https://github.com/strohel/locations-rs-rocket/tree/rocket-v05-dev).

Porting from async Actix to async Rocket v0.5-dev was even easier than to Rocket v0.4.
[Here is the 147 insertions, 140 deletions commit that did the job](https://github.com/strohel/locations-rs-rocket/commit/879cd88).[^lines]

Compared to v0.4, Rocket v0.5-dev is *boring*, in the best possible sense of the word.
Persistent connections are without problems.
There is no need to fiddle with the number of workers.
I attribute this to porting to up-to-date hyper 0.13.x and async Rust ecosystem.

## Results

All graphs below are interactive and infinitely scalable SVGs --- zoom in if necessary.

<embed type="image/svg+xml" src="/images/2020-09-29-bench-actix-rocket/startup_time.svg" />

The startup time is measured from the moment Docker finishes setting up the container
to the moment when we receive a valid HTTP response to `GET /`.

No real difference, single-digit milliseconds correspond to the required round-trip to Elasticsearch server.

<embed type="image/svg+xml" src="/images/2020-09-29-bench-actix-rocket/errors_vs_connections.svg" />

Error response ratio in percents.

Nicely flat for all frameworks up to extreme 1024 and 2048 connections.
Actix manages to be slightly better there with ~1% of errors, compared to ~1.5% of both Rocket versions.

<embed type="image/svg+xml" src="/images/2020-09-29-bench-actix-rocket/requests_vs_connections.svg" />

High-water mark *successful* requests per second as the number of concurrent connections grows, one of our main metrics.

Recurring readers will recognise the curve of Actix, which peaks at ~11,000 requests per second.  
Rocket v0.4 is initially, when throughput is bound by latency,
handicapped by the lack of keep-alive support discussed above,
but then manages to climb close to 7,000 req/s.  
Development snapshot of Rocket v0.5 already performs better than its stable predecessor.
Its CPU efficiency doesn't allow it to reach the throughput of Actix, though.

<embed type="image/svg+xml" src="/images/2020-09-29-bench-actix-rocket/latency_vs_connections_50.svg" />

<embed type="image/svg+xml" src="/images/2020-09-29-bench-actix-rocket/latency_vs_connections_90.svg" />

<embed type="image/svg+xml" src="/images/2020-09-29-bench-actix-rocket/latency_vs_connections_99.svg" />

50th, 90th and 99th percentile latencies plotted using a *logarithmic scale*.

Actix is generally the best performer also when it comes to latency.
We see interesting ranking shifts between Rocket v0.4 and v0.5-dev between 50th, 90th, and 99th percentile.

<embed type="image/svg+xml" src="/images/2020-09-29-bench-actix-rocket/max_mem_usage.svg" />

Here we measure high-water mark memory usage of the container from its start till the end of each
benchmark step, as reported by Docker (i.e. not just momentary memory usage).

Memory consumption of Actix is decent, only surpassing 100 MiB after 1024 concurrent connections.  
But it cannot that of Rocket v0.4, which stays de-facto *constant* at ~12 MiB throughout the stress-test.
It should be attributed to inherent back-pressure caused by the fixed number of worker threads
--- the "thread & blocking calls" approach shines at capping resource usage.

I think that some form of resource limiting could be applied also to async servers,
but the difference is that back-pressure needs to be deliberately injected there.

Rocket v0.5-dev's memory usage sky-rockets (pun intended),
2 of its runs are out-of-memory killed when reaching 512 MiB.
This is a development version, so don't get too worried about it.
It could be a simple bug or some development artefact,
let's check it later when 0.5.0 release approaches.

<embed type="image/svg+xml" src="/images/2020-09-29-bench-actix-rocket/max_mem_usage_per_requests.svg" />

The same metric as above, but divided by the number of successful requests per second;
which gives an unconventional unit of *megabyte-seconds per request*.

Actix shows it's usual "basin" shape that was not explained since the last post.
If you have a clue, you definitely should speak up now.

Constant memory of Rocket v0.4 gets diluted into the increasing number of served requests.

Rocket v0.5-dev per-request memory is not decreasing even in the 2--16 connection range
where its throughput grows exponentially
--- a sign that the said bug could be a memory leak?

<embed type="image/svg+xml" src="/images/2020-09-29-bench-actix-rocket/cpu.svg" />

Consumed CPU time as reported by Docker API.

Actix and Rocket v0.4 show surprisingly similar pattern here.
While Rocket v0.4 is not itself async,
our implementation spawns per-thread *basic* Tokio async runtimes, as Actix does.

I attribute higher initial CPU consumption of Rocket v0.5-dev to its use of the threaded work-stealing Tokio runtime.
When such runtime was used in one of the Rocket v0.4 variants, its initial CPU consumption was similarly higher.

<embed type="image/svg+xml" src="/images/2020-09-29-bench-actix-rocket/cpu_per_request.svg" />

Consumed CPU time divided by the number of successful requests per second, or *CPU efficiency*.

Here, Actix manages to be incredibly efficient in the range of small hundreds of concurrent connections
--- that's a recipe for achieving record peak throughputs.

Synchronous Rocket v0.4 demonstrates the most stable performance,
it is least affected by the number of connections.

<embed type="image/svg+xml" src="/images/2020-09-29-bench-actix-rocket/cpu_vs_requests.svg" />

Running total of successfully served requests (time-integrated throughput)
over the cumulative sum of used CPU time, or *bang for the buck* for short.
Going up means serving requests, while going right means consuming CPU.
Line slope corresponds to CPU efficiency.

Actix again shows that it is heavily optimised.
The two Rocket versions display interestingly similar efficiency.
Their difference is that v0.5-dev is able to serve more requests in given wall-clock time,
but it asks appropriately more CPU ticks for it.

## Conclusion

The main takeaway is probably that both prominent Rust web frameworks are fast enough that
you can stop caring about performance, and concentrate on other aspects. The endpoint handler
tested here is trivial: the more complex your handler is, the less framework overhead matters.

I made a [detour to play with calling async Rust functions from a sync code](#calling-async-functions-from-blocking-handlers),
which may find use well outside web service backend development.

I have also [pointed-out some quirks of keep-alive support in Rocket v0.4](#keep-alive-connections).
That should serve as a reminder to check whether your *transitive* dependencies are maintained/up-to-date.
`cargo outdated` can help here.
Old dependencies sometimes also bring duplicate packages into dependency tree,
`cargo tree -d` will show them.
And finally [lib.rs](https://lib.rs/) shows outdated dependencies in red, though not the transitive ones.

Thanks for reading, and as usual I would be glad for any comments, thoughts and remarks.
