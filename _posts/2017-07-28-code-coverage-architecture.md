---
title: Overview of the Code Coverage Architecture at Mozilla
---

Firefox is a huge project, consisting of around 20K source files and 3M lines of code (if you only consider the Linux part!), supporting officially four operating systems, and being written in multiple programming languages (C/C++/JavaScript/Rust). We have around 200 commits landing per day in the mozilla-central repository, with developers committing even more often to the try repository. Usually, code coverage analysis is performed on a single language on small/medium size projects. Therefore, collecting code coverage information for such a project is not an easy task.

I'm going to present an overview of the current state of the architecture for code coverage builds at Mozilla.

Tests in code coverage builds are slower than in normal builds, especially when we will start [disabling more compiler optimizations to get more precise results](https://bugzilla.mozilla.org/show_bug.cgi?id=1344165). Moreover, the amount of data generated is quite large, each report being around 20 MB. If we had one report for each test suite and each commit, we would have around ~100 MB x ~200 commits x ~20 test suites = ~400 GB per day.
This means we are, at least currently, only running a code coverage build per mozilla-central push (which usually contain around ~50 to ~100 commits), instead of per mozilla-inbound commit.

<figure>
  <img src="/assets/linux64-ccov.png" alt="View of linux64-ccov build and tests from Treeherder" />
  <figcaption><b>Figure 1</b>: A linux64-ccov build (B) with associated tests, from <a href="https://treeherder.mozilla.org/#/jobs?repo=mozilla-central&filter-searchStr=linux64-ccov&group_state=expanded">https://treeherder.mozilla.org/#/jobs?repo=mozilla-central&filter-searchStr=linux64-ccov&group_state=expanded</a>.</figcaption>
</figure>

Each test machine, e.g. *bc1* (first chunk of the [Mochitest](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/Mochitest) browser chrome) in Figure 1, generates gcno/gcda files, which are parsed directly on the test machine to generate a [LCOV](https://github.com/linux-test-project/lcov) report.

Because of the scale of Firefox, we could not rely on some existing tools like LCOV. Instead, we had to redevelop some tooling to make sure the whole process would scale. To achieve this goal, we developed [grcov](https://github.com/marco-c/grcov), an alternative to LCOV written in Rust (providing performance and parallelism), to parse the gcno/gcda files. With the standard LCOV, parsing the gcno/gcda files takes minutes as opposed to seconds with grcov (and, if you multiply that by the number of test machines we have, it becomes more than 24 hours vs around 5 minutes).

Let's take a look at the current architecture we have in place:

<figure>
  <img src="/assets/code_coverage_overall_architecture.svg" alt="Architecture view" />
  <figcaption><b>Figure 2</b>: A high-level view of the architecture.</figcaption>
</figure>

Both the **Pulse Listener** and the **Uploader Task** are part of the awesome **Mozilla Release Engineering Services** ([https://github.com/mozilla-releng/services](https://github.com/mozilla-releng/services)). The release management team has been contributing to this project to share code and efforts.

## Pulse Listener

We are running a [pulse](https://pulseguardian.mozilla.org/) listener process on Heroku which listens to the [taskGroupResolved](https://docs.taskcluster.net/reference/platform/taskcluster-queue/references/events#taskGroupResolved) message, sent by TaskCluster when a group of tasks finishes (either successfully or not). In our case, the group of tasks is the linux64-ccov build and its tests (note: you can now easily choose this build on trychooser, run your own coverage build and generate your report. See [this page](https://developer.mozilla.org/en-US/docs/Mozilla/Testing/Measuring_Code_Coverage_on_Firefox#Generate_Code_Coverage_report_from_a_try_build_(or_any_other_treeherder_build)) for instructions).

The listener, once it receives the "group resolved" notification for a linux64-ccov build and related tests, spawns an "uploader task".

The source code of the **Pulse Listener** can be found [here](https://github.com/mozilla-releng/services/tree/master/src/shipit_pulse_listener).

## Uploader Task

The main responsibility of the uploader task is aggregating the coverage reports from the test machines.

In order to do this, the task:
1. Clones mozilla-central;
2. Builds Firefox (using artifact builds for speed); this is currently needed in order to generate the mapping between the URLs of internal JavaScript components and modules (which use special protocols, such as [chrome://](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XUL/Tutorial/The_Chrome_URL) or [resource://](http://www.iana.org/assignments/uri-schemes/prov/resource)) to the corresponding files in the mozilla-central repository (e.g. *resource://gre/modules/Services.jsm* &rarr; *toolkit/modules/Services.jsm*);
3. Rewrites the LCOV files generated by the JavaScript engine for JavaScript code, using the mapping generated in step 2 and also resolving preprocessed files (yes, we do preprocess [some JavaScript source files](https://dxr.mozilla.org/mozilla-central/search?q=regexp%3APP_COMPONENTS%7CPP_JS_MODULES&redirect=false) with a C-style preprocessor);
4. Runs grcov again to aggregate the LCOV reports from the test machines into a single JSON report, which is then sent to [codecov.io](https://codecov.io/gh/marco-c/gecko-dev) and [coveralls.io](https://coveralls.io/github/marco-c/gecko-dev).

Both codecov.io and coveralls.io, in order to show source files with coverage overlay, take the contents of the files from GitHub. So, we can't directly use our Mercurial repository, but we have to rely on our Git mirror hosted on GitHub ([https://github.com/mozilla/gecko-dev](https://github.com/mozilla/gecko-dev)). In order to map the mercurial changeset hash associated with the coverage build to a Git hash, we use a [Mapper service](https://wiki.mozilla.org/ReleaseEngineering/Applications/Mapper).

Code coverage results on Firefox code can be seen on:
- [codecov.io](https://codecov.io/gh/marco-c/gecko-dev)
- [coveralls.io](https://coveralls.io/github/marco-c/gecko-dev)

The source code of the **Uploader Task** can be found [here](https://github.com/mozilla-releng/services/tree/master/src/shipit_code_coverage).

## Future Directions
### Reports per Test Suite and Scaling Issues
We are interested in collecting code coverage information per test suite. This is interesting for several reasons. First of all, we could suggest developers which suite they should run in order to cover the code they change with a patch. Moreover, we can evaluate the coverage of [web platform tests](https://developer.mozilla.org/en-US/docs/Mozilla/QA/web-platform-tests) and see how they fare against our built-in tests, with the objective to make web platform tests cover as much as possible.

Both codecov.io and coveralls.io support receiving multiple reports for a single build and showing the information both in separation and in aggregation ("flags" on codecov.io, "jobs" on coveralls.io). Unfortunately, both services are currently choking when we present them with too much data (our reports are huge, given that our project is huge, and if we send one per test suite instead of one per build... things blow).

### Coverage per Push
Understanding whether the code introduced by a set of patches is covered by tests or not is very valuable for risk assessment. As I said earlier, we are currently only collecting code coverage information for each mozilla-central push, which means around 50-100 commits (e.g. [https://hg.mozilla.org/mozilla-central/pushloghtml?changeset=07484bfdb96b](https://hg.mozilla.org/mozilla-central/pushloghtml?changeset=07484bfdb96b)), instead of for each mozilla-inbound push (often only one commit). This means we don't have coverage information for each set of patches pushed by developers.

Given that most mozilla-inbound pushes in the same mozilla-central push will not change the same lines in the same files, we believe we can infer the coverage information for intermediate commits from the coverage information of the last commit.

### Windows, macOS and Android Coverage
We are currently only collecting coverage information for Linux 64 bit. We are looking into [expanding it to Windows](https://bugzilla.mozilla.org/show_bug.cgi?id=1381163),macOS and Android. Help is appreciated!

### Support for Rust
Experimental support for gcov-style coverage collection [landed recently in Rust](https://github.com/rust-lang/rust/pull/42433). The feature needs to be ship in a stable release of Rust before we can use it; [this issue](https://github.com/rust-lang/rust/issues/42524) is tracking its stabilization.
