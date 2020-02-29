+++

date = 2020-02-29
title = "A look into ways to implement and share data with interrupt handlers in Rust"
slug = "interrupt-comparison"

+++

In this blog entry I will explain a bit what interrupts are and they work in
embedded systems and compare various interrupt implementation and sharing
methods in Rust.

<!-- more -->

# Interrupt handling

## Interrupt

An interrupt is a hardware method to react to special events recognized by the
system to **interrupt** regular program flow and do some special handling of the
event which caused the interrupt. This mechanism does not only exist for micro
controllers but is also present on regular CPUs. On the latter it is used
especially by external components, e.g. extension cards or connected external
peripherals which can notify the system about events (e.g. data to be
processed is available) by generating an interrupt signal. The other common case
is that interrupts are generated internally in the processing core by
configuration of certain trigger events, e.g. if some memory access violation
happened, some timer expired or the transmit buffer is empty.

To enable the use of interrupts on a system there's a special hardware element
called an **interrupt controller** which can be programmed to handle such special
events happening in the system. Interrupts have a defined or configurable
**priority** and whenever an interrupt is generated with a higher priority than
what the current execution priority, then the ongoing execution flow will be
suspended and execution continues at a special function called an **interrupt handler**...

## Interrupt handler

An interrupt handler is a special **function** which is
designed to handle one or multiple of the interrupt types an interrupt controller can
support. During the execution of an interrupt handler the regular program
execution is suspended and all processing resources are dedicated to the interrupt
handler. Most interrupt controllers are so called **nested interrupt
controllers** which means that the execution of an interrupt handler can also be
suspended to work on an even higher priority interrupt, should one occur.

As you can imagine, in a system where multiple priority levels exist, this can
be problematic. If a part of the program is working with a resource, gets
suspended and then an interrupt handler takes over and works with the same
resource, this can easily cause **race conditions**.

There're multiple ways around this and we'll talk about this later but for now
I'll briefly mention the two simplest solution for single-core systems
(multi-core systems can execute code in parallel which causes additional problems):

* Let everything run on the same priority: Without preemption there can't be
  data races (between interrupt handlers, at least)
* Disable interrupts temporarily. This is called a Critical Section (or CS) and
  we'll see this a few times in a bit.

At the end of the interrupt handler, execution will continue at the last suspension point.

# Resource sharing

There're essentially two pieces of resources you might you might want to "share"
between interrupt handlers and regular code. It is essential to appropriately
manage access from different parts of the program to them, especially for a
language like Rust which takes safety very seriously.

## Resource types

There're two important classes of data which we need to consider:

### Data

Data can be any structure located at some address in random access memory. Those can be simply variables
but also more complex structures like buffers where data piles up until it can
be processed by a different piece of code.

### Peripherals

In typical micro controllers all peripherals are accessible via memory mapped
addresses which means that these magic addresses do interact directly with
hardware when written to or read from. 

## Access management

There're 3 essential mechanisms to manage access to those resources:

### Uniqueness

Every function, interrupt handler, memory address, etc. can only exist once. For
the most part Rust already guarantees that this is the case, however it can not
do this for raw pointers to memory addresses which is the reason why it is
**unsafe** to dereference them into concrete types. However we **do** want to
have access to those memory locations but still obtain unique handles to those
resources which Rust will help us to deal with, so how do we do this?

The trick is that we're mapping those memory mapped addresses into concrete
types which give us the required abstraction to safely use those and in addition
we wrap them into a singleton so it's only possible to (safely) obtain the whole
set of resources exactly once and let the Rust compiler handle most of that for us.

### Sharing

Usually sharing resources in Rust is simple, you just pass borrows around and
the Rust borrow checker will ensure that the lifetimes of the borrowed resources
are not mixed up and that a mutable borrow and a regular borrow are mutually
exclusive.

