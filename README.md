This program is made just for trying async-await code in the current ecosystem.
It features the following capabilities:

 * do https requests
 * do multiple requests at a time, one per page
 * use async closures

The code was done synchronously first, and then moved to async with a surprisingly small amount of
changes.

The greatest difficulties were around getting https to work. Besides, it's clearly a learning process
to understand the implications of futures better. Constructs with `async` tend to _look_ synchronous,
but show their teeth closures and ownership. Everything is solvable, just own everything, yet I think
more borrowing will be enabled once `async` lands on _stable_.

Something I absolutely agree with is the [statements in the async book](https://rust-lang.github.io/async-book/01_getting_started/02_why_async.html)
which indicate that not everything needs to be async. Personally, I would probably start `sync`, and
wait for performance requirements to change before making the switch.

Using async code already is quite trivial.

Something I look forward to is to see fully-async libraries emerge, for example, to interact with `git`,
which will perform better than existing libraries written in other languages.

With `async`, Rust can be even more so change the game!

### Installation

```bash
cargo install --git https://github.com/Byron/github-star-counter
```

Currently this crate cannot be published to _crates.io_.

### Running and usage

```bash
count-github-stars Byron
```

```bash
count-github-stars --help
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

### Changes

#### v1.0.0 - Initial Release


### Reference

[This gist](https://gist.github.com/yyx990803/7745157) got me interested in writing a Rust version of it.
