+++

date = 2020-09-07
title = "Rust Roadmap 2021: Adding a useful dev mode and making it the default"
slug = "roadmap-2021-usable-dev-mode"

+++

In this blog entry following the **[Call for Rust 2021 Roadmap items]**, I
shall layout my vision for giving Rust the ability to create debuggable
binaries which will also fit in the flash of a microcontroller.

<!-- more -->

# Motivation

This request has multiple motivations but following the **[Call for Rust 2021
Roadmap items]** rules I'm notating them in the agile user story way:

* As an embedded developer I would like to be able to create debuggable applications
* As an embedded developer I would also like to be able to create applications
  which fit into the very limited flash available to microcontrollers

So let me give you a bit more background about the status quo and what I mean
by those two stories before giving you my ideas how to address them.

## Debuggable applications which fit into flash

The "problem" with Rust applications in general is that once even just the
bare-minimum optimisations are enabled, the compiler will completely
restructure the whole application in a way that makes debugging hard or even
impossible.

On the other hand, if optimisations are completely disabled, `rustc` will
happily churn out code which is so highly inefficient and large that it either
won't fit in the available flash space or be so slow that crucial deadlines can
not be met (like USB handling) and the application will simply fail; quite
often those two issues also rear their ugly heads at the same time. It also
introduces `panic`ing branches into code which is not even supposed to be
anything more than a compiler hint, cf. [Issue 68208].

Other languages do a lot better by creating more efficient code without any
optimisations but they also offer optimisation levels which don't or only
lightly effect the debugability of an application built with them.

In addition to the code generation problems it's also a big stumbling block and
source of surprise and frustration that `rustc` uses **dev** mode by default,
often creating unusable binaries or even just errors.

Imagine exploring **Rust** coming from **C** and seeing that the most basic
program suddenly takes up many kB of precious flash instead of a few hundred
Bytes when compiled with default flags (and including a fixed overhead of a few
hundred bytes):

```
   text	   data	    bss	    dec	    hex	filename
    532	      0	      0	    532	    214	miniblink.elf
```

vs.

```
   text    data     bss     dec     hex filename
   9976       0       4    9980    26fc blinky
```
# Implementation ideas

I think it should be possible to come up with a set of debugging friendly basic
optimisations applied at `opt-level=g` which operate similar to clang's or
gcc's `-Og`. Those are similar to `opt-level=1` but disable debugging harming
optimisations. Furthermore this level should be the default for `dev` mode to
make Rust in embedded scenarios useful by default!

Thanks for your attention and hope to blog to y'all soon!

[Call for Rust 2021 Roadmap items]: https://blog.rust-lang.org/2020/09/03/Planning-2021-Roadmap.html
[Issue 68208]: https://github.com/rust-lang/rust/issues/68208
