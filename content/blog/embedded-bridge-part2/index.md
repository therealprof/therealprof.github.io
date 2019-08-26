+++

date = 2019-08-26
title = "A deeper look into embedded-bridge with an example"
slug = "Embedded-Bridge-Part-2"

+++

In the previous part I introduced the concept of [embedded-bridge]. Now let's
dive a little bit deeper and have a look at a real life example.

<!-- more -->

# Implementation details

## Low level communication

As I mentioned, the concept is to bridge the hardware peripherals from the MCU
(or target) over to the host via some available communication port which is
available both on the target and the host. The communication channel needs to
be bi-directional but other than that the sky is the limit.

### UART / serial port

One of the most common choices for communication with an MCU is a UART
interface with TTL voltages (usually 3.3 volts), often referred to as a "serial
port". While no computer in the world offers a native 3.3V connection to other
devices this is predominant interface to talk to and programm microcontrollers
of all kind or load bitstreams into FPGAs that a lot of manufacturers through a
USB to serial adapter chip onto a lot of boards or build it into their MCU
debugging interfaces. This makes it a natural choice for the first
implementation, so that's what it is.

### USB

Also a lot of MCUs have USB support built-in which is fantastic because it
allows direct connections from a computer to a MCU. It is harder to implement
both in the hardware design (since it usually requires additional care to
obtain the precise clocks needed for USB) and on the software side but it is
usually hardware assisted and thus a lot faster and more ubiquitous.
[embedded-bridge] does not support it yet but this is one of the next items on
the todo list to make happen.

### BLE

Something which would be really nice to have but will not happen without
volunteers is Bluetooth Low Energy support. It's not the fastest protocol in
the world nor is it easy to handle but there're few MCUs out there which
natively support it and addding support would allow any modern laptop or mobile
phone to use [embedded-bridge] entirely remotely which would be really nice.

### Ethernet / WiFi

Quite a few chips have built-in Ethernet or WiFi or allow easy external
additional of such interfaces. Similarly to BLE this would allow remote control 
over the [embedded-bridge], but unlike BLE this could also be done out of the
reach of BLE from far away which would also be nice to have.

## Higher level protocol

At the higher level the `bridge-common` submodule provides serialisable enums
which are turned into efficient binary code with the help of `serde` and [postcard].

The enums are "task based", so each action which can be taken at a peripheral
will receive their own enum. This includes intialisation, setting, clearing,
sending and receiving per peripheral type.

By having `bridge-common` implemented as a `no_std` compatible module this can
both be used on the firmware and on the host side, ensuring that both parties
are implementing the protocol in the exact same way. The latest updates also
include versioning, allowing the host to ensure that the firmware is running on
the same version of the protocol before attempting to use it.

To make things even more obvious (and the peripheral use fully transparent),
the host side allows for en- and decoding of all packets, allowing to trace all
data exchange between the host and peripherals; we'll see more about this with
the example later.

The serialised data is pushed over the lower layer, picked up at the target,
decoded, processed, a reply encoded and sent back the host over the same
channel.

On the target side all processing is going through the HAL abstraction, albeit
with some trickery to simplify the state handling dramatically; if I find a way
to address this I'll do it but it's quite likely that it will stay that way.

On the host side one can either use a high levellish API or and [embedded-hal]
compatible abstraction layer on top of it which allows to use (or develop)
regular drivers directly on the host.

# A first example

## Meet the hardware

My development platform of choice is the [STM Nucleo32-F042K6] which is a cute
little and very affordable little board consisting of a STMicroelectronics
STM32F042 ARM Cortex-M0 MCU and a ST-Link/V2 debugger in a breadboard friendly
form factor. The MCU is not only very featurerich and cheap, it is also very
makerfriendly, which makes it one of my go-to choices for simple custom
hardware designs. The built-in ST-Link debugger with USB serial support (and
internal serial connection between ST-Link and MCU) also makes it usable
out-of-the-box for this purpose. It even supports crystal-less USB, however
there's no second USB socket on the board to use that.

