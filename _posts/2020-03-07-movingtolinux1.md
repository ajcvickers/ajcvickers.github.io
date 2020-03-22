---
layout: default
title: "Moving to Linux - Part 1: Background and first impressions"
date: 2020-03-22 07:30
day: 22nd
month: March
year: 2020
author: ajcvickers
permalink: 2020/03/07/movingtolinux1/
---

# Moving to Linux

<div class=big-image>
<a href="/assets/logo-linux.png"><img src="/assets/logo-linux.png" alt="Linux!" width=40% /></a>
</div>

# Part 1: Background and first impressions

A few weeks ago, I decided to [move from Windows to Linux as my primary development platform](https://twitter.com/ajcvickers/status/1224736879683072003).
These posts are about my experience.

* Part 1: Background and first impressions
* Part 2: [My life in operating systems](/2020/03/07/movingtolinux2/)
* Part 3: [Installation and day-to-day use](/2020/03/07/movingtolinux3/)
* Part 4: [Conclusions](/2020/03/07/movingtolinux4/)

## Background

### Why?

I work for Microsoft on .NET Core. (However, the opinions expressed here are my own.)
Cross-platform support is a key part of [.NET Core](https://dotnet.microsoft.com/learn/dotnet/what-is-dotnet) and running Linux in the cloud is a key part of Azure.

<div class=big-image>
<a href="https://dotnet.microsoft.com/download/dotnet-core"><img src="/assets/dotneteverywhere.png" alt=".NET Core everywhere!" width=80% /></a>
</div>

So we have been targeting and testing against Linux for several years now.
But most .NET development still happens on Windows and Mac machines.

Using Linux for day-to-day development will help me get a better understanding of how Linux works.
In the same way that emersion in a culture is a good way to understand that culture, so emersion in an operating system is a good way to understand that operating system.
In particular, I'll get a better understanding of how .NET works in non-Windows environments.

### But why _now_?

But I've been here before--see [_My life in operating systems_](/2020/03/07/movingtolinux2/).

Beyond wanting to understand Linux better, two other things motivated me to pull the trigger now:
* I'm a big fan of full-featured IDEs. [Rider](https://www.jetbrains.com/rider/) brings this to Linux for .NET. I couldn't work on Linux without a good IDE.
* I'm less happy with Windows than I used to be. (More details in later posts.)

All that being said, I'm also doing this because it's a fun and interesting experience. :smile_cat:

### What, _exactly_?

It's important to note that I still use Windows.
I'm gradually using Windows less, but I'll always need to use it to test Windows-specific behavior in the same way as I previously used Linux to test Linux-specific behavior.
Also, Outlook is hard to replace.

I was advised Ubuntu was a good place to start, so I installed [Ubuntu 19.10](https://ubuntu.com/) natively onto my hardware.
I considered using [WSL2](https://docs.microsoft.com/en-us/windows/wsl/wsl2-index) or running Linux in a VM.
Those are both things that I have used and will continue to use going forward.
WSL2 in particular is very good.
However, for this experiment I wanted a pure Linux experience with nothing else getting in the way.

## First impressions

Getting [Ubuntu onto a USB stick](https://ubuntu.com/tutorials/tutorial-create-a-usb-stick-on-windows#1-overview) was trivially easy.<sup><span style="font-size: x-small">([1](#footnote-1))</span></sup>
This is what you need to install Ubuntu, but you can also just [boot into Ubuntu from the USB stick](https://ubuntu.com/tutorials/try-ubuntu-before-you-install#1-getting-started) without installing anything.<sup><span style="font-size: x-small">([2](#footnote-2))</span></sup>
I found this a great way to play with things safely before taking the plunge.

Part of my playing was to try to [build EF Core](https://github.com/dotnet/efcore/blob/master/docs/getting-and-building-the-code.md). I was successful in a matter of minutes.

<div class=big-image>
<a href="/assets/buildefcoreeasy.jpeg"><img src="/assets/buildefcoreeasy.jpeg" alt="Building EF Core without installing an OS" width=max /></a>
</div>

Remember this is _without installing the operating system_.

The most impressive part of this was the response I got when trying to use git.
Windows default:

```
C:\Users\Arthur>git
'git' is not recognized as an internal or external command, operable program or batch file.
C:\Users\Arthur>
```

Ubuntu default:

```
ubuntu@ubuntu:~$ git

Command 'git' not found, but can be installed with:

sudo apt install git

ubuntu@ubuntu:~$
```

And then following those instructions worked perfectly!

<div class=big-image>
<a href="/assets/gitinstall.png"><img src="/assets/gitinstall.png" alt="sudo apt install git" width=max /></a>
</div>

This may seem like a small thing, but it was very helpful to not have to leave the terminal just to find out how to install git.

## Up next...

[Part 2](/2020/03/07/movingtolinux2/) of the series is deep background on my life in operating systems.
Skip this unless you're interested in old computer stuff.

In [part 3](/2020/03/07/movingtolinux3/) I talk about my experience installing Linux.

### Footnotes

<div style="font-size: small">

<a name="footnote-1"></a>
<sup>(1)</sup> What else are USB sticks even for these days?

<a name="footnote-2"></a>
<sup>(2)</sup> The tricky part is getting your machine to boot from USB stick if it has secure [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) boot settings.
On both my home machines booting from the USB was easy because I wasn't doing any of this.
On my work machine it required a lot of messing with UEFI settings.

</div>