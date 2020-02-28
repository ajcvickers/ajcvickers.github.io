---
layout: post
title: EF Core 1.1: Looking at your model in the debugger
date: 2016-11-17 12:05
author: ajcvickers
comments: true
categories: [EF Core, EF Core Metadata, Entity Framework, Metadata]
---
As developers on the EF team it is often useful to see what is in the metadata model. To facilitate this, the model and all other metadata elements are able to generate a "debug view" which provides lots of useful information about the metadata in a terse but human-readable form.



<h3>Getting at it</h3>

In the debugger, expand context -> Model -> DebugView -> View.

<a href="https://oneunicorn.files.wordpress.com/2016/11/debugview.png"><img src="https://oneunicorn.files.wordpress.com/2016/11/debugview.png" alt="EF Core model debug view" width="640" height="297" class="alignright size-full wp-image-451" /></a>

(The additional level of indirection from DebugView to View is intentional. For very large models it can be expensive to build the debug view string, which in turn can cause perf issues in the debugger. The indirection prevents the debugger from building the string unless it is requested by expanding the DebugView property.)

<h3>What it looks like</h3>

Here's the output for the model I was using while writing the post on notification entities:

[code lang=text]
Model: 
  EntityType: Blog ChangeTrackingStrategy.ChangingAndChangedNotificationsWithOriginalValues
    Properties: 
      Id (_id, int) Required PK ReadOnlyAfterSave RequiresValueGenerator ValueGenerated.OnAdd 0 0 0 -1 0
      Title (_title, string) 1 1 -1 -1 -1
    Navigations: 
      Posts (<Posts>k__BackingField, ICollection<Post>) Collection ToDependent Post Inverse: Blog 0 -1 -1 -1 -1
    Keys: 
      Id PK
    Annotations: 
      Relational:TableName: Blogs
  EntityType: Post
    Properties: 
      Id (int) Required PK ReadOnlyAfterSave RequiresValueGenerator ValueGenerated.OnAdd 0 0 0 -1 0
      BlogId (int) Required FK Index 1 1 1 -1 1
      Title (string) 2 2 -1 -1 -1
    Navigations: 
      Blog (<Blog>k__BackingField, Blog) ToPrincipal Blog Inverse: Posts 0 -1 2 -1 -1
    Keys: 
      Id PK
    Foreign keys: 
      BlogId -> Blog.Id ToDependent: Posts ToPrincipal: Blog
    Annotations: 
      Relational:TableName: Posts
Annotations: 
  ProductVersion: 1.1.0-preview1-22509
  SqlServer:ValueGenerationStrategy: IdentityColumn
[/code]

Things to notice:

<ul>
<li>Special configuration such the change tracking strategy is called out. Default configuration (such as snapshot change tracking) is usually not shown.</li>
<li>Properties have the form <Name> (<field>, <type>) <flags> <indexes>

<ul>
<li>Indexes are useful when debugging internal EF issues but probably not much use for app developers</li>
<li>Notice that whether a property is a PK and/or FK is called out in the flags</li>
</ul></li>
<li>Navigations have the form <Name> (<field>, <type>) <flags> <inverse> <indexes>

<ul>
<li>The inverse information makes it easy to see that two ends of a relationship are correctly associated with each other</li>
</ul></li>
<li>Keys lists both the PK (called out) and any alternate keys</li>
<li>Foreign keys have the form <dependent property> -> <principal type>.<principal property> <dependent nav> <principal nav></li>
<li>All elements list additional annotations at the end

<ul>
<li>Relational mappings, such as table names, are listed in the annotations</li>
</ul></li>
</ul>

Hopefully this will help with understanding of the EF model and make it easier to find any issues. If you end up filing a bug against EF, then consider including this view with your issue.
