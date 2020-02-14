---
layout: post
title: So you want to contribute to EF? Part 4: Developer experience
date: 2012-07-19 09:39
author: ajcvickers
comments: true
categories: [EF6, Entity Framework, Open Source, OSS]
---
This is the fourth part of a <a href="http://blog.oneunicorn.com/2012/07/19/so-you-want-to-contribute-to-ef-part-1-introduction/">series</a> providing some background to those who may want make contributes to the Entity Framework. In this post I’ll talk a bit about maintaining a good experience for the developers who use EF.<!--more-->
<h2>The focus on developer experience</h2>
The current EF team is very focused on making EF enjoyable for developers to use. (It’s hard to always get this right but that doesn’t stop it being a priority.) This includes both the public APIs and the behavior. In particular, we spent a lot of time making DbContext, Code First, Migrations, and so on intuitive and easy to use. We spent a lot of time polishing the APIs, considering the concept count to get started, ensuring a good grow-up story from simple tasks to more complex scenarios, and so on.

The reason this is important if you are going to contribute to EF is that any API surface you add or behavior you change must maintain or improve the developer experience. What that means in concrete terms depends on the specific change being made, but, for example, a change that adds a new public member to DbContext is going to get a lot of scrutiny because keeping the API surface of DbContext small and clean is a priority.
<h3>Iterative polishing</h3>
<strong>Don’t be put off by this. </strong>We’re not going to just reject a contribution because it doesn’t match what we think the API or behavior should be. Instead we are likely to iterate on it with you to get the change made in a way that works for you and others and maintains a good experience. This will likely go both ways with us providing feedback to you and you providing feedback to us.

This iterative polishing is how the EF team has worked for some time now so it’s not something we are making up for external contributions but rather an extension of the way we work to involve the community.
<h2>Breaking changes</h2>
If a code change causes an application that worked before the change to stop working after the change, then that change is considered a <em>breaking change</em>. Breaking changes cause problems for people upgrading their applications to new versions of a framework like the EF. On the other hand, sometimes the old code is so flawed that a breaking change is necessary to keep moving things forward.
<h3>Types of breaking change</h3>
There are two main categories of breaking change: API breaks and behavior breaks. API breaks happen when the public surface changes in such a way that code that previously compiled against it will no longer compile. Behavior changes are more subtle—the code that uses the framework will still compile but not the framework behaves differently causing the application to break.
<h3>Assessing breaking changes</h3>
Any breaking change will get a lot of scrutiny to make sure the benefit of making the change is significantly greater than the pain caused by the break. In particular, the biggest risk is breaking an application in a way that the break is not recognized until the new version of the application is put into production.

API breaks are usually easier to assess for this because the change causes an obvious failure (compile break) and if the way to fix the compile break is also obvious then it is hard for the breaking change to ultimately cause a production application break. (I’m leaving in-place updates aside here since we will try very, very hard to avoid them.) That being said, an API break should still be avoided if at all reasonable since it sill causes pain to those needing to fix compile breaks.

Significant behavior breaks are very likely to be rejected. If a significant change in behavior is introduced then it should usually be either accessed through a new API or should only take effect when an explicit change to application configuration is also made—e.g. when a flag switches on the new behavior.
<h2>Summary</h2>
Any changes to the EF must strive to preserve a good developer experience, both in terms of API surface and behavior. This includes avoiding breaking changes where possible. In the next post I’ll take a very high-level look at the EF architecture.
