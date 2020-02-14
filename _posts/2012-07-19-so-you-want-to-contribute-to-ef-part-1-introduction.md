---
layout: post
title: So you want to contribute to EF? Part 1: Introduction
date: 2012-07-19 09:16
author: ajcvickers
comments: true
categories: [EF6, Entity Framework, OSS]
---
As you have probably heard <a href="http://blogs.msdn.com/b/adonet/archive/2012/07/19/entity-framework-and-open-source.aspx">the Entity Framework source code is now available under an open source licens</a>e. This means that the EF team are ready and excited to take your contributions.

The process for getting and building the EF code and for making contributions are documented on the <a href="http://entityframework.codeplex.com/documentation">EF CodePlex site</a>. This series does not cover that information again but instead gives some context to help you in working with the code. This is the same kind of information that you would get if you joined the EF team.<!--more-->
<h2>Series Contents</h2>
<h3>Part 1: Introduction</h3>
<ul>
	<li>This post—an introduction to and overview of the series</li>
	<li>Deciding what to contribute</li>
	<li>How to get additional help</li>
</ul>
<h3>Part 2: <a title="The code" href="http://blog.oneunicorn.com/2012/07/19/so-you-want-to-contribute-to-ef-part-2-the-code/">The code</a></h3>
<ul>
	<li>The high-level organization of the code into namespaces and assemblies</li>
</ul>
<h3>Part 3: <a title="Testing" href="http://blog.oneunicorn.com/2012/07/19/so-you-want-to-contribute-to-ef-part-3-testing/">Testing</a></h3>
<ul>
	<li>Expectations around testing when you make a contribution</li>
	<li>Types of test, test organization, and testing conventions</li>
	<li>Special considerations for changes to the core code</li>
</ul>
<h3>Part 4: <a title="Developer experience" href="http://blog.oneunicorn.com/2012/07/19/so-you-want-to-contribute-to-ef-part-4-developer-experience/">Developer experience</a></h3>
<ul>
	<li>Expectations around public API and behavior changes</li>
	<li>Breaking changes</li>
</ul>
<h3>Part 5: <a title="High-level architecture" href="http://blog.oneunicorn.com/2012/07/19/so-you-want-to-contribute-to-ef-part-5-high-level-architecture/">High-level architecture</a></h3>
<ul>
	<li>The Entity Data Model (EDM)</li>
	<li>High-level overview of EF sub-systems</li>
</ul>
<h2>Deciding what to contribute</h2>
If you want something to get started on, then the <a href="http://entityframework.codeplex.com/workitem/list/basic">EF issue tracker on CodePlex</a> contains bugs that need fixing. Fixing bugs is a good way to start getting an understanding of the code and the process of making a contribution.

It’s great if you want to tackle something bigger than a bug and the EF team is fully supportive of this. If you do want to tackle something bigger than a bug then you should <a href="http://entityframework.codeplex.com/discussions">start a discussion on CodePlex</a> so that we can collaborate on the design. It’s also a good idea to read part 4 of this series to ensure whatever you work on meets expectations for the developer experience.
<h2>Additional help</h2>
The EF team holds a weekly “design meeting” in which we discuss design and other aspects of writing and maintaining the EF code. We have started putting <a href="http://entityframework.codeplex.com/wikipage?title=Design%20Meeting%20Notes">the notes from this meeting onto our CodePlex site</a> and you might find these notes useful in gaining context for an area that you want to change.

The EF CodePlex site also contains other <a href="http://entityframework.codeplex.com/documentation">documentation</a> and a <a href="http://entityframework.codeplex.com/discussions">discussion</a> area in which the team or other members of the community may be able to help you.

Feel free to contact me (see below) or others on the team if you need help. We want you to make contributions and we will do what we can to make you successful in doing so.
<h2>Future posts…</h2>
I have started out with five parts to this series but I’m willing to write more if people find it useful. If there is something that you would like to get more information about then let me know by leaving a comment, sending me a tweet, or emailing me (avickers) at Microsoft.

Have fun and I look forward to seeing your contributions!