However things get more tricky if there're non-linear program flows involved as
it is the case with threads and interrupt handlers. As soon as there's an
external party involved which can interfere with program execution flow like a
operating system scheduler or interrupt controller, all shared resources require
special protection which know how to deal with a particular case. For threads we
have the `Sync` trait to cover concurrent access however this assumes that there
is an overseeing party which makes sure that some over-the-top guarantees are
upheld, however there're cases in embedded where this cannot not work.

### Moving

Similarly to sharing, moving now becomes a problem because the Rust compiler
cannot track resources which are supposed to move into e.g. an interrupt handler.
To the Rust compilers' eye those are not even part of **the** program itself but
more a kind of standalone functions which are never even called and for some weird
reason just happen to be defined in the same code base are the "real" program.
Silly developers!

So special precautions need to be applied to a resource which is initialized in
once part of the "real" program but then supposed to move to a different
location which is not seen as part of the "real" program but a weird function
which seemingly unused.

# Implementation approaches

Over the last couple of years a few mechanisms have been developed and evolved
to deal with the implementation and handling of the before mentioned shenanigans
in a sane way.

For the rest of the post I will look at different approaches to handle
interrupts, share and move resources. For this I've developed a simple example
application for a Cortex-M0 MCU and we will look at the different
implementations with the different approaches, compare the generated code sizes
and I will give you my opinion about the pros and cons of the various methods.

So this is what the end result will have to look like:

<video src="Nucleo_flashing.mp4" loop controls></video>

All of the code can be found in the [interrupt-comparison] repo under the
`flashing` workspace.

So these are the contenders:

## Cortex-m-rt with delay

This is almost the most basic approach one could take without interrupts or an
kind of sharing and thus our primitive "baseline":
* Set up the hardware and a timer
* Busy-wait for the timer to expire
* Turn LED on or off

### Code

```rust
#![no_main]
#![no_std]

use panic_halt as _;
use stm32f0xx_hal::{delay::Delay, prelude::*, stm32};
use cortex_m::peripheral::Peripherals;
use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    if let (Some(mut p), Some(cp)) = (stm32::Peripherals::take(), Peripherals::take()) {
        let mut state: u8 = 0;

        let (mut led, mut delay) = cortex_m::interrupt::free(|cs| {
            // Configure clock to 48 MHz (i.e. the maximum) and freeze it
            let mut rcc = p.RCC.configure().sysclk(48.mhz()).freeze(&mut p.FLASH);

            // Obtain resources from GPIO port A
            let gpioa = p.GPIOA.split(&mut rcc);

            // (Re-)configure PA5 as output
            let led = gpioa.pa5.into_push_pull_output(cs);

            // Get delay provider
            let delay = Delay::new(cp.SYST, &rcc);

            (led, delay)
        });

        loop {
            if state < 10 {
                // Turn off the LED
                led.set_low().ok();
                state += 1;
            } else {
                // Turn on the LED
                led.set_high().ok();
                state = 0;
            }
            delay.delay_ms(100_u16);
        }
    }

    loop {
        continue;
    }
}
```

### Binary results

#### Optimized

```
Bloat for example flashing_delay
File  .text Size          Crate Name
0.2%  74.5% 328B flashing_delay flashing_delay::__cortex_m_rt_main
0.0%  14.5%  64B    cortex_m_rt Reset
0.0%   1.4%   6B      [Unknown] main
0.0%   0.5%   2B    cortex_m_rt HardFault_
0.0%   0.5%   2B    cortex_m_rt DefaultPreInit
0.0%   0.5%   2B    cortex_m_rt DefaultHandler_
0.0%   0.0%   0B                And 0 smaller methods. Use -n N to show more.
0.3% 100.0% 440B                .text section size, the file size is 154.7KiB

Section sizes:
   text    data     bss     dec     hex filename
    632       0       4     636     27c flashing_delay
```

#### Unoptimized

