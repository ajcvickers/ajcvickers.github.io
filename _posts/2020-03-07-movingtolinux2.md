---
layout: default
title: "Moving to Linux - Part 2: My life in operating systems"
date: 2020-03-22 07:30
day: 22nd
month: March
year: 2020
author: ajcvickers
permalink: 2020/03/07/movingtolinux2/
---

# Moving to Linux

<div class=big-image>
<a href="/assets/logo-linux.png"><img src="/assets/logo-linux.png" alt="Linux!" width=40% /></a>
</div>

# Part 2: My life in operating systems

A few weeks ago, I decided to [move from Windows to Linux as my primary development platform](https://twitter.com/ajcvickers/status/1224736879683072003).
These posts are about my experience.

* Part 1: [Background and first impressions](/2020/03/07/movingtolinux1/)
* Part 2: My life in operating systems
* Part 3: [Installation and day-to-day use](/2020/03/07/movingtolinux3/)
* Part 4: [Conclusions](/2020/03/07/movingtolinux4/)

## You may want to skip this post

There's nothing in this post specifically about my current experience on Linux.
Instead, this is some deep background on where I am coming from.

I'm mainly writing this because my personal experience with operating systems in the past will necessarily color my experiences this time.
For example, I _have_ used Linux in the past; I'm not a total newbie.

Also, I like reminiscing about old computer stuff, and maybe some others will find this interesting. :smile_cat:

## Sharp MZ-80K

My first computer was a [Sharp MZ-80K](https://en.wikipedia.org/wiki/Sharp_MZ)<sup><span style="font-size: x-small">([1](#footnote-1))</span></sup>

<div class=big-image>
<a href="/assets/mz80k.jpg"><img src="/assets/mz80k.jpg" alt="Sharp MZ-80K" width=50% /></a>
</div>

When turned on it presented a very simple loader, with a prompt.

```
** MONITOR SP-100X **
* 
```

That's it.
There was no BASIC in ROM;
you loaded it off a cassette tape.
_Everything_ had to be loaded from tape.
You [ZX-Spectrum](https://en.wikipedia.org/wiki/ZX_Spectrum) owners with your fancy BASIC in ROM didn't know how good you had it!

The operating system experience could only get better from here! (But believe me, I worshipped that MZ-80K because...I had a computer!)

## Amiga

I used a few other computers after the MZ-80K, but the first with what I would consider a real, general purpose operating system was the [Amiga A500](https://en.wikipedia.org/wiki/Amiga_500), which I got my hands on in 1990.

<div class=big-image>
<a href="/assets/a500.jpg"><img src="/assets/a500.jpg" alt="Amiga A500. Image © Bill Bertram 2006." width=50% /></a>
</div>

The GUI shell and file manager ([Workbench](https://en.wikipedia.org/wiki/Workbench_(AmigaOS))) were built on a native windowing system called [Intuition](https://en.wikipedia.org/wiki/Intuition_(Amiga)).
These were both ahead of their time, especially for a home operating system.

The Amiga also supported [full, preemptive multitasking](https://en.wikipedia.org/wiki/Preemption_(computing)#PREEMPTIVE).
As a geek, this was very cool.
However, looking back I don't think it was actually very important.
Windows caught up partially in 95, and completely in NT.

## Enter Windows

The Amiga was a hard act to follow in many ways.
Pretty much every aspect was better than early PC clones.
I used Windows [2.x](https://en.wikipedia.org/wiki/Windows_2.1x) and [3.x](https://en.wikipedia.org/wiki/Windows_3.1x), but found both very lacking.

But Amiga as a platform didn't last.
In 1996 I finally gave up saving for an [Amiga A4000](https://en.wikipedia.org/wiki/Amiga_4000) and switched to a PC clone.
[Windows 95](https://en.wikipedia.org/wiki/Windows_95) made this possible.

<div class=big-image>
<a href="/assets/win95.png"><img src="/assets/win95.png" alt="Windows 95 ©Microsoft" width=50% /></a>
</div>

A lot of nasty things have been written about Windows 95.
But I found it to be a very good operating system.
Most importantly, it was easy and pleasurable to use, at least compared to the competition at the time.

And yes, I include pre OS-X Mac operating systems in this.
Coming from an Amiga background, the Mac OS just never seemed that good.
Macs were also expensive.<sup><span style="font-size: x-small">([2](#footnote-2))</span></sup>
Windows 95 copied a lot of ideas from other places, but it did what Microsoft did best--package them into a consumer product people could afford and use.

## Windows NT

I recently finished reading [_Show Stopper!: The Breakneck Race to Create Windows NT and the Next Generation at Microsoft_](https://www.amazon.com/Show-Stopper-Breakneck-Generation-Microsoft/dp/0029356717).
What a great book!<sup><span style="font-size: x-small">([3](#footnote-3))</span></sup>
NT evolved out of ideas that Dave Cutler and others brought with them from the world of DEC workstations.
By the release of [Window NT 4.0](https://en.wikipedia.org/wiki/Windows_NT_4.0), it had become a very successful melding of workstation concepts to PC hardware.

<div class=big-image>
<a href="/assets/nt4.png"><img src="/assets/nt4.png" alt="Windows NT 4.0 ©Microsoft" width=50% /></a>
</div>

This continued with [Windows 2000](https://en.wikipedia.org/wiki/Windows_2000) (NT 5.0), [Windows XP](https://en.wikipedia.org/wiki/Windows_XP) (NT 5.1), and finally<sup><span style="font-size: x-small">([4](#footnote-4))</span></sup> [Windows 7](https://en.wikipedia.org/wiki/Windows_7) (NT 6.1).
What a great line of operating systems!

## Enter Linux

Even though it was first released in 1991, Linux was really starting to become popular at about the same time as Windows NT.
In 1997, I bought one of the InfoMagic CD packs containing various distros and other good stuff.

<div class=big-image>
<a href="/assets/linuxoncd.jpg"><img src="/assets/linuxoncd.jpg" alt="InfoMagic Linux CDs" width=50% /></a>
</div>

It was fun to play with and learn, but the ease of use, the applications available, and the desktop experience was nowhere near what NT was offering.
Even as a programmer, I found it very frustrating to have resolve dependencies and compile things before I could use them.
That's just not something I want to bother with, even if it is technically superior in some situations.

Also, the lack of a good IDE (such as [Visual Studio 6.0](https://en.wikipedia.org/wiki/Microsoft_Visual_Studio#6.0_(1998))) was really limiting.

So I mostly used Linux as a server OS in the late '90s and early 2000s.
For example, I used an old [IBM PS/2](https://en.wikipedia.org/wiki/IBM_Personal_System/2) as a router for sharing an Internet connection with multiple PCs on home Ethernet.<sup><span style="font-size: x-small">([5](#footnote-5))</span></sup>

## Tektronix

I joined [Tektronix](https://en.wikipedia.org/wiki/Tektronix) in 2006 to work on the design software for oscilloscopes.
Tektronix was full of hard-cord Unix guys.<sup><span style="font-size: x-small">([6](#footnote-6))</span></sup>

All my development at Tektronix was on Linux.
It was a terrible experience.
We were forced to use really old (even for then) Red Hat Enterprise Linux, and it crashed all the time.
And by crash, I mean the whole machine would just randomly restart with no warning.

<div class=big-image>
<a href="/assets/rhel4.png"><img src="/assets/rhel4.png" alt="Red Hat Enterprise Linux 4." width=50% /></a>
</div>

Also, the lack of a good IDE was still really limiting.
(See the patten here!)
By this time [Eclipse](https://en.wikipedia.org/wiki/Eclipse_(software)) was available, and it was my favorite IDE for Java.
But for C++ on Linux it just wasn't great.

Even the hardcore Unix guys had Windows machines for everything other than coding and circuit design.
I was not at all sad to leave Linux behind when I left Tektronix to join Microsoft.

## Windows 10

[Windows 8](https://en.wikipedia.org/wiki/Windows_8) and [8.1](https://en.wikipedia.org/wiki/Windows_8.1) are best left in the Vista bucket, but [Windows 10](https://en.wikipedia.org/wiki/Windows_10) is much better.
It is, of course, the latest incarnation of NT, so it has good heritage.
The push to improve fundamentals has made Windows faster, smaller, and more secure.
Also, features like [Windows Hello](https://en.wikipedia.org/wiki/Windows_10#System_security) are very nice to use.

<div class=big-image>
<a href="/assets/win10.png"><img src="/assets/win10.png" alt="Windows 10 ©Microsoft" width=50% /></a>
</div>

I'm happy on Windows 10, but some things still feel worse than they were in NT's golden years:

* Hitting the Windows Key followed by typing still sometimes misses the first few letters typed.
This is very annoying given that this is my primary way of running applications.
(This has got better, but is still a problem, especially when using a remote desktop.)

* Once I do start searching, Windows frequently can't find my applications and files anymore.
For example, I have one application installed where the only way I can run it is to manually find the .exe.
I've used this application (updating frequently) for years, and it's never had this issue on previous Windows releases.<sup><span style="font-size: x-small">([7](#footnote-7))</span></sup>

* Too often Windows 10 feels like using a mobile operating system on a desktop PC.
It works, but its clunky, not elegant.

## Summary

So that's a brief history of my life in operating systems.<sup><span style="font-size: x-small">([8](#footnote-8))</span></sup>
I work for Microsoft and I've been most recently using Windows 10, but that's only a small part of where I'm coming from in this move to try Linux again.

## Up next

[Part 3](/2020/03/07/movingtolinux3/) covers my experience installing Ubuntu Linux on two modern PCs.

Go back to [Part 1](/2020/03/07/movingtolinux1/) for general background and my first impressions of Ubuntu.

### Footnotes

<div style="font-size: small">

<a name="footnote-1"></a>
<sup>(1)</sup> Given to me by my grandad when he upgraded to a [BBC Model B](https://en.wikipedia.org/wiki/BBC_Micro).
There's a good story there I should tell sometime.

<a name="footnote-2"></a>
<sup>(2)</sup> Some things don't change.
The reason I have never developed on a Mac is because Apple hardware is just too expensive.

<a name="footnote-3"></a>
<sup>(3)</sup> If you can get past the terrible attempts to make technical concepts understandable to laymen.
Seriously, if you don't know what an operating system is, you're unlikely to be reading a book about Windows NT!

<a name="footnote-4"></a>
<sup>(4)</sup> [Windows Vista](https://en.wikipedia.org/wiki/Windows_Vista) is best left unmentioned.
When I started at Microsoft it was common to set up Windows Server as a desktop OS basically to avoid using Vista.

<a name="footnote-5"></a>
<sup>(5)</sup> For those interested, it was [10Base2 Ethernet](https://en.wikipedia.org/wiki/10BASE2) and we shared a [14,400 bits/s modem](https://en.wikipedia.org/wiki/Modem#Echo_cancellation,_9600_and_14,400) using [SLiRP](https://en.wikipedia.org/wiki/Slirp) to my Iowa State University shell account.
Very slow, but very reliable.
We could stay connected for a full weekend.
Much to the annoyance of my family who could never get hold of me by phone!

<a name="footnote-6"></a>
<sup>(6)</sup> Back in the '80s, Tektronix was also a hotbed for Smalltalk and IDEs, and one of the birthplaces of agile development.
I loved hearing stories of Kent Beck and Ward Cunningham pair-programming, even though I was there too late to meet them.
I had read [_Extreme Programming Explained_](https://www.amazon.com/Extreme-Programming-Explained-Embrace-Change/dp/0321278658) several years before, and it changed my life.
Its tag line of "Embrace Change" is just as relevant now as it was back then.

<a name="footnote-7"></a>
<sup>(7)</sup> I realize that the application is likely doing something "wrong".
That a common application can easily do something "wrong" and this doesn't get fixed is part of the problem.

<a name="footnote-8"></a>
<sup>(8)</sup> I have missed out all the embedded and mobile operating systems that I've used over the years, since they aren't relevant for desktop use.

</div>