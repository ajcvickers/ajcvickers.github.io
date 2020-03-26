---
layout: default
title: "Totally made up conversations about choosing Entity Framework version numbers"
date: 2020-03-26 09:55
day: 26th
month: March
year: 2020
author: ajcvickers
permalink: 2020/03/26/numbersarehard/
---

# All your versions are belong to us

### Totally made up conversations about choosing Entity Framework version numbers

These conversations are a work of fiction.
Any resemblance of to real Microsoft employees, either past or present, is purely coincidental.

### EF1

A: So this Entity Framework thing we're shipping. What version is it?<br>
B: It's version 1. Duh.<br>
A: So the assemblies are stamped with 1.0?<br>
B: Err. Well, no. The assemblies are in .NET Framework, so they are versioned as 3.5.<br>
A: So this is really version 3.5? Isn't that weird for a v1?<br>
B: I guess we shouldn't give it any version, since it's in .NET Framework.<br>
A: Okay...<br>
B: How about, "ADO.NET Entity Framework for .NET Framework 3.5 Service Pack 1"?<br>
A: Catchy!<br>

### EF4

A: So this Entity Framework thing we're shipping. What version is it?<br>
B. It's version 2. Duh.<br>
A: So the assemblies are stamped with 2.0?<br>
B: Err. Well, no. The assemblies are in .NET Framework, so they are versioned as 4.0.<br>
A: So this is really version 4.0? Isn't that weird for a v2?<br>
B: I guess we shouldn't give it any version, since it's in .NET Framework.<br>
A: Err. We've already been telling customers that it's EF2.<br>
B: Hmm.<br>
A: Also, customers are referring to "ADO.NET Entity Framework for .NET Framework 3.5 Service Pack 1" as EF1.<br>
B: What!? Can't we stop them? We gave it a catchy name for a reason!<br>
A: Customers will be customers...<br>
B: Fine. Fine. EF4 it is.<br>
A: Four?<br>
B: Yes. it's in .NET 4 so it's EF4.<br>
A: But...<br>

### Code First

A: So this Entity Framework Code Only thing we're shipping. What's it called?<br>
B: It's a preview right?<br>
A: Yes.<br>
B: And it's a preview of new features, right?<br>
A: Yes...<br>
B: So call it a "Feature Community Technology Preview". That's the catchy name we have for previews.<br>
A: Why not just "Preview"?<br>
B: Because otherwise people won't know it's for the community, uses technology, and has features.<br>
A: Right. Got it. So it's "ADO.NET Entity Framework Community Technology Preview"?<br>
B: Yep. Catchy and informative!<br>
A: Hmm.<br>
B: Hang on a minute. What is "Code Only" anyway?<br>
A: Well it describes how only code is needed to specify database mappings.<br>
B: But we have "Database First" and "Model First". Can't we have "Code First" too? It would be really catchy!<br>
A: But that implies you must write the code first.<br>
B. Yeah, but "Code First" is catchy!<br>
A: But...okay, fine. What about the new DbContext API surface? How we refer to that?<br>
B: You can't give it a name. New brands are bad.<br>
A: How do we refer to it?<br>
B: Call it "productivity improvements".<br>
A: Nice! Catchy and it doesn't mean anything!<br>

### EF 4.1

A: So this Entity Framework thing we're releasing on NuGet. What version is it?<br>
B: It's a minor release that builds on EF4, right?<br>
A: Yep.<br>
B: Then call the package EntityFramework and version it as 4.1.<br>
A: Are you sure?<br>
B: Yes. Doesn't it make sense?<br>
A: No, no. It does make sense. I just wasn't expecting it to!<br>
 
### EF 4.2

A: I'm going to go out on a limb here. Can it be EF 4.2?<br>
B: Ship-it!<br>
A: I can't believe this is happening. It makes so much sense!<br>

### EF 4.3

A: Rinse and repeat?<br>
B: Yep.<br>
A: Awesome!<br>

### EF5

A: So this Entity Framework thing we're shipping. What version is it?<br>
B. So it's aligned with .NET 4.5, right?<br>
A: Yes.<br>
B: So EF 4.5 then?<br>
A: But it's a major release and semantic versioning says it should be 5.0.<br>
B: Fine. Call it EF5, then. I miss the catchy names we used to have. But hey, we're living in a NuGet world!<br>
A: Awesome!<br>

### EF6

A: EF6?<br>
B: Yes.<br>

### EF Core 1.0

A: So this new Entity Framework thing we're building. What do we call it?<br>
B: Entity Framework?<br>
A: But it's a different code base and works differently. We've been calling it EF Lite.<br>
B: No, no. New brands are bad. It must be called Entity Framework.<br>
A: Won't that be confusing and send the wrong message?<br>
B: Just bump the major version.<br>
A: So, EF7?<br>
B: Yep.<br>
A: But... Okay.<br>
B: No, wait. We just decided, everything Project K will now be "Core".<br>
A: So, EF Core?<br>
B: Yep. But you have to change the package name as well. Some people can't tell EntityFramework comes from Microsoft.<br>
A: Microsoft.EFCore?<br>
B: Not catchy enough. How about, "Microsoft.EntityFrameworkCore"?<br>
A: So be it. By the way, you do realize we're not actually really ready for a 1.0 release?<br>
B: Does it work.<br>
A: Well, yeah. But there's a lot of stuff missing.<br>
B: It must ship!<br>

### EF Core 1.1 - 3.1

A: Can we use semantic versioning and make sane version number changes?<br>
B: Why would we not do?<br>
A: I love new Microsoft.<br>

### EF Core 5.0

A: So, we're not calling ".NET Core" ".NET Core" anymore?<br>
B: Nope.<br>
A: What about EF Core.<br>
B: It can stay the same.<br>
A: Cool. What version number?<br>
B: Everything in .NET 5 is version 5.<br>
A: But EF Core isn't technically even part of .NET 5. It should be just EF Core 4.0.<br>
B: Everything in .NET 5 is version 5.<br>
A: Five might be confusing because we already had an EF5. And a six. And kind of a seven. Maybe EF Core 8?<br>
B: Everything in .NET 5 is version 5.<br>
A: Well, it given our history, it certainly could be worse!<br>
