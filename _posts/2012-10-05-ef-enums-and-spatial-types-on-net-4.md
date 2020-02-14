---
layout: post
title: EF enums and spatial types on .NET 4
date: 2012-10-05 16:08
author: ajcvickers
comments: true
categories: [EF6, Entity Framework, Enums, Open Source, Spatial]
---
In EF5 some features (such as enums and spatial types) are only available when targeting .NET 4.5. This is because EF5 relies on core EF libraries in the .NET Framework. With EF6 this changes because the core EF code is now included in EntityFramework.dll and recent commits to the code make all EF features (except async) available on .NET 4. This includes enums, spatial support, TVFs, auto-compiled queries and other EF perf improvements, together with many smaller changes/bug fixes in the core code.<!--more-->
<h2>How do I get it?</h2>
At the time of writing EF6 is still being developed but you can easily get prerelease builds in two ways:
<ul>
	<li>Nightly builds are available as a NuGet feed from <a href="http://www.myget.org/gallery/aspnetwebstacknightly">myget.org</a>. It’s really easy to <a href="http://entityframework.codeplex.com/wikipage?title=Nightly%20Builds">set up a package source</a> in Visual Studio to get these builds and they are also fully signed.</li>
	<li>Set up a build machine and <a href="http://entityframework.codeplex.com/">build EF6 yourself</a>. Use this approach if you want to browse or modify the source or make contributions back to EF.</li>
</ul>
<h2>What else do I need to know?</h2>
There are a few things you should keep in mind when using EF6:
<ul>
	<li>EF6 is still being actively developed and we will add/remove/change behavior as we iterate on the development. That being said, the nightly builds are generally very usable for non-production environments, and if you find things that don’t work then please let us know!</li>
	<li>The move out of the .NET Framework necessitated some breaking changes. Be sure to read <a href="http://entityframework.codeplex.com/wikipage?title=Updating%20Applications%20to%20use%20EF6">this post</a> on updating EF applications to work with EF6.</li>
	<li>EF6 doesn’t yet work in partial trust environments.</li>
	<li>The only providers currently available for EF6 are for SQL Server and SQL Server Compact Edition.</li>
	<li>Task-based async support still requires .NET 4.5. This is because it relies on low-level support in the System.Data assembly.</li>
</ul>
