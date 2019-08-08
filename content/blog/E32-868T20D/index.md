+++

date = 2019-08-06
title = "A closer look at the E32-868T20D LoRA Module with UART interface"
slug = "E32-868T20D"

+++

I've recently set up a gateway for [The Things Network] and to spice things up
a bit a was looking into possibilities of hooking up some microcontroller based
board to it. So it started to look into options how to make this happen.

<!-- more -->

As it turns out, Semtech is one of the biggest manufacturers of LoRaWAN capable
IP cores which are also used by a lot of manufacturers in one way or another.

So I was hoping I would find a [SX1276] based board somewhere in my
drawer but unfortunately I only found [RAK811] boards which are kind of
expensive boards which already come with a microcontroller implementing a
sophisticated LoRA/LoRAWAN stack requiring interfacing and a [E32-868T20D]
module from EByte which does something similar but in a much more [primitive
and mostly useless way][E32-868T20D manual].

This is what it looks like:

{{ resize_image(path="blog/E32-868T20D/board_whole.jpg", width=500, height=500, op="fit") }}

Since EByte claimed that this is module contains a SX176 based design paired
with premium components, I'd say we have a look under the hood and see whether
we can make it useful for us; shall we?`

## Under the hood

So with a bit of fiddling (and the help of SRA chip removal alloy) I removed
the metal shield and was greeted with these components on the board:

{{ resize_image(path="blog/E32-868T20D/board.jpg", width=500, height=500, op="fit") }}

From left to right we can see:

* The SMA connector
* A discrete filter and impedance matching network (look at those beautiful wirewound inductors!)
* The [SX1276]
* A crystal
* A [STM8L151G] microcontroller
* Two 3V3 voltage regulators
* Various decoupling caps and resistors
* The 2.54mm pin headers to hook up to a MCU or USB to serial interface

A few noteworthy things here: First of all, it's a really beautiful, well
structured and clean design, the SX1276 and the antenna interface are textbook
and use really high quality components. For whatever reason EByte has chosen
a rather powerful and expensive [STM8L151G] microcontroller for such a dumb
job. The reason for the two (and different brand!) 3V3 regulators is that one
is for the MCU and the other one for the SX1276 which is actually conntrolled
by the microcontroller and only turned on when communication is required.

## Reversing the module

The module is a bit annoying in that the serial interface is rather useless. It
only supports a P2P LoRA mode so it can only talk to another module of its kind
-- pretty much. So my idea was to figure out what's happending under the hood,
removing the serial interface and wire up the SPI interface of the SX1276 for
direct use. 

So here's the important bits marked up:

{{ resize_image(path="blog/E32-868T20D/board_reversed.jpg", width=500, height=500, op="fit") }}

So basically we should be able to connect the `SCK`, `MISO` and `MISO` pins
from the SX1276 directly to the `AUX`, `TXD` and `RXD` pins, permanently turn
on the second LDO and everything should work just fine.

## Mod the module

So here's the modded version:

{{ resize_image(path="blog/E32-868T20D/board_mod.jpg", width=500, height=500, op="fit") }}

I decided to remove the LDO for the STM8L151G as there's really no point in
having a second one which just adds to the noise and draws current all the
time. Then I'vve used shame wire to cross from the signals coming over from the
SX1276 to run them directly into the previous output pins of the STM8L151G. 
The `RESET` pin I simply bridged over to `M0`, for this the former LDO line had
to go, otherwise that'd be connected to the LDO which is probably not good
for power sequencing reasons. Last but not least the LDO was set to be
permanently on.

Of course everything was tested for shorts and proper voltages/power
consumption so until I can test it via software I'm going to assume it was
succssfully converted.

## Next steps

Of course now we'll need some proper Oxidization and turn to writing some Rust
software. I actually found two implementations, one from Rust Embedded WG
member Ryan Kurte [radio-sx127x] and one from Charles Wade [sx127x_lora]. So
all that there's left to do is try them out.

Exciting times. Until next time...

[E32-868T20D]: http://www.ebyte.com/en/product-view-news.aspx?id=132
[SX1276]: https://www.semtech.com/products/wireless-rf/lora-transceivers/SX1276
[STM8L151G]: https://www.st.com/resource/en/datasheet/stm8l151g6.pdf
[RAK811]: https://www.rakwireless.com/en/module/lora/RAK811
[The Things Network]: https://www.thethingsnetwork.org/
[E32-868T20D manual]: http://www.ebyte.com/en/downpdf.aspx?id=132
[radio-sx127x]: https://crates.io/crates/radio-sx127x
[sx127x_lora]: https://crates.io/crates/sx127x_lora