```
Bloat for example flashing_delay
File  .text   Size         Crate Name
0.5%  15.2% 1.4KiB stm32f0xx_hal stm32f0xx_hal::rcc::CFGR::freeze
0.2%   6.1%   592B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_clock
0.2%   5.0%   482B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_pll
0.1%   2.9%   280B stm32f0xx_hal stm32f0xx_hal::rcc::CFGR::freeze::{{closure}}
0.1%   2.8%   276B stm32f0xx_hal stm32f0xx_hal::gpio::gpioa::PA5<MODE>::into_push_pull_output
0.1%   2.1%   206B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_clock::{{closure}}
0.1%   2.1%   204B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_clock::{{closure}}
0.1%   1.9%   180B stm32f0xx_hal <stm32f0xx_hal::delay::Delay as embedded_hal::blocking::delay::DelayUs<u32>>::delay_us
0.1%   1.9%   180B stm32f0xx_hal stm32f0xx_hal::rcc::CFGR::freeze::{{closure}}
0.1%   1.7%   162B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_pll::{{closure}}
1.7%  57.8% 5.5KiB               And 109 smaller methods. Use -n N to show more.
3.0% 100.0% 9.5KiB               .text section size, the file size is 312.7KiB

Section size:
   text    data     bss     dec     hex filename
  11628       0       4   11632    2d70 flashing_delay
```

### Pros

* Doesn't get any simpler than that

### Cons

* Only useful to build simple reactive systems, no hard real-time possible
* Asynchronous handling is really tedious

## Cortex-m-rt with interrupts and moved resources

So compared to our baseline, here we'll also:

* Explicitly set up the interrupt handling
* Define a `Mutex` protected resource for the LED so we can use the hardware
  safely from the interrupt handler
* Define static resources in our interrupt handler which exclusively are
  accessible from there
* Move out the LED pin from the `Mutex` protected resource into our local static
  resource on first execution of the interrupt handler

### Code

```rust
#![no_main]
#![no_std]

use panic_halt as _;
use stm32f0xx_hal::{gpio::*, prelude::*, stm32};
use cortex_m::{interrupt::Mutex, peripheral::syst::SystClkSource::Core, Peripherals};
use cortex_m_rt::{entry, exception};
use core::cell::RefCell;

// Mutex protected structure for our shared GPIO pin
static GPIO: Mutex<RefCell<Option<flashing::LEDPIN>>> = Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    if let (Some(mut p), Some(mut cp)) = (stm32::Peripherals::take(), Peripherals::take()) {
        cortex_m::interrupt::free(move |cs| {
            // Configure clock to 48 MHz (i.e. the maximum) and freeze it
            let mut rcc = p.RCC.configure().sysclk(48.mhz()).freeze(&mut p.FLASH);

            // Get access to individual pins in the GPIO port
            let gpioa = p.GPIOA.split(&mut rcc);

            // (Re-)configure the pin connected to our LED as output
            let led = gpioa.pa5.into_push_pull_output(cs);

            // Transfer GPIO into a shared structure
            *GPIO.borrow(cs).borrow_mut() = Some(led);

            // Set source for SysTick counter, here full operating frequency (== 48MHz)
            cp.SYST.set_clock_source(Core);

            // Set reload value, i.e. timer delay 48 MHz/4 Mcounts == 12Hz or 83ms
            cp.SYST.set_reload(4_000_000 - 1);

            // Start counting
            cp.SYST.enable_counter();

            // Enable interrupt generation
            cp.SYST.enable_interrupt();
        });
    }

    loop {
        continue;
    }
}

// Define an exception handler, i.e. function to call when exception occurs. Here, if our SysTick
// timer generates an exception the following handler will be called
#[exception]
fn SysTick() -> () {
    // Our moved LED pin
    static mut LED: Option<flashing::LEDPIN> = None;

    // Exception handler state variable
    static mut STATE: u8 = 0;

    // If LED pin was moved into the exception handler, just use it
    if let Some(led) = LED {
        // Check state variable, keep LED off most of the time and turn it on every 10th tick
        if *STATE < 10 {
            // Turn off the LED
            led.set_low().ok();

            // And now increment state variable
            *STATE += 1;
        } else {
            // Turn on the LED
            led.set_high().ok();

            // And set new state variable back to 0
            *STATE = 0;
        }
    }
    // Otherwise move it out of the Mutex protected shared region into our exception handler
    else {
        // Enter critical section
        cortex_m::interrupt::free(|cs| {
            // Move LED pin here, leaving a None in its place
            LED.replace(GPIO.borrow(cs).replace(None).unwrap());
        });
    }
}
```

