---
layout: default
title: "Internal code in EF Core 1.1"
date: 2016-11-09 16:46
day: 9th
month: November
year: 2016
author: ajcvickers
permalink: 2016/11/09/internal-code-in-ef-core-1-1/
---

# EF Core 1.1
# Internal code

There is very little code in EF Core that is truely "internal". Instead, code that would traditionally have been internal has been made public but documented as internal. This post looks at what this means for application developers and how you can use this code responsibly.



<h3>History</h3>

When I started work on the Entity Framework team the prevailing philosophy was to make all code internal unless it was specifically part of the fixed-in-stone public API surface. Similarly, many classes were sealed, abstract base classes were preferred over interfaces, and methods were not made virtual. All of this was to ensure the most freedom for future development of the codebase without making "breaking changes". Not making breaking changes was important for code that shipped as part of a monolithic .NET Framework, especially when that framework was updated in-place.

<h3>Where we are now</h3>

Skip forward a few years to a place where code is shipped on NuGet and applications control which package versions to use and when to update to a new package. Not making breaking changes is still important, but there is now some freedom to break for the long term good of the product. However, putting aside minor breaks to public APIs, there is still a lot of code that we know will need to break as the product evolves. For example:

<ul>
<li>The low-level guts of the implementation</li>
<li>Features that are not fully yet fully baked even though they are used internally</li>
</ul>

This is the type of code that would have been historically made internal.

<h3>Why use this code?</h3>

There are situations where applications need access to this kind of code. For example, imagine there is a bug in some low-level internal service. You can file a bug on GitHub and even send a pull request with a fix. But there will still be some period of time before that fix makes it to a production release. What if, while you waited, you could implement a fix yourself and have it used by your app without having to re-build EF or do anything gnarly?

As another example, think back to the days of Code First conventions in EF 4.1. The conventions API was not fully baked, so even though it was used internally, it was not exposed publicly. This made it very hard to implement your own conventions or tweak the ones shipped with EF. Eventing is a bit like this in EF Core 1.1--there are some internal hooks, but no baked public surface.

<h3>The solution</h3>

The solution to these problems is to make all the EF Core code public. This means that you can, for example, extend from an internal service base class and implement a fix, then use the ReplaceService API to use the fixed service in your app. Likewise, for eventing, if you want to be notified when the state of an entity changes, you can register listeners with some internal services to do this.

<h3>The consequence</h3>

But here's the kicker. In order to move the codebase forward we still need to be able to change the low-level guts of EF, and improve the not-yet-baked APIs. Therefore, if your app makes use of this "internal" code, then it may break when you update to a new version of EF.

Usually these breaks should be minor--for example, a new parameter added to a method or a new member added to an internal service interface. Our goal is not to make it hard to upgrade to a new version, but rather to balance a few breaks when upgrading with an "internal" codebase that can evolve the way it needs to.

<h3>Drinking internal code responsibly</h3>

The first thing you need to know is when you are using internal code. Internal code is almost always in an internal namespace. For example, <code>Microsoft.EntityFrameworkCore.Internal</code>. In addition, all internal code is documented like this:

``` c#
/// <summary>
///     This API supports the Entity Framework Core infrastructure and is not intended to be used
///     directly from your code. This API may change or be removed in future releases.
/// </summary>
```

So once you know that the code is internal you can make a conscious choice to use that code and accept that when you update to a new version of EF, then you might have to make some updates to your code.

<h3>Let the EF team know if you like this...</h3>

There has been a fair amount of pressure internally in taking this approach. It comes with a big risk that customers will make use of internal code, possibly without realizing it, and will then complain loudly when their application breaks. It won't take much of this to trigger a reversal of the decision to make all code public. This could lead to all the code currently documented as internal being made truly internal. So if you think keeping this code public is a good idea, then it's probably worth letting the EF team know this. I am happy to make sure any comments made on this blog are seen by the team.
