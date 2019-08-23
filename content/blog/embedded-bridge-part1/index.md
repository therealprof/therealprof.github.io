+++

date = 2019-08-23
title = "Introduction into a new approach to develop with and for embedded devices"
slug = "Embedded-Bridge-Part-1"

+++

Maybe you've wondered whether there was a simpler way to prototype with MCUs or
or interface with typical MCU peripherals in a simpler way than the usual code
(even in the slightly more complicated **no_std** flavour), program, debug,
test cycle. Wonder no more because now there's a approach on the horizon and
this first article will explain the concept behind that...

<!-- more -->

## The idea

As mentioned in the intro, the usual development cycle can get a bit long in
the tooth if all you want to do is quick prototyping, test, write a driver or
anything else in an iterative way for microcontrollers. Not only does
**no_std** development create some additional restrictions which require
additional effort working around but you'll also be faced with slightly less
help from code editors, slightly more involved processes to flash and run the
code (also coming with additional delays), hit the occasional out-of-flash
barrier (especially when you'd like to use a debugger and hence should use
**dev** builds) and generally have a lot less visibility about what is actually
happening.

To get rid of many of those issues I had the idea of bridging between the
hardware and a regular computer, granting full access to many MCU peripherals
from applications running on the latter. So all one has to do is flash a
special firmware once to the microcontroller and everything else will be
controlled by software running on the computer; of course using the same
[embedded-hal] interface one would use on the microcontroller, so whenever the
approach is developed/tested/proven, it can be implemented directly on the
MCU.

Intrigued? Read on then...

## The concept

As mentioned before the concept consists of a hardware specific firmware
running on the MCU as well as something which provides the [embedded-hal]
interface on the computer. Now you may wonder how those two elements will be
able to communicate with each other... In this concept that gap is bridged by a
third component which serialises peripheral specific commands and results which
are then transferred using any available streaming or packetized channel.

![Concept image](concept.svg "The building blocks of the Embedded Bridge concept")

Broken down we have three crates:
* Bridge Host: This is a crate to be used on a developer machine which provides implementations for the [embedded-hal] traits, based on a user provided communication channel (at the moment only serial, but USB is already planned and further channels are possible, e.g. BLE, Ethernet, WiFi...). The peripherals and their capabilities are created on the fly and negotiated with the Bridge Firmware.
* Bridge Common: This crate is `#[no_std]` and used both by the Bridge Host and also by the Bridge Firmware. It contains the de-/serialisation of the protocol and is used to ensure that both ends are using a compatible protocol.
* Bridge Firmware: This crate implements a `#[no_std]` firmware for microcontrollers (at the time of writing only for `STM32F042`), ingesting all peripherals and managing them via the remote protocol.

## Possible uses

There are lots of uses but one of the main uses is certainly prototyping on the
host and writing new drivers. Of course one could always find a Raspberry Pi
somewhere, put in an SD card, install Rasbpian, install a Rust toolchain and
(slowly) develop natively on the Pi but it's still rather limited compared to
development on a real development machine.

Another major usecase I can envision is cheap hardware interfacing: A cheap and
easy to implement USB connected STM32F042 can do ADC, I2C, SPI, Serial, CAN,
IR, HDMI CEC, I2S1 as well as various GPIO modes and Timer/PWM tricks.

Also it is possible to extend the reach of a microcontroller quite a bit by
removing the `#[no_std]` restrictins and flash/RAM size limitations by using a
regular computer for the heavy lifiting.

Of course the major disadvantage of the concept is speed; anything requiring
short and defined reaction times will simply not be possible with this kind of
setup. Also if you stop the host application, the microcontroller will simply
stop, waiting for new commands.

I plan to follow up this post by some example uses of mine.

## The implementation

**⚠ WARNING ⚠**: If you're easily offended by tons of deliberately `unsafe`
Rust code I would humbly suggest you abstain from continuing to read.

I've posted a first crude version at [embedded-bridge]. It includes a firmware
for the `STM32F042` MCU which works like a charm on the `NUCLEO32-F042K6` and
an blinking the on-board LED on that board over the serial port provided by the
on-board STLink and hooked up internally to the MCU. I also have code running
on the host driving a I2C SSD1306 display hooked up to the MCU.

Pull requests to clean up or implement new functionality are very much welcome!

More to come soon...

[embedded-bridge]: https://github.com/therealprof/embedded-bridge
[embedded-hal]: https://crates.io/crates/embedded-hal