### Binary results

#### Optimized

```
Bloat for example flashing_rt_move
File  .text Size            Crate Name
0.2%  52.9% 296B flashing_rt_move flashing_rt_move::__cortex_m_rt_main
0.1%  22.9% 128B        [Unknown] SysTick
0.0%  11.4%  64B      cortex_m_rt Reset
0.0%   1.1%   6B              std core::result::unwrap_failed
0.0%   1.1%   6B              std core::panicking::panic_fmt
0.0%   1.1%   6B              std core::panicking::panic
0.0%   1.1%   6B        [Unknown] main
0.0%   0.4%   2B      cortex_m_rt HardFault_
0.0%   0.4%   2B       panic_halt rust_begin_unwind
0.0%   0.4%   2B      cortex_m_rt DefaultPreInit
0.0%   0.4%   2B      cortex_m_rt DefaultHandler_
0.0%   0.0%   0B                  And 0 smaller methods. Use -n N to show more.
0.3% 100.0% 560B                  .text section size, the file size is 169.3KiB

Section sizes:
   text    data     bss     dec     hex filename
    796       0      12     808     328 flashing_rt_move
```

#### Unoptimized

```
Bloat for example flashing_rt_move
File  .text    Size         Crate Name
0.4%  12.0%  1.4KiB stm32f0xx_hal stm32f0xx_hal::rcc::CFGR::freeze
0.3%   9.6%  1.1KiB           std core::fmt::Formatter::pad
0.2%   4.8%    592B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_clock
0.1%   3.9%    482B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_pll
0.1%   2.3%    280B stm32f0xx_hal stm32f0xx_hal::rcc::CFGR::freeze::{{closure}}
0.1%   2.3%    276B stm32f0xx_hal stm32f0xx_hal::gpio::gpioa::PA5<MODE>::into_push_pull_output
0.1%   1.7%    210B           std core::ptr::swap_nonoverlapping_bytes
0.1%   1.7%    206B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_clock::{{closure}}
0.1%   1.7%    204B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_clock::{{closure}}
0.0%   1.5%    180B stm32f0xx_hal stm32f0xx_hal::rcc::CFGR::freeze::{{closure}}
1.9%  58.0%  6.9KiB               And 150 smaller methods. Use -n N to show more.
3.3% 100.0% 12.0KiB               .text section size, the file size is 357.7KiB

Section size:
   text    data     bss     dec     hex filename
  14180       0      12   14192    3770 flashing_rt_move
```

### Pros

* Easy to understand and straight-forward "text-book" syntax

### Cons

* Requires use of `static` `Mutex` which have annoyingly exact type specification requirements
* `Mutex` require critical sections to access the values which block all interrupt processing

## `IRQ`

This uses the [irq] crate from Jonas Schievink which takes a slightly different
approach than [cortex-m-rt] but still uses it under the hood to do the interrupt
handling. Instead of defining the interrupt handlers manually the `irq` crate
provides a `scoped_interrupts` macro to define the interrupts which are turned
into regular `cortex-m-rt` interrupt handler functions. The real handlers are
defined as closures within the main program flow and parsed to a special `scope`
function while will register the real handler closure. As long as the `scope` is
active any triggered interrupt handler internally created by `scoped_interrupts`
will call the registered closure to do the real processing.

