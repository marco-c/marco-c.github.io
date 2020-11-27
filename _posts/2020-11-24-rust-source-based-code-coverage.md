---
title: How to collect Rust source-based code coverage
tags: [code-coverage]
---

**TL;DR**: For those of you who prefer an example to words, you can find a complete and simple one at [https://github.com/marco-c/rust-code-coverage-sample](https://github.com/marco-c/rust-code-coverage-sample).

Source-based code coverage was [recently introduced](https://github.com/rust-lang/rust/issues/34701#issuecomment-723178087) in Rust. It is more precise than the gcov-based coverage, with fewer workarounds needed. Its only drawback is that it makes the profiled program slower than with gcov-based coverage.

In this post, I will show you a simple example on how to set up source-based coverage on a Rust project, and how to generate a report using [grcov](https://github.com/mozilla/grcov) (in a readable format or in a JSON format which can be parsed to generate custom reports or upload results to Coveralls/Codecov).

### Install requirements

First of all, let's install grcov:
<pre style="background-color:black;color:white;">
cargo install grcov
</pre>

Second, let's install the llvm-tools Rust component (which grcov will use to parse coverage artifacts):
<pre style="background-color:black;color:white;">
rustup component add llvm-tools-preview
</pre>
At the time of writing, the component is called `llvm-tools-preview`. It might be renamed to `llvm-tools` soon.


### Build
Let's say we have a simple project, where our main.rs is:
{% highlight rust %}
use std::fmt::Debug;

#[derive(Debug)]
pub struct Ciao {
    pub saluto: String,
}

fn main() {
    let ciao = Ciao{ saluto: String::from("salve") };

    assert!(ciao.saluto == "salve");
}
{% endhighlight %}

In order to make Rust generate an instrumented binary, we need to use the `-Zinstrument-coverage` flag (Nightly only for now!):
<pre style="background-color:black;color:white;">
export RUSTFLAGS="-Zinstrument-coverage"
</pre>

Now, build with `clang build`.

The compiled instrumented binary will appear under target/debug/:
<pre style="background-color:black;color:white;">
.
├── Cargo.lock
├── Cargo.toml
├── src
│   └── main.rs
└── target
    └── debug
        └── rust-code-coverage-sample
</pre>

The instrumented binary contains information about the structure of the source file (functions, branches, basic blocks, and so on).

### Run
Now, the instrumented executable can be executed (`cargo run`, `cargo test`, or whatever). A new file with the extension 'profraw' will be generated. It contains the coverage counters associated with your binary file (how many times a line was executed, how many times a branch was taken, and so on). You can define your own name for the output file (might be necessary in some complex test scenarios like we have on grcov) using the `LLVM_PROFILE_FILE` environment variable.

<pre style="background-color:black;color:white;">
LLVM_PROFILE_FILE="your_name-%p-%m.profraw"
</pre>
%p (process ID) and %m (binary signature) are useful to make sure each process and each binary has its own file.


Your tree will now look like this:
<pre style="background-color:black;color:white;">
.
├── Cargo.lock
├── Cargo.toml
├── default.profraw
├── src
│   └── main.rs
└── target
    └── debug
        └── rust-code-coverage-sample
</pre>

At this point, we just need a way to parse the profraw file and the associated information from the binary.

### Parse with grcov
[grcov](https://github.com/mozilla/grcov) can be downloaded from GitHub ([from the Releases page](https://github.com/mozilla/grcov/releases)).

Simply execute grcov in the root of your repository, with the `--binary-path` option pointing to the directory containing your binaries (e.g. `./target/debug`). The `-t` option allows you to specify the output format:
- "html" for a HTML report;
- "lcov" for the LCOV format, which you can then translate to a HTML report using genhtml;
- "coveralls" for a JSON format compatible with Coveralls/Codecov;
- "coveralls+" for an extension of the former, with addition of function information.
There are [other formats](https://github.com/mozilla/grcov#alternative-reports) too.

Example:
<pre style="background-color:black;color:white;">
grcov . --binary-path PATH_TO_YOUR_BINARIES_DIRECTORY -s . -t html --branch --ignore-not-existing -o ./coverage/
</pre>

This is the output:
<iframe src="/assets/source_based_coverage/src/main.rs.html" width="100%" height="300">
  <p>Your browser does not support iframes.</p>
</iframe>

This would be the output with gcov-based coverage:
<iframe src="/assets/gcov_coverage/src/main.rs.html" width="100%" height="300">
  <p>Your browser does not support iframes.</p>
</iframe>


You can also run grcov outside of your repository, you just need to pass the path to the directory where the profraw files are and the directory where the source is (normally they are the same, but if you have a complex CI setup like we have at Mozilla, they might be totally separate):
<pre style="background-color:black;color:white;">
grcov PATHS_TO_PROFRAW_DIRECTORIES --binary-path PATH_TO_YOUR_BINARIES_DIRECTORY -s PATH_TO_YOUR_SOURCE_CODE -t html --branch --ignore-not-existing -o ./coverage/
</pre>

grcov has other options too, simply run it with no parameters to list them.

In the [grcov's docs](https://github.com/mozilla/grcov#grcov-with-travis), there are also examples on how to integrate code coverage with some CI services.
