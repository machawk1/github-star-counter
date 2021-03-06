[![Build Status](https://travis-ci.org/Byron/github-star-counter.svg?branch=master)](https://travis-ci.org/Byron/github-star-counter)

This program is made just for trying async-await code in the current ecosystem.
It features the following capabilities:

 * do https requests
 * do multiple requests at a time, one per page
 * use async closures

The code was done synchronously first, and then moved to async with a surprisingly small amount of
changes.
It was interesting to see how the [`ascync` constructs](https://github.com/Byron/github-star-counter/blob/e3746b9182a28a9e9a9e8dd55cdb660f6b1b97df/src/lib.rs#L90)
allow to control parallelism precisely, to the point where I was able to design interdependent
futures to match the data dependency. That way, things run concurrently when they can run concurrently, 
which can be visualized neatly with a dependency graph.

The greatest difficulties were around getting https to work. Besides, it's clearly a learning process
to understand the implications of futures better. Constructs with `async` tend to _look_ synchronous,
but show their teeth with closures and ownership. Everything is solvable, just own everything, yet I think
more borrowing will be enabled once `async` lands on _stable_.

Something I absolutely agree with is the [statements in the async book](https://rust-lang.github.io/async-book/01_getting_started/02_why_async.html)
which indicate that not everything needs to be async. Personally, I would probably start `sync`, and
wait for performance requirements to change before making the switch. However, threads I would avoid in _future_,
unless it truly is the simpler solution.

Something I look forward to is to see fully-async libraries emerge, for example, to interact with `git`,
which will probably perform better than existing libraries. _Using_ `async` libraries already is a breeze!

When thinking about the parallelism of this simple application it already becomes evident that one would want to control the amount of in-flight futures. Just imagine
the adverse effects of making too may concurrent connections to the same host, or the limits of resources imposed by the operating system itself. One would want to 
have executors who are aware of what kind of future they are running, and have them limit the amount of concurrently running ones.

With `async`, Rust can be even more so change the game!

### Installation

```bash
cargo install github-star-counter
```

### Running and usage

```bash
count-github-stars Byron
```

```bash
count-github-stars --help
```

A more complete example, showing how massive the speedups can be. However, please keep in mind that this can also be contention, e.g. there
are simply too many concurrent requests which are much slower together than they would be individually.
```
2019-08-15 08:47:49,553 INFO  [github_star_counter] Total bytes received in body: 11.5 MB
2019-08-15 08:47:49,553 INFO  [github_star_counter] Total time spent in network requests: 366.84s
2019-08-15 08:47:49,553 INFO  [github_star_counter] Wallclock time for future processing: 22.62s
2019-08-15 08:47:49,553 INFO  [github_star_counter] Speedup due to networking concurrency: 16.22x
Total: 214379
Total for seanmonstar: 3818
Total for orgs: 210561

mozilla/pdf.js         ★  27611
mozilla/DeepSpeech     ★  10899
mozilla/BrowserQuest   ★  8249
mozilla/send           ★  8165
mozilla/togetherjs     ★  6393
mozilla/nunjucks       ★  6207
tokio-rs/tokio         ★  5598
linkerd/linkerd        ★  5042
hyperium/hyper         ★  5031
linkerd/linkerd2       ★  4342
➜
```

### Development

```bash
git clone https://github.com/Byron/github-star-counter
cd github-star-counter
# Print all available targets 
make
```

All other interactions can be done via `cargo`.

### Difficulties on the way...

Please note that at the time of writing, 2019-08-13, the ecosystem wasn't ready.
Search the code for `TODO` to learn about workarounds/issues still present.

* `async || {}` _(without move)_ is not yet ready, and needs to be move. This comes with the additional limitation that references can't be passed as argument, everything it sees must be owned.
* `reqwest` with await support is absolutely needed. The low-level hyper based client we are using right now will start failing once github gzips its payload. For now I pin a working hyper version, which hopefully keeps working with Tokio.
* Pinning of git repositories is not as easy as I had hoped - I ended up creating my own forks which are set to the correct version. However, it should also work with the `foo = { git = "https://github.com/foo/foo", rev = "hash" }` syntax. Maybe my ignorance though.
* I would be interested in something like `collect::Result<Vec<Value>, Error>` for `Vec<Future<Output = Result<Value, Error>>>`. `join_all` won't abort on first error, but I think it should be possible to implement such functionality based on it.
* Defining a closure with `let mut closure: impl FnMut(User, usize) -> impl Future<Output = Value>` doesn't seem to work. The closure return type must be a type parameter.

### Changelog

For the parallelism diagrams, a data point prefixed with `*` signals that multiple data is handled at the same time.

#### v1.1.0 - Support for 'tera' templates

Thanks to the generous contribution of @mre there now is support for rendering to custom tera
templates. [Look here](https://endler.dev/about/) for an example.

#### v1.0.6 - Assurance of correctness

Github can silently adjust the page size, e.g. one asks for 1000 items per page and generates queries accordingly, but it will respond only with 100.
Now we check and abort with a suggested page size, if the given one was not correct. The current page size seems to be limited to 100.

#### v1.0.5 - Better performance metrics

#### v1.0.4 - Even better progress - less is more

Just show the aggregated result

#### v1.0.3 - Better progress messages

Even though the header is parsed and received relatively quickly, the body is read afterwards which takes additional time.
This will now be logged as well.

#### v1.0.2 - Even more parallel query of user's repositories

Parallelism looks like this:
```
 user-info+---->orgs-info+---->*(user-of-orgs+---->*repo-info-page)
          |
          |
          +---->*repo-info-page
```
Now it's as parallel as it can be, based on the data dependency. This is real nice actually!

#### v1.0.1 - More parallel query of user's repositories

Parallelism looks like this:
```
user-info+---->orgs-info+-+-->*(user-of-orgs+---->*repo-info-page)
         |                |                       ^
         |          wait  |                       |
         +----------------+-----------------------^
```
We don't wait for fetching org user info, but still wait for orgs information before anything makes progress.
Fetching repo information for the main user waits longer than needed.

#### v1.0.0 - Initial Release

Parallelism looks like this:
```
user-info+---->orgs-info+--->*(user-of-orgs-and-main-user+---->*repo-info-page)
```

### Reference

[This gist](https://gist.github.com/yyx990803/7745157) got me interested in writing a Rust version of it.