### Code

```rust
#![no_main]
#![no_std]

use panic_halt as _;

use stm32f0xx_hal::{gpio::*, prelude::*, stm32};

use cortex_m::{peripheral::syst::SystClkSource::Core, Peripherals};
use cortex_m_rt::{entry, exception};
use irq::{handler, scope, scoped_interrupts};

// Hook `SysTick` using the `#[exception]` attribute
scoped_interrupts! {
    enum Exception {
        SysTick,
    }

    use #[exception];
}

#[entry]
fn main() -> ! {
    if let (Some(mut p), Some(mut cp)) = (stm32::Peripherals::take(), Peripherals::take()) {
        let mut led = cortex_m::interrupt::free(move |cs| {
            // Configure clock to 48 MHz (i.e. the maximum) and freeze it
            let mut rcc = p.RCC.configure().sysclk(48.mhz()).freeze(&mut p.FLASH);

            // Get access to individual pins in the GPIO port
            let gpioa = p.GPIOA.split(&mut rcc);

            // (Re-)configure the pin connected to our LED as output
            let led = gpioa.pa5.into_push_pull_output(cs);

            // Set source for SysTick counter, here full operating frequency (== 48MHz)
            cp.SYST.set_clock_source(Core);

            // Set reload value, i.e. timer delay 48 MHz/4 Mcounts == 12Hz or 83ms
            cp.SYST.set_reload(4_000_000 - 1);

            // Start counting
            cp.SYST.enable_counter();

            // Enable interrupt generation
            cp.SYST.enable_interrupt();

            led
        });

        // State variable
        let mut state: u8 = 0;

        handler!(
            systick = || {
                // Check state variable, keep LED off most of the time and turn it on every 10th tick
                if state < 10 {
                    // Turn off the LED
                    led.set_low().ok();

                    // And now increment state variable
                    state += 1;
                } else {
                    // Turn on the LED
                    led.set_high().ok();

                    // And set new state variable back to 0
                    state = 0;
                }
            }
        );

        // Create a scope and register the handlers
        scope(|scope| {
            scope.register(Exception::SysTick, systick);

            loop {
                continue;
            }
        });
    }

    loop {
        continue;
    }
}
```

### Binary results

#### Optimized

```
Bloat for example flashing_irq
File  .text Size        Crate Name
0.2%  55.7% 312B flashing_irq flashing_irq::__cortex_m_rt_main
0.0%  11.4%  64B  cortex_m_rt Reset
0.0%   7.1%  40B flashing_irq flashing_irq::__cortex_m_rt_main::{{closure}}
0.0%   7.1%  40B    [Unknown] SysTick
0.0%   6.4%  36B          std core::ops::function::FnOnce::call_once{{vtable.shim}}
0.0%   1.1%   6B          std core::panicking::panic_fmt
0.0%   1.1%   6B          std core::panicking::panic
0.0%   1.1%   6B    [Unknown] main
0.0%   0.4%   2B  cortex_m_rt HardFault_
0.0%   0.4%   2B   panic_halt rust_begin_unwind
0.0%   0.4%   2B  cortex_m_rt DefaultPreInit
0.0%   0.4%   2B  cortex_m_rt DefaultHandler_
0.0%   0.4%   2B          std core::ptr::real_drop_in_place
0.0%   0.0%   0B              And 0 smaller methods. Use -n N to show more.
0.3% 100.0% 560B              .text section size, the file size is 162.4KiB

Section sizes:
   text    data     bss     dec     hex filename
    808       0      12     820     334 flashing_irq
