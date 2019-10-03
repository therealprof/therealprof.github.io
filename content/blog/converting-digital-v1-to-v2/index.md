+++

date = 2019-10-03
title = "Converting an Embedded HAL impl from digital::v1 to digital::v2 traits"
slug = "digital::v1_to_digital::v2"

+++

A while ago people had the idea to introduce fallible traits for GPIO pin to
allow the implementation of virtual pins and port expanders. In this blog I'm
demonstrating how to convert an existing implementation to the `v2` version of
the traits.

<!-- more -->

# Some background

[Embedded HAL](https://github.com/rust-embedded/embedded-hal) ([embedded-hal]
on crates.io) contains a number of traits specifying the interface to implement
and use peripherals usually found on embedded devices. The `digital` traits in
particular define functions/methods to be used to both read the electrical
state of an **input pin** as well as change the electrical state of an **output
pin**.

Now a **pin** might not be necessarily a thing which directly corresponds to
some local memory at the device the program itself is running on but might be
managed by a remote chip. The big difference is that usually with a local pin
there's really nothing that can go wrong when reading or changing state as long
as you can access the memory location related to it[1]. This all changes if the
pin is not under direct control, e.g. because you're using something like my
[embedded-bridge] or a GPIO exander chip like the
[PCF857x](https://crates.io/crates/pcf857x), in which case there are number of
reasons which a seemingly operation like reading the state might actually fail
and you want to convey that information to the user of the HAL implementation.

This is why people decided to introduce a new version of the `digital` traits
to convey errors along. The new traits were introduced in the version `0.2.3`
of `embedded-hal` which came with quite a bit of controversy due to the patch
level only bump together with a rather prominent (and thus potentially
annoying) deprecation warning.

But since `digital::v2` is here to stay, let's see how we can actually make the
switchover.

# Switching versions

The following steps are excertps from the [nrf51-hal] conversion, so if you'd
like to follow along you are invited you to check out this
[commit](https://github.com/nrf-rs/nrf51-hal/commit/6462b00bbe2ee063d410f58dd8c49a455d8589c2).

So let's get started...

## Use the right version of the traits

In order to use `digital::v2` rather than `digital::v1` you either need to
spell out the version explicitely all the time or you may want to change a use
statement to import the correct version into the namespace.

So before I used:
```Rust
use embedded_hal::digital::{InputPin, OutputPin, StatefulOutputPin};
```

and I switched that over to:

```Rust
use embedded_hal::digital::v2::{InputPin, OutputPin, StatefulOutputPin};
```

## Changing function signatures

Since the new versions always return a `Result` and the `Err` part of the
`Result` may vary, there's now an associated `Error` type which needs to be
specified.

Since (as mentioned before) for direct MCU pins the operation usually
infallible I'm specifying the type to be `core::convert::Infallible`, so we're
changing from:

```Rust
impl<MODE> OutputPin for $PXx<Output<MODE>> {
```

to:

```Rust
impl<MODE> OutputPin for $PXx<Output<MODE>> {
    type Error = Infallible;
```

and then of course we need to change the functions to return a result, too, so
we go from:

```Rust
fn set_high(&mut self) {
    ...
}

fn set_low(&mut self) {
    ...
}
```

to:

```Rust
fn set_high(&mut self) -> Result<(), Self::Error> {
    Ok(...)
}

fn set_low(&mut self) -> Result<(), Self::Error> {
    Ok(...)
}
```

Of course in some cases it's slightly more complex because the `is_*` methods
are often implemented based on the other implementation and now we need to take
into account that we're not just inverting a truth value but working with a
`Result` so we might have to go from:

```Rust
fn is_set_high(&self) -> bool {
    !self.is_set_low()
```

to something like this:

```Rust
fn is_set_high(&self) -> Result<bool, Self::Error> {
    self.is_set_low().map(|v| !v)
}
```

## Prelude

An additional thing you might want to do is to make sure that you add the used
`digital` traits to the prelude of the HAL impl so users can simply use the new
impls without much fuzz.

So here we add the following to `src/prelude.rs`:

```
pub use embedded_hal::digital::v2::InputPin as __nrf51_hal_rng_InputPin;
pub use embedded_hal::digital::v2::OutputPin as __nrf51_hal_rng_OutputPin;
pub use embedded_hal::digital::v2::StatefulOutputPin as __nrf51_hal_rng_StatefulOutputPin;
pub use embedded_hal::digital::v2::ToggleableOutputPin as __nrf51_hal_rng_ToggleableOutputPin;
pub use embedded_hal::digital::v2::toggleable as __nrf51_hal_rng_toggleable;
```

## Bump the minor version

This these changes will be breaking for your users you definitely want to well
document the change, at least by bumping the minor version so people will not
simply run nasty and unexpected troubles when compiling their code.

And with this you are done and I hope to see you again on my blog soon.

[nrf51-hal]: https://github.com/nrf-rs/nrf51-hal
[embedded-bridge]: https://github.com/therealprof/embedded-bridge
[embedded-hal]: https://crates.io/crates/embedded-hal

[1]: There might be some security features thwarting that direct acces but in the vast majority of cases this not a problem
