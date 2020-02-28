---
layout: post
title: Why open sourcing Entity Framework is a great move
date: 2012-07-19 08:58
author: ajcvickers
comments: true
categories: [Entity Framework, Open Source, OSS]
---
You have likely heard by now that the <a title="Entity Framework source code has been released under an open source license" href="https://docs.microsoft.com/archive/blogs/adonet/entity-framework-and-open-source">Entity Framework source code has been released under an open source license</a>. I’m a developer on the EF team and I’d like to give you my perspective on this.

I’ve been on the team for several years and I have been involved a lot in the decision to take EF open source over the last year. (We’ve been working on this for quite some time.) That being said, keep in mind that everything on this blog including this post are my opinions and don’t necessarily reflect those of Microsoft or others on the team.


<h2>We are passionate about EF</h2>
First, as <a title="Scott Guthrie" href="http://weblogs.asp.net/scottgu/archive/2012/07/19/entity-framework-and-open-source.aspx">Scott Guthrie </a>and <a title="Scott Hanselman " href="http://www.hanselman.com/blog/EntityFrameworkMagicUnicornAndMuchMoreIsNowOpenSourceWithTakeBacks.aspx">Scott Hanselman </a>have both made clear, making EF open source is <em><strong>not </strong></em>about reducing investment in EF. The group of people I’m sitting with have, for the most part, also been on the EF team for several years. We’ve been working passionately during that time to steadily make EF better and better. The newer members of the team share that passion. And I think I can safely say that we all believe that making EF open source is the best way to continue making EF better. We’re all-in on this. The excitement within the team around this announcement is palpable.
<h2>Why we’re going open source</h2>
Making EF open source is not any easy thing to do and has taken a lot of time and work. There are lots of reasons why it’s a good thing, but to me it boils down to three main points:
<ul>
	<li>To further open up the design and development process that we started in earnest with the EF 4.1 preview releases. People who are interested will be able to get nightly builds, see the source code changes, engage us in discussion about the design and implementation, comment on our design meeting notes and provide feedback and input into the design. Will this be as easy as it would be if you were here? No, of course not. But the team is committed to opening up the development and involving the community and we will work hard to make that happen.</li>
	<li>To allow others to not only be involved in the design and implementation, but also to actively contribute to it.</li>
	<li>To allow others to fork the EF code. This could be just to fix a bug quickly to unblock work that uses EF, or it could be to create a custom version of EF with different capabilities. Since the changes can also be contributed back this gives people a lot greater flexibility in how they develop with EF.</li>
</ul>
<h2>There’s a lot to do</h2>
The team here has a long backlog of features to implement in EF. Some of these things (e.g. batch CRUD, public conventions, stored proc mapping in Code First, async support) you’re probably aware of. Others are more forward looking and innovative. We’re going to be working hard to do these things and others, but we also want your help. <strong>We’re serious about taking contributions.</strong> The process may take some time to get running smoothly, but if it seems frustrating at any time be aware that we will be sharing your frustration and working hard to remove any roadblocks. We’re going to make this work.
<h2>Collaboration</h2>
The EF team has been a very collaborative team for several years now. We discuss things openly in the team, take ideas from everyone, iterate on them, try things, throw some of it away keeping just the good bits, and so on. It is our intention to continue this process but as much as possible make it open to involve the community. We want to make bugs, code reviews, discussions, plans, backlogs, and so on openly available. All of this may not happen immediately, but we do intend to make this a truly open project with as much collaboration from the community as possible.
<h2>Challenges</h2>
In addition to the challenges of opening up the team and process there are also considerable technical challenges with making EF open source. EF was traditionally shipped as part of the .NET Framework and it now needs to be moved out of the .NET Framework*, which entails breaking certain dependencies and making certain breaking changes. We’ve already worked on removing some of these dependencies; others we are currently working on. There will be issues and things that break in early builds, but with community help we think we can make the transition relatively smooth. Your feedback is really important for this. Grab builds, find things that either aren’t great or are just broken and let us know. Better yet, send a pull request with a fix!

Also, the EF code base is not very clean. This shouldn’t be a surprise—it has a long history reaching back into the WinFS days and has had a lot of people working on it. It may be a challenge to make changes in some of the code—it is for us sometimes. But the community is smart and we fully intend to help the community learn as much as possible.

*Please note that the core EF assemblies that are in the Framework (for example, System.Data.Entity.dll) will continue to be part of the Framework and continue to work. We won’t break existing applications. However, future versions of EF will stop using these assemblies.
<h2>We have a bright future</h2>
I am really excited about the future of an open source EF. With your help we’re going to build on the progress we’ve made over the last few years and EF will continue to get better and better. I look forward to working with all of you!

Thanks,
Arthur
