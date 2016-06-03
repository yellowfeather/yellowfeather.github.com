---
layout: post
published: true
title: 3D print job notifications in Slack with Octoprint
date: 2016-06-03 20:57
comments: true
categories: [lulzbot, octoprint, slack]
author: chrisr
---

I recently purchased a [Taz 6](https://www.lulzbot.com/store/printers/lulzbot-taz-6) 3D printer from
[Lulzbot](https://www.lulzbot.com), great printer BTW, and one of the things I wanted was a
notification when a print job completed. There is no way to connect the Taz 6 directly to a network, but a quick
search led me to [Octoprint](http://octoprint.org), and more specifically [OctoPi](https://github.com/guysoft/OctoPi).
[Octoprint](http://octoprint.org) gives full remote control and monitoring of your 3D printer with a 'snappy web
interface'. I had a spare Raspberry Pi, so opted for the OctoPi SD card image. Installation was a breeze, just follow
the [instructions](https://github.com/guysoft/OctoPi#how-to-use-it) to install OctoPi to an SD card and configure the
WiFi. Insert the SD card into the Pi, connect the Taz 6 via USB to the Pi and boot. Once the Pi has booted you can
access OctoPrint at [http://octopi.local](http://octopi.local) (or the IP address if your computer doesn't support Bonjour).

Next up you need to setup an [Incoming webhook](https://my.slack.com/services/new/incoming-webhook) with Slack. Choose
a channel to post to (or create a new channel), enter a Descriptive Label, Customize Name, and upload an image for
Customize Icon (for my setup I left the Descriptive Label blank, enetered 'Taz 6' for Customize Name, and uploaded the
[Lulzbot logo](https://www.lulzbot.com/sites/all/themes/lulzbot/build/img/svg/png/logo-small.png) for the Customize
Icon). Take a copy of the Webhook URL so that we can paste it into the Octoprint Slack plugin config later.

You can install the Octoprint Slack plugin from within Octoprint, go to the
[Plugin manager](https://github.com/foosel/OctoPrint/wiki/Plugin:-Plugin-Manager), hit the 'Get more' button, search
for the Slack plugin and install it. In the plugin configuration, paste in the Webhook URL, update the remaining config
as required (I left them as the defaults). Thats it, you should now be getting notifications in Slack whenever a
print job is started, completed, fails, or is paused, cancelled or resumed!

![Slack OctoPi notification](/img/octopi-slack-notification.jpg)