```

#### Unoptimized

```
Bloat for example flashing_irq
File  .text   Size         Crate Name
0.4%  14.7% 1.4KiB stm32f0xx_hal stm32f0xx_hal::rcc::CFGR::freeze
0.2%   5.9%   592B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_clock
0.1%   4.8%   482B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_pll
0.1%   2.8%   280B stm32f0xx_hal stm32f0xx_hal::rcc::CFGR::freeze::{{closure}}
0.1%   2.8%   276B stm32f0xx_hal stm32f0xx_hal::gpio::gpioa::PA5<MODE>::into_push_pull_output
0.1%   2.1%   206B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_clock::{{closure}}
0.1%   2.0%   204B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_clock::{{closure}}
0.1%   1.8%   180B stm32f0xx_hal stm32f0xx_hal::rcc::CFGR::freeze::{{closure}}
0.0%   1.6%   162B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_pll::{{closure}}
0.0%   1.5%   148B stm32f0xx_hal stm32f0xx_hal::rcc::CFGR::freeze::{{closure}}
1.8%  59.3% 5.8KiB               And 121 smaller methods. Use -n N to show more.
3.0% 100.0% 9.8KiB               .text section size, the file size is 321.0KiB

Section size:
   text    data     bss     dec     hex filename
  12148       0       8   12156    2f7c flashing_irq
```

### Pros

* All data is in the same single scope so rustc can do its work properly
* No overhead for resource transfer and local storage
* Can change interrupt handlers on the fly

### Cons

* Overhead for the handler indirection

## cortex-m-rtfm

[cortex-m-rtfm] is a very clever toolbox combining a macro based domain specific
language annotation with static analysis to provide ideal results with
calculable worst time interrupt execution times and lock-free operation. 

### Code

```rust
#![no_main]
#![no_std]

use panic_halt as _;
use stm32f0xx_hal::{gpio::*, prelude::*, stm32};
use cortex_m::peripheral::syst::SystClkSource::Core;
use rtfm::app;

#[app(device = crate::stm32, peripherals = true)]
const APP: () = {
    // Late resources
    struct Resources {
        led: flashing::LEDPIN,
    }

    #[init]
    fn init(cx: init::Context) -> init::LateResources {
        // Cortex-M peripherals
        let mut core = cx.core;

        // Device specific peripherals
        let mut device = cx.device;

        // Configure clock to 48 MHz (i.e. the maximum) and freeze it
        let mut rcc = device
            .RCC
            .configure()
            .sysclk(48.mhz())
            .freeze(&mut device.FLASH);

        // Get access to individual pins in the GPIO port
        let gpioa = device.GPIOA.split(&mut rcc);

        let led = cortex_m::interrupt::free(move |cs| {
            // (Re-)configure the pin connected to our LED as output
            gpioa.pa5.into_push_pull_output(cs)
        });

        // Set source for SysTick counter, here full operating frequency (== 48MHz)
        core.SYST.set_clock_source(Core);

        // Set reload value, i.e. timer delay 48 MHz/4 Mcounts == 12Hz or 83ms
        core.SYST.set_reload(4_000_000 - 1);

        // Start counting
        core.SYST.enable_counter();

        // Enable interrupt generation
        core.SYST.enable_interrupt();

        init::LateResources { led }
    }

    // Define an exception handler, i.e. function to call when exception occurs. Here, if our SysTick
    // timer generates an exception the following handler will be called
    #[task(binds = SysTick, priority = 1, resources = [led])]
    fn systick(c: systick::Context) {
        // Exception handler state variable
        static mut STATE: u8 = 0;

        // If LED pin was moved into the exception handler, just use it
        // Check state variable, keep LED off most of the time and turn it on every 10th tick
        if *STATE < 10 {
            // Turn off the LED
            c.resources.led.set_low().ok();

            // And now increment state variable
            *STATE += 1;
        } else {
            // Turn on the LED
            c.resources.led.set_high().ok();

            // And set new state variable back to 0
            *STATE = 0;
        }
    }
};
```

### Binary results

#### Optimized

```
Bloat for example flashing_rtfm
File  .text Size       Crate Name
0.1%  61.5% 236B   [Unknown] main
0.0%  16.7%  64B cortex_m_rt Reset
0.0%  10.4%  40B   [Unknown] SysTick
0.0%   0.5%   2B cortex_m_rt HardFault_
0.0%   0.5%   2B cortex_m_rt DefaultPreInit
0.0%   0.5%   2B cortex_m_rt DefaultHandler_
0.0%   0.0%   0B             And 0 smaller methods. Use -n N to show more.
0.2% 100.0% 384B             .text section size, the file size is 154.8KiB

