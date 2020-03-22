---
layout: default
title: "Moving to Linux - Part 3: Installation and day-to-day use"
date: 2020-03-22 07:30
day: 22nd
month: March
year: 2020
author: ajcvickers
permalink: 2020/03/07/movingtolinux3/
---

# Moving to Linux

<div class=big-image>
<a href="/assets/logo-linux.png"><img src="/assets/logo-linux.png" alt="Linux!" width=40% /></a>
</div>

# Part 3: Installation and day-to-day use

A few weeks ago, I decided to [move from Windows to Linux as my primary development platform](https://twitter.com/ajcvickers/status/1224736879683072003).
These posts are about my experience.

* Part 1: [Background and first impressions](/2020/03/07/movingtolinux1/)
* Part 2: [My life in operating systems](/2020/03/07/movingtolinux2/)
* Part 3: Installation and day-to-day use
* Part 4: [Conclusions](/2020/03/07/movingtolinux4/)

## Installation

Overall, installation went very smoothly.
I had only minor issues with getting my corporate machine to boot from USB, and with my NVIDIA graphics card.

### Secure UEFI is a pain

I first installed Ubuntu onto my main Microsoft dev box, which previously booted to Windows 10.
It was a major pain messing with the [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) to get it to boot from the [Ubuntu USB stick](https://ubuntu.com/tutorials/tutorial-create-a-usb-stick-on-windows#1-overview).
My home machines didn't have these problems, and booting from the USB stick was very easy.

### Third-party drivers

Part of the installation process asks you if you want to install third-party drivers.

<div class=big-image>
<a href="/assets/trickquestion.png"><img src="/assets/trickquestion.png" alt="Do you want your computer to work?" width=70% /></a>
</div>

This option is unchecked by default.
Since I didn't fully grok the implications, I went with the default.
That was a mistake.

The result was a very unstable system with frequent freezes.
The reason being that I have a modern NVIDIA graphics card, and multiple hi-res monitors.
The open-source drivers simply don't work for this; the propriety drivers are required.

To me, this is an example of old-school Linux thinking hampering a consumer product.
I get the open source philosophy involved here, and I understand the advantage of being 100% open source.
But in this case it's much more important to get a working, stable system by default.
When things crash, people give up very quickly.

On the positive side, it was very easy to fix.
Just hit the Windows Key and type "drivers".

<div class=big-image>
<a href="/assets/drivers.png"><img src="/assets/drivers.png" alt="Additional Drivers" width=70% /></a>
</div>

I didn't make this mistake on my second machine.

## The Windows Key

In [_My life in operating systems_](/2020/03/07/movingtolinux2/) I talked about how the Windows Key doesn't work very well for me on Windows anymore.
To recap, the main problems are:
* Hitting the Windows Key and then immediately typing sometimes misses the first few letters typed, especially in a remote desktop.
* I frequently no longer find what I'm looking for even when it captures what I'm typing correctly.

So it was fantastic to find that this Windows Key muscle memory works perfectly on Ubuntu!
It works like it used to on Windows--I just type and more often than not I find what I'm looking for.
For example, see above for how I used it to install additional drivers by intuitively hitting Windows and typing "drivers".

<div class=big-image>
<a href="/assets/sittingontheshouldersofgiants.png"><img src="/assets/sittingontheshouldersofgiants.png" alt="Penguin Key" width=40% /></a>
</div>

Most of my other muscle memories for keyboard shortcuts also work on Ubuntu.
For example, I've been using Alt-Tab to switch applications without even thinking about it.
This really helps when coming from Windows.

## Applications

In the [old days](/2020/03/07/movingtolinux2/), finding and installing applications on Linux was a major pain.
I certainly don't want to be compiling myself and resolving dependencies just to install everyday apps.
Thankfully, things are much better now.
Ubuntu makes installing apps easy, as well as keeping them up-to-date.

<div class=big-image>
<a href="/assets/ubuntuapps.png"><img src="/assets/ubuntuapps.png" alt="Ubuntu Applications" width=70% /></a>
</div>

These are the applications I've been using day-to-day:
* [Chrome](https://www.google.com/chrome/)<sup><span style="font-size: x-small">([1](#footnote-1))</span></sup>
* [Dropbox](https://www.dropbox.com/)
* Files<sup><span style="font-size: x-small">([2](#footnote-2))</span></sup>
* GNOME Screenshot
* GNOME System monitor (to watch my cores working<sup><span style="font-size: x-small">([3](#footnote-3))</span></sup>)
* GNOME Terminal
* GNU Image Manipulation Program ([GIMP](https://www.gimp.org/); for all images in these posts)
* [Remmina](https://remmina.org/) (for remote desktop to Windows)
* JetBrains [Rider](https://www.jetbrains.com/rider/)
* SimpleScreenRecorder (for capturing screenshot videos<sup><span style="font-size: x-small">([3](#footnote-3))</span></sup>)
* [Spotify](https://www.spotify.com/us/download/linux/)
* Microsoft [Teams](https://teams.microsoft.com/downloads)
* [Visual Studio Code](https://code.visualstudio.com/)

I wasn't expecting to find Dopbox or Spotify<sup><span style="font-size: x-small">([4](#footnote-4))</span></sup>, so this was a very nice surprise.
Also, the built-in screen capture works great, and GIMP is now much more complete and easy to use than it used to be.<sup><span style="font-size: x-small">([5](#footnote-5))</span></sup>

I have also used [LibraOffice](https://www.libreoffice.org/) for the few times I need this kind of office application.
It was fine.<sup><span style="font-size: x-small">([6](#footnote-6))</span></sup>

### Jekyll

[Jekyll](https://jekyllrb.com/) is where things started to go a bit old-school for me.
I ended up compiling the tools I needed locally.
And managing version conflicts.
As I mentioned above, I don't want to do this.

Now I realize that I likely didn't find the correct instructions, or misunderstood something, and that Jekyll installation could have been much smoother.
Nevertheless, for me, it was a pain.

## High DPI and multiple monitors

I use multiple monitors with different resolutions and orientations at both home and work.

<div class=big-image>
<a href="/assets/toomany.jpg"><img src="/assets/toomany.jpg" alt="Enough monitors?" width=70% /></a>
</div>
 
Windows has put a lot of effort into support for this and it now works very well.
At least by default, Ubuntu with GNOME is not nearly as good.
For example, Ubuntu tends to open new applications with strange sizes and locations.

That being said, a little bit of monitor reorganization got me to a place where I was happy.
I'm also hoping there is better support available in Linux if I go outside the Ubuntu defaults.

## .NET development

### .NET Core SDK

As covered in [part 1](/2020/03/07/movingtolinux1/), I do most of my day-to-day development using C# in .NET Core.

The [.NET Core SDK](https://dotnet.microsoft.com/download) was trivial to install on Ubuntu.
Nothing else really needs to be said; I installed it; it worked.

<div class=big-image>
<a href="/assets/dotnetlinux.png"><img src="/assets/dotnetlinux.png" alt=".NET Core on Linux" width=85% /></a>
</div>
 
Likewise, installing [SQL Server Developer Edition for Linux](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-setup?view=sql-server-ver15) was also very easy.
In fact, it was much easier than that last time I tried to go through the torturous installation process on Windows.
There is no LocalDb on Linux, but I don't miss it at all.
Developer Edition works better for what I do.

### Rider

As I have mentioned several times in these posts, I am a big fan of IDEs and spend most of my development time using one.
On Windows, Visual Studio is the _de facto_ standard for C# development, but Visual Studio is not available for Linux.

[Visual Studio Code](https://code.visualstudio.com/) has great support for C# and runs very well on Linux.
It's also great as a general text editor, especially for markdown.
But it's not a fully-featured IDE in the vein of Visual Studio.

Luckily, JetBrains [Rider](https://www.jetbrains.com/rider/) is such an IDE for C#, and it runs extremely well on Linux.
This isn't a post about Rider, so I won't go into details, but I have been exceptionally happy with it.
I love programming, and Rider is a joy to use that makes my coding time both fun and productive.

<div class=big-image>
<a href="/assets/rider.png"><img src="/assets/rider.png" alt="Rider on Linux" width=85% /></a>
</div>

I used the [JetBrains Toolbox](https://www.jetbrains.com/toolbox-app/) to install Rider and it has been happily keeping it up-to-date for me.  

### Git

Git runs just fine on Linux, as is to be expected given its [origins](https://en.wikipedia.org/wiki/Git).
I'm fairly good at git on the command line, but I would also sometimes use [TortoiseGit](https://tortoisegit.org/) on Windows.
Lately, even on Windows, the built-in git support in Rider and VS Code has meant using TortoiseGit less.
I rarely feel the need to leave Rider to do anything with git these days.

I've also been happy with [GitKraken](https://www.gitkraken.com/) for the times where I needed a bit more power. 

## Outlook

Microsoft Outlook is kind of the elephant in the room.
As of now, I am still using remote desktop to my Windows machines when I want to use Outlook--which is every day at work.
I am planning to try the web application, which I have been assured is much better than it was years ago.

I'm very reluctant to use a third-party Outlook client.
Again, this may now work well, but I've been burned in the past.
Outlook is so vital to my day-to-day work at Microsoft that I'm very wary of anything messing this up!  

If you're lucky enough to have both Windows and Linux machines under your desk, then [Remmina](https://remmina.org/) makes remote access to Windows really nice and easy.
Just make sure to enable font smoothing in the Remmina options.

<div class=big-image>
<a href="/assets/outlookremote.png"><img src="/assets/outlookremote.png" alt="Outlook running in a remote desktop" width=85% /></a>
</div>

Running Windows in a virtual machine would be another option for Outlook if you don't have a spare Windows machine.

## Conclusion

Basically, my experience as a C# programmer on Ubuntu has been very good.
The [final post in this series](/2020/03/07/movingtolinux4/) summarizes my findings in more detail.

### Footnotes

<div style="font-size: small">

<a name="footnote-1"></a>
<sup>(1)</sup> I'm not a Firefox fan. :man_shrugging:
Edge for Linux? Yes please!

<a name="footnote-2"></a>
<sup>(2)</sup> I also installed [Midnight Commander](https://midnight-commander.org/) for old times sake.
It looks exactly the same as it always did!
Some thing change.
Some things stay the same. 

<a name="footnote-3"></a>
<sup>(3)</sup> For example, see [all my cores being used](https://twitter.com/ajcvickers/status/1227496521744166912). 

<a name="footnote-4"></a>
<sup>(4)</sup> Me: This is Spotify running on Linux on my new computer with an optical digital connection to the receiver and output to my subwoofer and Klipsch speakers!
Wife: That’s impressive, sweetie. But you know what would have been more impressive? If you had put these dishes up!
  
<a name="footnote-5"></a>
<sup>(5)</sup> Ten years ago I was a Photoshop user on Windows.
At some point it became too expensive to be worth it anymore.
Its really nice that GIMP has got to the point where I don't miss Photoshop.

<a name="footnote-6"></a>
<sup>(6)</sup> I'm not a Microsoft Office power user.
I use PowerPoint, Word, and Excel occasionally, but I doubt I have used any of the new features added in the last 10 to 15 years.
Outlook is a different matter.

</div>


