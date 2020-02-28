---
layout: default
title: "EF 4.1 Quick Tip: Add/Attach Returns Your Entity"
date: 2011-04-19 22:16
day: 19th
month: April
year: 2011
author: ajcvickers
permalink: 2011/04/19/ef-4-1-quick-tip-addattach-returns-your-entity/
---

# Entity Framework 4.1
# Quick Tip: Add/Attach returns your entity

<p>We did a few things in EF 4.1 that were not at all ground-breaking or cool or anything like that, but were intended only to make programming a little bit more pleasant. One of those was making the Add and Attach methods return a reference to the added or attached entity.</p><p>This is nice because it can sometimes save a local variable or a line or two of code and thereby make your code easier to read.>&nbsp; For example, in a little test app I was just working on I wanted to create a new “Doc” entity, add it to the context, then use it later in the same method.>&nbsp; Instead of writing:</p>  

``` c#
var doc = new Doc();
context.Docs.Add(doc);
```

<p>I was able to write this:</p>

``` c#
var doc = context.Docs.Add(new Doc());
```

<p>Another advantage of returning the added entity is that it allows the Add/Attach methods to be more composeable. For example, later in the same method I had some code that attempted to find an entity in the context and, if the entity was not found, created a new one and added that new entity to the context. Instead of writing:</p>

``` c#
var word = context.ChangeTracker.Entries<Word>()
                                .Select(e => e.Entity)
                                .SingleOrDefault(w => w.TheWord == theWord);
if (word == null)
{
    word = new Word { TheWord = theWord };
    context.Words.Add(word);
}
```

<p>I was able to use a more functional approach:</p>

``` c#
var word = context.ChangeTracker.Entries<Word>()
                                .Select(e => e.Entity)
                                .SingleOrDefault(w => w.TheWord == theWord)
    ?? context.Words.Add(new Word { TheWord = theWord });
```

<p>Hopefully once in a while you can use the return value from Add or Attach and be just a little bit happier because of it. :-)</p>
