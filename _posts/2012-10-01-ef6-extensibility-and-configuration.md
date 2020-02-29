---
layout: post
title: EF6 Extensibility and Configuration
date: 2012-10-01 14:03
author: ajcvickers
comments: true
categories: [EF6, Entity Framework, Extensibility, Open Source, OSS]
---
<h2>Introduction</h2>
I just published two posts to the Entity Framework CodePlex site that cover some changes we are making to EF6 to improve <a href="http://entityframework.codeplex.com/wikipage?title=EF%20Configuration%20and%20Extensibility">extensibility</a> and <a href="http://entityframework.codeplex.com/wikipage?title=Code-based%20Configuration">code-based configuration</a>. These posts provide conceptual overviews of the features with relevant design and implementation details. The idea is that people who are interested in where EF is going can read posts like this and provide feedback on the direction, design, and implementation. Providing this kind of feedback is an important way to contribute to EF without even having to write any code.
<h2>Overview and goals</h2>
EF has historically taken a rather ad-hoc approach to runtime configuration and extensibility. The changes for EF6 described in these posts are intended to replace this ad-hoc approach with some building blocks that will provide a common mechanism for configuration. In doing so they also allow EF to be more easily extended by extracting out services that can then be resolved to different implementations at runtime.

In particular, the changes have the following high-level goals:
<ul>
	<li>Provide a common mechanism and building blocks whereby aspects of existing EF functionality can be factored out or new functionality added in such a way that different implementations can be injected without the core EF code knowing about the specifics of these implementations.</li>
	<li>Provide a unified mechanism for EF code to access configuration regardless of whether that configuration has been set in code (“code-based”) or in the application's config file.</li>
	<li>Ensure that configuration can be discovered by design-time tools such that actions such as running the Code First pipeline can be correctly performed by tools.</li>
	<li>Allow, but not require, EF dependencies to be injected using the application developer's Inversion-of-Control (IoC) container of choice.</li>
</ul>
<h2>Dependency resolution</h2>
The <a href="http://entityframework.codeplex.com/wikipage?title=EF%20Configuration%20and%20Extensibility">first post</a> describes the new IDbDependencyResolver interface that allows EF to be configured and extended by resolving services that it depends on. The approach is based loosely a similar mechanism in MVC and Web API and makes use of the Service Locator pattern but with important modifications that address some of the issues with that pattern.

The dependency resolution design allows for an IoC container to be used with EF but does not require it or prescribe which one. We will publish another post providing an example of how to use an IoC container with EF.
<h2>Code-based configuration</h2>
The <a href="http://entityframework.codeplex.com/wikipage?title=Code-based%20Configuration">second post</a> describes the new DbConfiguration class that allows all configuration to be made in code while still allowing design-time tools to find and use this configuration. This is important when using Code First because design-time tools often need to know about the EF model, but this cannot be built correctly without also knowing how EF has been configured.

The code-based configuration also allows application developers to configure EF without needing to learn anything about dependency resolvers, even though these are the underlying building blocks on which the configuration is based. This is important for keeping the concept count low for developers just getting started with EF.
<h2>Feedback, please!</h2>
The reason we spend time writing this stuff and putting it out there is so that you can tell us what you think before it gets into a shipped product and becomes hard to change. Regardless of whether you love it, hate it, or think it's pointless, please do let us know!
