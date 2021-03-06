---
layout: default
title: Why are the DbContext T4 templates so different from the EF4 POCO templates?
date: 2011-11-24 14:51
day: 24th
month: November
year: 2011
author: ajcvickers
permalink: 2011/11/24/why-are-the-dbcontext-t4-templates-so-different-from-the-ef4-poco-templates/
---

# Entity Framework 4.2
# Why are the DbContext T4 templates so different from the EF4 POCO templates?

<p>A while ago <a href="http://thedatafarm.com/blog/">Julie Lerman</a> asked the question above and then suggested that the response should be blogged—so, better late than never, here's the post.</p><p>Of course, the most obvious difference is that the DbContext templates use DbContext while the EF4 POCO templates use ObjectContext. But beyond that, the POCO templates have a lot of code that attempts to do fixup of properties and relationships even when the entity is not attached to the context. The DbContext templates have none of this code. Why?</p>  <p>The answer is that it was a difference in philosophy between how we created the two templates. For the original POCO templates we went with the approach of making them do everything that we could even though they were POCO. So with these templates you get change tracking proxies and fixup in the templates. There turned out to be several downsides with this, one being that the templates became very complex and while the entity classes were technically still POCO they became less “pure” POCO because of all the EF-generated code built into them. Another downside was that they generated change tracking proxies by default. Change tracking proxies are great for some scenarios, but we have observed that in the majority of situations using just lazy-loading proxies reduces complexity and works better. Also, when using change tracking proxies and fixup in the same entities you often have both the EF stack and the proxies trying to do the same fixup at the same time, which causes additional problems. </p>  <p>So for the DbContext templates we took a different approach and created templates that generated entities and a context that is really, really simple. This means that when you get your classes and look at them you can immediately see everything that they do—something that cannot be said for any of the other types of entities that we generate. This also means that the templates are quite simple, which makes it easier for people to use them as a starting point for their own customizations. They don't do fixup, but that seldom seems to be a problem for most types of app, especially because the context does fixup quite frequently—essentially, whenever DetectChanges is called, which happens quite often when using DbContext. </p>  <p>So the bottom line is that for most people the DbContext templates and the entity classes that they generate are a better starting point for domain models than the POCO templates and their entity classes. </p> 
