+++
categories = [ "electronics", "IoT" ]
date = "2016-04-16T10:33:41-04:00"
description = "Wherein I think about hooking up a Wii balance board to the Fitbit API..."
keywords = [ "Wii", "Raspberry Pi", "Bluetooth", "Fitbit" ]
title = "Wii Balance Board Fitbit logger - Part 1"

+++

There was a [pretty interesting post](https://blog.adafruit.com/2016/04/15/weight-tracking-with-a-humorous-text-messaging-scale-pi3-piday-raspberrypi-raspberry_pi/)
that I recently came across - basically a link to [`InitialState/smart-scale` on GitHub ](https://github.com/InitialState/smart-scale/wiki).

The basic idea is you take a Wii Balance Board (the one that came with Wii Fit),
and hook it up via Bluetooth to a Raspberry Pi 3 (or any other computer with
Bluetooth).

When I was reading the post, I tapped my wrist and realized my Fitbit was out of
battery power and I'd have to go charge it soon. This got me thinking...
What if I could use my Wii Balance Board to send weight updates to Fitbit
through their API?

### Some gaps...

So there's a few things I'll need to figure out:

1. What language do I want to write all this in?
  - The example's in Python, but I'd prefer to use Go, because...
2. How to authenticate with the Fitbit API?
  - They use OAuth2 (OAuth1 was deprecated and is no longer supported), and it
  seems I need to use the ["Implicit Grant" flow](https://dev.fitbit.com/docs/oauth2/#implicit-grant-flow)
3. I have zero knowledge of Bluetooth, so I'm sure there'll be some stumbling
  blocks along the way...
  - Can I keep my Raspberry Pi in the basement but have the balance board on the
  top floor?

I'm going to get started (vaguely), and will report back in part 2! 👋
