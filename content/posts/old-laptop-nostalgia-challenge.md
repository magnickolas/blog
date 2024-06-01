---
title: "Old Laptop: Nostalgia Challenge"
subtitle: ""
date: 2024-05-31T13:00:00+01:00
draft: false
description: "A small story of buying a used laptop and reviving it."

tags: [retro, windowsXP, hackintosh]
categories: [stories]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: "images/old-laptop-nostalgia-challenge/x230.webp"

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

## Mind wandering

For some time I had an idea of getting some old PC/laptop, the motivation being: play old games, revisit the operating system of my childhood, put the machine in order, mess with it as I please. Basically, the only specific requirement was the presence of a CPU from Ivy Bridge generation for a full support of Windows XP (yeah I am not that old so it makes me nostalgic). It didn't have to have a powerful CPU with a lot of cores or a GPU, as games from 1990s-2000s are fairly fine to run with Intel Iris or whatever you pull.

In the end of March 2024, I have taken a look at what is available in a one of refubrished electronic devices online stores and came across a used Lenovo ThinkPad X230 from 2013 with Intel Core i5-3320M (the IvyBridge generation as requested). It cost starting from €59 (~$64) when I opted out of the additional RAM/storage, a pre-installed system, etc. I really liked the prospect of getting a hopefully functional laptop for that price, especially considering that just the charging adapter or the battery alone are being sold for nearly half of that each, and they are both included. So I went for it and made an order right away.

## You get what you deserve

When I unpacked the package, I was greeted by a bit sticky laptop with a chipped corner and a missing trackpoint. The charging adapter came with a mismatched power cord. Quite an auspicious start to my journey, I would say <img src="/images/emoji/harold.png" height=20px>. I went to a local store and bought a power cord for about €3.

As soon as I turned the laptop on, it gave me a message about the battery being unsupported, and the laptop really was just immediately turning off without the charger connected. At this point I got a bit frustrated, yet had the determination to persevere and power to fight. It turned out that this issue arises due a hard-coded whitelist in the BIOS, which could be bypassed by [patching the embedded controller part][bios-patch]. I booted a live Debian image and followed the instructions, which led me to success.

Now that I fixed the immediate issues and had a full-fledged laptop (at least at the first glance), I could clean it up and begin doing something with it. First thing, I tried installing Windows XP, which is already not that straightforward as after creating a bootable USB drive using Rufus, I encountered an error about a missing End-User License Agreement. A quick search led me to using [WinSetupFromUSB][win-setup-from-usb] for preparing a USB drive instead, and that resulted in success. Later on, I also found a guide on [skipster-1337's site][skip-blog] about this and more.

<img src="/images/old-laptop-nostalgia-challenge/x230.webp" width="100%">

## Upgrading

<img src="/images/old-laptop-nostalgia-challenge/cybermen.webp" width="100%">

As the specs of the laptop were not top-notch (4 GB RAM + 128 GB SSD), I decided to order some parts and accessories for it:

- a trackpoint rubber cap since it was missing
- additional 4 GB of DDR3 RAM, making total 8 GB
- 256 GB mSATA SSD drive (I replaced the WWAN module with it)
- a power adapter connector converter from USB Type C for portability
- an HDMI to Mini DisplayPort converter to be able to connect an external monitor.

This "upgrade kit" cost me €37.8, and together with the price of the laptop and the power cord, the total expense is €99.8 (however the charger is not being used anymore after getting the converter from Type C).

## Next challenge

Curious what else there was making this laptop special, I discovered that people had been successfully installing and using macOS on it for many years. As a child, I was captivated by the macOS interface and researching ways to transform some of my devices into a [Hackintosh][hackintosh]. Perhaps now, it was finally time to get closure on this as a professional software engineer.

There is a [repository of banhbaoxamlan][x230-hackintosh-repo] containing prepared drivers, kexts, configurations, and other necessary files, which seemed like at least a good starting point. After reading through the documentation and a few attempts, I was able to see the installation UI of macOS BigSur and then get a somewhat working OS. Bluetooth didn't seem to work, and Wi-Fi worked only with [HeliPort][heliport-repo]. However, since BigSur is no longer maintained by Apple, installing packages was a pain in the ass, and I was not ready to wait billions of years for all the dependencies from Homebrew to be built on an old dual-core CPU.

My next step was to try to install macOS Monterey. This was supposed to be more challenging, as it doesn't natively support the integrated graphics of this laptop. However, there is an [OpenCore Legacy Patcher][oclp-repo] that fixes it. In the end, the installation process was not that different from the previous one. Wi-Fi worked "out of the box" with the system's native control panel, and I was able to get Bluetooth working. The operating system itself runs quite smoothly, especially for a laptop from 2013 that is not natively supported.

Now, I can conclude that I have got myself a HackBook for under €100 with:

- 256 GB + 128 GB SSDs
- 8 GB RAM
- dual-core CPU
- and distinguishing features such as a 180° hinge, ThinkLight and the socially adored trackpoint.

## Side quest

The last thing I want to mention is the [Computrace][computrace] security feature, which is basically a laptop tracking software designed for anti-theft purposes. It only works with Windows, and for the end users of decommissioned equipment, this is just another hurdle if it has not been disabled.

In my case, I discovered that the feature was enabled in the BIOS, and since it couldn't be disabled manually, I reached out to the tech support team at Absolute. They informed me that Computrace was already disabled for my laptop and that their software doesn't support Windows 7 anymore (which I had installed to try to synchronise changes), but only Windows 8 and newer. After installing Windows 10, disabling the built-in firewall and rebooting twice, I was finally able to switch it off.

[bios-patch]: https://github.com/hamishcoleman/thinkpad-ec
[win-setup-from-usb]: http://www.winsetupfromusb.com
[skip-blog]: https://skipster1337.github.io/posts/xp-info.html
[hackintosh]: https://en.wikipedia.org/wiki/Hackintosh
[x230-hackintosh-repo]: https://github.com/banhbaoxamlan/X230-Hackintosh
[heliport-repo]: https://github.com/OpenIntelWireless/HeliPort
[oclp-repo]: https://github.com/dortania/OpenCore-Legacy-Patcher
[computrace]: https://en.wikipedia.org/wiki/Absolute_Home_%26_Office