Section sizes:
   text    data     bss     dec     hex filename
    576       0       4     580     244 flashing_rtfm
```

#### Unoptimized

```
Bloat for example flashing_rtfm
File  .text    Size         Crate Name
0.4%  12.8%  1.4KiB stm32f0xx_hal stm32f0xx_hal::rcc::CFGR::freeze
0.3%   8.9%   1018B           std core::fmt::Formatter::pad_integral
0.2%   5.2%    592B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_clock
0.1%   4.2%    482B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_pll
0.1%   2.8%    320B           std core::fmt::num::imp::fmt_u32
0.1%   2.4%    280B stm32f0xx_hal stm32f0xx_hal::rcc::CFGR::freeze::{{closure}}
0.1%   2.4%    276B stm32f0xx_hal stm32f0xx_hal::gpio::gpioa::PA5<MODE>::into_push_pull_output
0.1%   1.8%    206B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_clock::{{closure}}
0.1%   1.8%    204B stm32f0xx_hal stm32f0xx_hal::rcc::inner::enable_clock::{{closure}}
0.1%   1.8%    204B      cortex_m cortex_m::peripheral::scb::<impl cortex_m::peripheral::SCB>::set_priority::{{closure}}
1.9%  55.5%  6.2KiB               And 117 smaller methods. Use -n N to show more.
3.4% 100.0% 11.2KiB               .text section size, the file size is 331.1KiB

Section size:
   text    data     bss     dec     hex filename
  14100       0       4   14104    3718 flashing_rtfm
```


### Pros

* Look ma! No locks!
* Extremely powerful possibilities, especially when different priorities are used
* Great compile time checks and guarantees
* Incredibly efficient binary code

### Cons

* Custom syntax feels the least idiomatic of all approaches
* Additional implementation complexity even if the full functionality is not required

# Epilog

Exploring the various options to use interrupts was quite a fun ride and also
quite enlightening. I've spent a bit of time to look under the hood of the
various approaches in order to understand what they're doing and whether there
might be potential to combine approaches or improve on them for future work.

As can be seen each of the approaches comes with a their own set of drawbacks
and it would be great if there was a way to combine them in a way that combines
the best of all worlds: The straight-forwardness of `cortex-m` with the
simplicity of `irq` but the power, efficiency and compile time guarantees of
`cortex-m-rtfm`.

Also it would be great to have an approach which covers more architectures than
ARM Cortex-M. However to cover all of Cortex-M I specifically opted to use a
Cortex-M0 which lacks e.g. compare-and-swap options and thus resembles the
lowest possible denominator for this particular vendor.

You can find all the code and tools used to conduct the tests in my repository
at [interrupt-comparison]. Feel free to experiment and let me know your
experiences. Also let me know if you have another approach you feel should be
covered here.

I was trying to include [cmim] in this comparison but I failed to get the
`SysTick` exception to work and failed. If this gets addressed I might update
this blog post with another implementation.

Thanks for sticking with me for this long and I hope to see you another time around.

[cmim]: https://crates.io/crates/cmim
[cortex-m-rt]: https://crates.io/crates/cortex-m-rt
[cortex-m-rtfm]: https://crates.io/crates/cortex-m-rtfm
[irq]: https://crates.io/crates/irq
[interrupt-comparison]: https://github.com/therealprof/interrupt-comparison



