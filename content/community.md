+++
title = "The rust-av Community"
description = "Get help, discuss problems, and join the fun"
+++

# About

[Rust-av](https://github.com/rust-av) aims to be a complete multimedia toolkit written in rust.

**Rust** is a quite promising language that aims to offer high execution speed while granting a number of warranties on the code behavior that you cannot have in C, C++, Java and so on.

Its **zero-cost abstraction** feature coupled with the fact that the compiler actively **prevents** you from committing a large class of mistakes related to **memory access** seems a perfect match to implement a multimedia toolkit that is **easy to use, fast** enough and **trustworthy**.

# Why something completely new?

Since rust code can be intermixed with C code, an evolutive approach of replacing little by little small components in a larger project is perfectly feasible, and it is what we are currently trying to do with [vlc](http://dev.unhandledexpression.com/slides/rustconf-2016/vlc/).

But rust is not just good to write some inner routines so they are fast and correct, its [trait](https://doc.rust-lang.org/book/second-edition/ch10-02-traits.html) system is also quite useful to have a more expressive API.

Most of the multimedia concepts are pretty simple at the high level (e.g frame is just a picture or some sound with some timestamp) with an excruciating amount of quirk and details that require your toolkit to make choices for you or make you face a large amount of complexity.

That leads to API that are either easy but quite inflexible (and opinionated) or API providing all the flexibility, but forcing the user to have to learn a lot of information in order to achieve what the simpler API would let you implement in an handful of lines of code.

# Projects

* matroska
* rust-av
* ffms2-rs
* rav1e
* speexdsp-rs
* opus-rs
* av-vorbis
* ave
* vpx-rs
* dav1d-rs
* flavors

and [much more](https://github.com/rust-av).

# Get Involved

Are you interested in getting involved with the code and participating in
our development?
Contact us on [IRC](https://web.libera.chat/#rust-av) or on the [Github Discussion](https://github.com/rust-av/rust-av/discussions).

# Contributors

We would like to thank to numerous volunteers around the globe for helping rust-av grow :).
