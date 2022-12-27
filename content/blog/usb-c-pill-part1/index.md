+++

date = 2020-08-15
title = "Introducing: The USB-C pill"
slug = "USB-C-Pill-Part1"

+++

In this blog entry I will explore a new excellent and cheap embedded hardware
board I came across in the context of embedded Rust. This is done in the hope
that people will jump on it and drop the crapadelic boards known as "BluePill"
like a hot potato.

<!-- more -->

# Foreword

As usual with ubiquitous hardware there's one company coming up with the idea
and initial design and as soon as the product is out, a ton of (often) Chinese
companies jump on and produce clones of (sometimes) questionable quality, often
underbidding the original company price-wise and thus destroying their business
case.

I want to make it very clear that **I** do not endorse this kind of practise
and recommend buying the original. For this reason I will make abundantly clear
what the original is and only point to authentic sources. The original is in
this case not even more expensive so there's no excuse to buy a clone!

In this particular case we're talking about the **[STM32F4x1 MiniF4]** from
**[WeAct Studio]**. They do provide legitimate [purchasing sources] so I'd
recommend checking there if you're interested.

# The hardware

The [STM32F4x1 MiniF4] exists in two different variants, one with a STM32F401
and one with a STM32F411 MCU. The STM32F401 has fewer resources and peripherals
but the look and feel as well as the handling is pretty much the same.

Someone has created a handy pinout sheet:

<img src="https://github.com/WeActStudio/WeActStudio.MiniSTM32F4x1/raw/master/images/STM32F4x1_PinoutDiagram_RichardBalint.png" alt="STM32F4x1 MiniF4 pinout" width="100%"/>

A few things worth noting here:
- A beautiful design with bevelled edges, curvy traces, small SMD parts, proper and useful silk screen and ENIG (probably only on the higher end version)
- USB-C is properly connected, also for data and we're going to make use of that
- The underside has an unpopulated footprint for SPI flash if you want to add that
- The STM32F4xx series has a proper USB bootloader in ROM so we can use the board without extra hardware
- The current version always has 512kB of flash which is plenty
- [WeAct Studio] also provices a CMSDIS-DAP firmware, so if you get two boards you can turn one into a debugger and work on the other
- It's not too obvious but the push button labelled KEY is hooked up to PA0
- The high speed crystal clocks in at 25MHz, there's also a low speed crystal at 32.768kHz for the RTC

# Getting started

The setup is super simple which makes it easy to use for beginners. If you
don't have soldering equipment you probably want to find someone who can solder
in the included headers which makes it a lot simpler to connect external
hardware or put it on a breadboard but other than that you're ready to go.

In order to load (aka flash) software onto these boards there're a number of
options but I'm focusing on the two most promising approaches:

## The easy route

As mentioned above the STM32F4x1 MCUs both have their USB-C connection properly
wired **and** the built-in bootloader supports flashing directly over it and we
have the required buttons wired to trigger the procedure 👍, so that's what
we're going to do now.

First off we need to have `dfu-util` installed; how to get them is left as an
exercise to the reader. Then we should also have `cargo-binutils` installed
which allows us to easily turn a binary into the required format for
`dfu-util`.

So if we assume we're in the checked out `stm32f4xx-hal` repository and we
wanted to flash the `stopwatch-with-ssd1306-and-interrupts` example to our
board (spoiler alert: This example will compile as is but not work fully
without code changes) we'd do these three steps:

1.
  ```
  # cargo objcopy --release --example stopwatch-with-ssd1306-and-interrupts --features="stm32f411,rt" -- -O binary out.bin
      Finished release [optimized + debuginfo] target(s) in 0.03s
  ```
  to compile the example for a STM32F411 MCU with `--release` profile and convert it to a binary file `out.bin`
2.  Hold the `NRST` and `BOOT0` buttons at the same time, then let go of `NRST` while still holding `BOOT0` button for a second longer
3.
  ```
  # dfu-util -a0 -s 0x08000000  -D out.bin
  dfu-util 0.9

  Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
  Copyright 2010-2016 Tormod Volden and Stefan Schmidt
  This program is Free Software and has ABSOLUTELY NO WARRANTY
  Please report bugs to http://sourceforge.net/p/dfu-util/tickets/

  dfu-util: Invalid DFU suffix signature
  dfu-util: A valid DFU suffix will be required in a future dfu-util release!!!
  Opening DFU capable USB device...
  ID 0483:df11
  Run-time device DFU version 011a
  Claiming USB DFU Interface...
  Setting Alternate Setting #0 ...
  Determining device status: state = dfuERROR, status = 10
  dfuERROR, clearing status
  Determining device status: state = dfuIDLE, status = 0
  dfuIDLE, continuing
  DFU mode device DFU version 011a
  Device returned transfer size 2048
  DfuSe interface name: "Internal Flash  "
  Downloading to address = 0x08000000, size = 27904
  Download	[=========================] 100%        27904 bytes
  Download done.
  File downloaded successfully
  ```

