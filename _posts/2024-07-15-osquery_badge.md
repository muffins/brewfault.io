---
layout: post
title: An osquery SAO Badge
comments: true
---

I spent a bit of time at the start of this year learning KiCAD and decided
I wanted to make a badge for DEFCON. It's super common to see [SAO](https://hackaday.io/project/175182-simple-add-ons-sao)s
floating around, so I figured I'd try and make whatever badge I produced compatible.
As I _used_ to be heavily affiliated with the [osquery](https://github.com/osquery/osquery)
project I happened to have a bunch of the logo SVGs laying around, and figured 
that'd make a good "babys first" badge. This post has (hopefully) everything one
needs to assemble the badge itself. If you're interested in making your own, or
reflashing the firmaware, check out the [Github here](https://github.com/muffins/osquery-badge).

## Bill of Materials

You'll need the following things (these should all be in the baggy if you bought
a kit at DEFCON)

1. 1 x ATTiny85
2. 8 x NeoPixel Mini 3535
3. 1 x 470 Ohm Resistor
4. 1 x Switch
5. (Optional) 1 JST Battery Connector
6. (Optional) 1 SAO Connector

## Assembly Instructions

You should have roughly the following (**Note** the ATTiny85 is missing from
this picture.)

![Badge Bundle](https://brewfault.io/images/osquery-badge/1.jpg)

### Solder the resistor

First solder on the resistor, polarity doesn't matter on resistors so it can be
soldered on either way.

![Badge Bundle](https://brewfault.io/images/osquery-badge/2.jpg)

### Solder on the NeoPixels

This is arguably the hardest part. **Take your time**, nice and slow. Many folks
recommend using lower soldering heat to not fry the LEDs. Your kit may not come
with extras so go slow and steady.

**Notes**:

- **The positioning of the LED matters!** The white marked corner on the LED
  should "point" to the angle of the L on the board. See the image below

![LED Positioning](https://brewfault.io/images/osquery-badge/3.jpg)

- I recommend tinning the pad by the angle on each LED position first, then go
    back through and attach each one.

![LED Pads Tinned](https://brewfault.io/images/osquery-badge/4.jpg)

### Solder on the Switch and JST connector (if using)

The kit comes with a switch and a JST connector, and maybe even a LiPol battery
if I had any left. Solder the switch onto the board, and if your battery needs
a JST connector solder this on as well. If your battery is just a red and black
wire, solder the wire to the pad closest to the top of the board as so:

![Positive Battery Pad](https://brewfault.io/images/osquery-badge/5.jpg)

Your board should now look like this

![Almost there!](https://brewfault.io/images/osquery-badge/6.jpg)

### Attach the ATTiny85

If you have socketed header you can consider using this, if you're planning
to reflash the chip. Otherwise, solder the chip on the board and attach the
battery.

![Done!](https://brewfault.io/images/osquery-badge/7.jpg)