The example in this blog post can be directly followed with this board.

## Getting started

You can follow along all examples by checking out the [embedded-bridge]
repository. I'll only highlight the important bits and commands here.

After checking out the repository the first step is to flash the generic
firmwware onto the MCU. To do this, you'll have to change into the
`embedded-firmware` subdirectory and (after installing all the tooling for
compiling Rust code on a Cortex-M0 MCU) compile it:
```
cargo build --release
```

The binary will end up in
`../target/thumbv6m-none-eabi/release/bridge-firmware` and needs to be
flashed to the target (after hooking it up via USB that is):
```
openocd -f nucleo.cfg -c "init" -c "targets" -c "reset halt" -c "program ../target/thumbv6m-none-eabi/release/bridge-firmware verify reset exit"
```

If this was successful you should see something like:
```
Info : clock speed 1000 kHz
Info : STLINK V2J31M21 (API v2) VID:PID 0483:374B
Info : Target voltage: 3.260853
Info : stm32f0x.cpu: hardware has 4 breakpoints, 2 watchpoints
Info : Listening on port 3333 for gdb connections
    TargetName         Type       Endian TapName            State
--  ------------------ ---------- ------ ------------------ ------------
 0* stm32f0x.cpu       hla_target little stm32f0x.cpu       halted

Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
target halted due to debug-request, current mode: Thread
xPSR: 0xc1000000 pc: 0x0800325c msp: 0x20001800
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
target halted due to debug-request, current mode: Thread
xPSR: 0xc1000000 pc: 0x0800325c msp: 0x20001800
Info : Unable to match requested speed 8000 kHz, using 4000 kHz
Info : Unable to match requested speed 8000 kHz, using 4000 kHz
** Programming Started **
Info : device id = 0x10006445
Info : flash size = 32kbytes
** Programming Finished **
** Verify Started **
** Verified OK **
** Resetting Target **
```

Once this is done, the setup is complete and you can now use the peripherals on
the host.

## Using the peripherals from the host side

Let's start with something simple, the good old blinky. The board contains an
LED at `PB3` so we can use that.

Backing out of `embedded-firmware` and going to `embedded-host` we can now talk
to the device via the built-in USB serial. To do so we create some boilerplate
to talk to the USB serial adapter which you can find in any of the
`nucleo_f042` examples; just copy an example and you're good to go.  The
initialisation should give you a serial port which we'll store in the variable
`port` and will use from here on.

You should always start your code by assuring you're using the same version of
the protocol as the firmware flashed onto the target, this can be done with
```rust
bridge_host::common::assert_version(port.clone());
```

Now to the fun part. We can get a [embedded-hal] `OutputPin` compatible object
for said pin `PB3` (the one connected to the green LED) by calling
```rust
let mut pin = bridge_host::gpio::PushPullPin::new("b3".into(), port.clone());
```

You can pass this object to anything taking an object requiring the `OutputPin`
trait. You can either call the trait methods directly or pass it on to any
driver requiring such an object.

For now we're just going to blink it:
```rust
loop {
    pin.set_low();
    std::thread::sleep(std::time::Duration::from_millis(500));

    pin.set_high();
    std::thread::sleep(std::time::Duration::from_millis(500));
}
```

This will turn the green LED off, wait for half a second (on the host), turn it
on again and wait another half a second before starting over.

You can compile and run the program like any other program you'd run on your
local machine which for me is:
```
cargo run --release --example nucleo_f042_gpio_blinky -- /dev/tty.usbmodem144423
```

Now watcht the LED blink and toy around as much as you'd like and I'll see you
with some more advanced article here soon...

[STM Nucleo32-F042K6]: https://os.mbed.com/platforms/ST-Nucleo-F042K6/
[embedded-bridge]: https://github.com/therealprof/embedded-bridge
[embedded-hal]: https://crates.io/crates/embedded-hal
[postcard]: https://crates.io/crates/postcard
