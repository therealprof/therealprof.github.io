+++

date = 2020-08-24
title = "Revamping Rust Embedded Error Handling － a case for embedded-error"
slug = "Revamping-Rust-Embedded-Error-Handling"

+++

In this blog post we'll have a look at error handling in Rust embedded
scenarios. How does this work at the moment, what are the user expectations
and what can we do to make error handling future proof?

<!-- more -->

# Introduction

There have been many great articles, discussions and talks about error handling
Rust in the last few months. The ecosystem has made great progress on that
front ever since the `std::error::Error` trait became powerful with the
introduction of the `source` method in 1.30.0 and the `#[non_exhaustive]`
attribute in 1.40.0. I'm really amazed about the fantastic work and would
highly recommend to have a look at e.g. [anyhow] and [thiserror] to see some
state of the art possibilities, if you're not familiar with those achievements.

**BUT** we're talking about embedded here and there're some notable differences
to what is possible in the `std` world mentioned above. Some of these are:
* we're restricted to `no_std` in embedded, i.e. `libcore` instead of `libstd` and there's no `Error` trait in `libcore`
* there's no allocator, so we cannot use `Box` or any other types requiring a heap
* we don't usually have an output medium, so all errors have to be machine read-/manageable instead of human-friendly
* error kinds need to be common across all dependencies and composable, "surprise" errors can only be dealt with in the worst possible way
* some common conditions which are not really errors are also signalled via the `Err` case in the `Result`

Of course it needs also to be possible to easily bridge the gap between
`no_std` and `std` since devices like a Raspberry Pi should also be able to use
embedded device drivers, so we should not stray to far away from known
territory.

# The current situation

In embedded we achieve composability by relying on the traits provided by the
[embedded-hal] crate. This provides an interface between HAL implementations
providing the access to the hardware peripherals on one side and drivers and/or
applications on the other side. In the current version most of the traits are
fallible, in the upcoming version 1.0 there will only be fallible methods, so
we will always have a `Result` returning us any errors or other conditions from
the hardware peripherals or external components hooked up to those peripherals.

However there's one slight snag: All error kinds are based on associated types
defined inside the traits. Take the `serial::Read` trait for example:
```
pub trait Read<Word> {
    /// Read error
    type Error;

    /// Reads a single word from the serial interface
    fn read(&mut self) -> nb::Result<Word, Self::Error>;
}
```

What this means is that the implementation defines its own error type and
asssigns it as an associated type in its implementation. So every user of this
trait needs to know exactly what the `Err` kind type is and be able to handle
the relevant specific cases of that particular type, which is... not ideal.

So within an application the `Result` handling is specific to the HAL impl it
is using. This might be okay in a few cases because you don't care about the
non-`Ok` case or you simply use wildcard handling for errors. Of course you
could also create a `cfg`-gated features mess to adapt to each possible
hardware abstraction your application is supposed to run on.

But what if you want to write a generic driver which only depends those traits
and does not even know which hardware it'll be running on and this particular
driver requires to handle specific conditions which can only be signalled using
the `Err` part? E.g. there're flash chips for the I2C bus which will return a
`NACK` condition when they're currently busy processing a previous request. The
driver would know what to do when that occurs but it cannot know about the
condition, because it's part of a custom type only known to the HAL
implementation. The only thing a driver can do is forward the `Err` to the
application which might either not be able to cope with it in the context of
the driver or run into the previously mentioned problem again.

# How to deal with the `Err` case in a generic way?

In order to address this problem with arbitrary unknown `Err` cases, I've
proposed and created a new crate called [embedded-error]. The concept behind
that is based on the very same principles used by [enum std::io::ErrorKind],
which are:
* provide an enumeration of simple and fixed error kinds
* use the `#[non_exhaustive]` attribute to allow for non-breaking additions of error kinds at any point in time

In order to make it useful for embedded uses I introduced a few additional ideas:
* each category of peripheral gets they own dedicated set of error kinds
* each enumeration also contains a nested error kind `Impl` which is shared between different categories and used to signal implementation and/or environment specific problems

This allows us to have dependable `Err` types which everyone knows and which
are able to convey an exact message to the user of an `embedded-hal`
implementation or driver and thus to handle errors efficiently and idependently
of the target platform while being easily extendable at any point in the future
whenever the need arises.

# Future plans for `embedded-error`

At the moment it is mostly a proof of concept and does by far not cover all
possible peripherals and error kinds. But that's okay because as mentioned we
can add more types at any time and I'd highly encourage everyone to PR their
favourite error kinds to the [embedded-error repository].

I will also convert various HAL impls (like [stm32f0xx-hal] and
[stm32f4xx-hal], unless someone beats me to it 😉) to those error kinds because
most of those custom error kinds are very rudimentatry. So at least from a
documentation and naming perspective such a change makes a lot of sense even
though we still have the associated type complication in [embedded-hal] for
now.

Of course the final step would be general acceptance and use through
[embedded-hal] by getting rid of the associated type. But quite frankly I'm
very tired of the discussion in [Issue #229] (and in private) where the
current opinion is stuck in "this is nice but not good enough for [insert
hypothetical use case here], so let's rather do nothing at all" limbo.

Please feel free to let me know your thoughts, either personally via the
available channels or via issues in the mentioned repositories. Of course I'd
also love to PRs implementing or furthering any of the mentioned things.

So long and until the next post. 👋

[anyhow]: https://crates.io/crates/anyhow
[thiserror]: https://crates.io/crates/thiserror
[embedded-hal]: https://crates.io/crates/embedded-hal
[embedded-error]: https://crates.io/crates/embedded-error
[enum std::io::ErrorKind]: https://doc.rust-lang.org/std/io/enum.ErrorKind.html
[embedded-error repository]: https://github.com/therealprof/embedded-error/
[stm32f0xx-hal]: https://github.com/stm32-rs/stm32f0xx-hal/
[stm32f4xx-hal]: https://github.com/stm32-rs/stm32f4xx-hal/
[Issue #229]: https://github.com/rust-embedded/embedded-hal/issues/229