After pushing the `NRST` button again your program will run.

## The powerful route

On the other end we have the ability to connect a debug probe and not only
flash that way but also debug and use some of the fancy new capabilities
provided by the various **[probe-rs]** projects.

In order to use this route you will need to have a debug probe hardware around.
There're various routes to obtain one so I'm only going to mention the one
involving the board we're already talking about.

[WeAct Studio] published a [CMSIS-DAP] firmware for our board. Download it and
make sure have `cargo-binutils` installed. Then let's convert the downloaded
file to something `dfu-util` can handle:

```
# rust-objcopy -Iihex -O binary CMSIS-DAP_WeActStudio.hex out.bin
```

Now flash the `out.bin` as described above to another board. After a reset you
have debug probe available for our next steps.

Conveniently that `CMSIS-DAP` debug probe firmware uses the same pins for probe
operation as the board to be debugged so by simply hooking up the four pins on
the end 1:1 to another board you're all set and can even power up the other
board. I also had success by not soldering any headers to the debug probe board
and simply funnelling the other header through the holes and letting it sit at
an angle. YMMV.

With that out of the way we can finally use all the goodies the embedded Rust
ecosystem now has to offer, e.g. by installing `cargo-flash` we can now build
and flash the firmware directly onto the board in a single step:
```
# cargo flash --release --example stopwatch-with-ssd1306-and-interrupts --features="stm32f411,rt" --chip stm32f411ceux
    Finished release [optimized + debuginfo] target(s) in 0.03s
    Flashing target/thumbv7em-none-eabihf/release/examples/stopwatch-with-ssd1306-and-interrupts
        WARN probe_rs::architecture::arm::core::m4 > Reason for halt has changed, old reason was Halted(Request), new reason is Exception
        WARN probe_rs::architecture::arm::core::m4 > Reason for halt has changed, old reason was Halted(Breakpoint), new reason is Request
        WARN probe_rs::architecture::arm::core::m4 > Reason for halt has changed, old reason was Halted(Request), new reason is Exception
     Erasing sectors ✔ [00:00:00] [############################################]  32.00KB/ 32.00KB @  36.90KB/s (eta 0s )
 Programming pages   ✔ [00:00:02] [############################################]  28.00KB/ 28.00KB @   9.32KB/s (eta 0s )
```

Of course flashing is just the beginning here but I won't go into debugging here.

# Next steps

Now that we are acquainted with the hardware and the basic setup, it's time to
work on some software. I'll leave some concrete examples to follow up posts but
of course I'm not going to hang out to dry here. 😄

We have excellent community support for these MCUs in `[stm32f4xx-hal]` from
which I've also taken the example code above. If you're going to use this
repository directly, there're a few caveats to watch out for:

- You need to specify the MCU every time you build an application
- The flash and RAM settings are very conservative to fit all possible MCUs in the family but these boards have plenty of flash and RAM which are not going to be utilized
- Some examples don't work on all models and cargo doesn't allow to specify alternative options, if in doubt, adjust `Cargo.toml`.
- Some examples expect hardware to be connected to specific peripherals/pins and might need adjusting
- Some examples rely on semi-hosting and will thus not do anything useful out-of-the-box (also friends don't let friends use semi-hosting!)

I'd highly recommend you start a new project and use [stm32f4xx-hal] as a
dependency and set up everything to your liking in the project which will allow
you to make much quicker progress without a lot of stumbling blocks.

If you run into problems please join us on [stm32-rs channel on Element.io],
there're always friendly people around (including me 😅) willing to help you
out.

# The finale

I ordered these boards a while ago and let them sit around without paying too
much attention however that was a big mistake. After some people mentioned them
I decided to take a look and have to say I am impressed by the quality and the
bang for the buck. Especially in comparison to the thing called a "BluePill"
which exists in a ton of varieties with subtle (and not so subtle!) differences
and generally bad quality and a lot of problems. These boards not only offer a
better build quality, USB-C, built-in USB bootloader, more speed, more FLASH,
more RAM and more peripherals but a much better user experience, too, while
still being really affordable.

Stay tuned for more content using this board, I'll definitely use them every
now and then for various purposes.

[STM32F4x1 MiniF4]: https://github.com/WeActStudio/WeActStudio.MiniSTM32F4x1
[WeAct Studio]: https://github.com/WeActStudio/
[CMSIS-DAP]: https://github.com/WeActStudio/WeActStudio.MiniSTM32F4x1/blob/master/SDK/CMSIS-DAP/CMSIS-DAP_WeActStudio.hex
[purchasing sources]: https://github.com/WeActStudio/WeActStudio.MiniSTM32F4x1#legitimate-purchase-links-as-well-as-pirated-links
[probe-rs]: https://probe.rs
[stm32f4xx-hal]: https://github.com/stm32-rs/stm32f4xx-hal
[stm32-rs channel on Element.io]: https://matrix.to/#/!WqsLCItsZbJGhRjnxP:matrix.org
