---
layout: post
title: EF 4.1 Quick Tip: Add/Attach Returns Your Entity
date: 2011-04-19 22:16
author: ajcvickers
comments: true
categories: [Code First, DbContext, DbContext API, DbSet.Add, DbSet.Attach, Entity Framework]
---
<p>We did a few things in EF 4.1 that were not at all ground-breaking or cool or anything like that, but were intended only to make programming a little bit more pleasant. One of those was making the Add and Attach methods return a reference to the added or attached entity.</p><!--more--><p>This is nice because it can sometimes save a local variable or a line or two of code and thereby make your code easier to read.&#160; For example, in a little test app I was just working on I wanted to create a new “Doc” entity, add it to the context, then use it later in the same method.&#160; Instead of writing:</p>  <div style="display:inline;float:none;margin:0;padding:0;" id="scid:C89E2BDB-ADD3-4f7a-9810-1B7EACF446C1:f9d0a090-4e72-4bc7-a12e-bbeee6ec4b19" class="wlWriterEditableSmartContent"><pre>
[sourcecode language="csharp" padlinenumbers="true"]
var doc = new Doc();
context.Docs.Add(doc);

[/sourcecode]
</pre>
</div>

<p>I was able to write this:</p>

<div style="display:inline;float:none;margin:0;padding:0;" id="scid:C89E2BDB-ADD3-4f7a-9810-1B7EACF446C1:124ec481-4d4c-4c6b-be15-0b563ea9cb97" class="wlWriterEditableSmartContent"><pre>
[sourcecode language="csharp"]
var doc = context.Docs.Add(new Doc());

[/sourcecode]
</pre>
</div>

<p>Another advantage of returning the added entity is that it allows the Add/Attach methods to be more composeable. For example, later in the same method I had some code that attempted to find an entity in the context and, if the entity was not found, created a new one and added that new entity to the context. Instead of writing:</p>

<div style="display:inline;float:none;margin:0;padding:0;" id="scid:C89E2BDB-ADD3-4f7a-9810-1B7EACF446C1:e8efa69b-65db-472b-82cb-52cfb7ad0828" class="wlWriterEditableSmartContent"><pre>
[sourcecode language="csharp"]
var word = context.ChangeTracker.Entries&lt;Word&gt;()
                                .Select(e =&gt; e.Entity)
                                .SingleOrDefault(w =&gt; w.TheWord == theWord);
if (word == null)
{
    word = new Word { TheWord = theWord };
    context.Words.Add(word);
}

[/sourcecode]
</pre>
</div>

<p>I was able to use a more functional approach:</p>

<div style="display:inline;float:none;margin:0;padding:0;" id="scid:C89E2BDB-ADD3-4f7a-9810-1B7EACF446C1:ce939017-b0c4-4ba4-9311-17c00d53e1f7" class="wlWriterEditableSmartContent"><pre>
[sourcecode language="csharp"]
var word = context.ChangeTracker.Entries&lt;Word&gt;()
                                .Select(e =&gt; e.Entity)
                                .SingleOrDefault(w =&gt; w.TheWord == theWord)
    ?? context.Words.Add(new Word { TheWord = theWord });

[/sourcecode]
</pre>
</div>

<p>Hopefully once in a while you can use the return value from Add or Attach and be just a little bit happier because of it. :-)</p>

<p>Thanks for reading!
  <br />Arthur</p>
