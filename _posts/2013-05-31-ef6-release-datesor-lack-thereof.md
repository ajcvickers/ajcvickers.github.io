---
layout: post
title: EF6 release dates…or lack thereof
date: 2013-05-31 09:13
author: ajcvickers
comments: true
categories: [EF6, Entity Framework]
---
On the EF team we frequently get questions along the lines of, “when will the next version of EF be released?” This is an entirely reasonable question to ask and it is frustrating to me personally that I cannot answer such questions. But of course this is nothing compared to the frustration felt by those of you who never get an answer. So why is it this way?

<strong>Update: </strong>Visual Studio 2013 has been <a href="http://blogs.msdn.com/b/bharry/archive/visual-studio-2013">announced for “this year” by Brian Harry</a>. If you read the rest of this post you will realized that this means EF6 RTM will be also released “this year”.
<h3>Disclaimer</h3>
Please keep in mind that this is my personal view and doesn’t necessarily reflect the views of my employer or others on the team.
<h3>Waves</h3>
First, let’s take a look at how we do RTM releases. It’s no secret that Microsoft releases lots of things that must work together. For example, EF and ASP.NET MVC both work together, and both of these must work with Visual Studio tooling. To coordinate such interactions and also for other reasons things are often released together in “waves”. So, as we have said elsewhere, EF6 will be released at the same time as the next version of Visual Studio.

“Okay then,” you say, “so when is the next version of Visual Studio going to be released?” As of right now, we on the EF team genuinely do not know the answer to that question. We do have to plan and so we do of course have a general idea of the timeframe involved. But we don’t know exact dates.

As we get closer to a major release we will learn exact dates…but we will still not tell them to you. This is partly because we aren’t allowed to tell you and could get fired if we did. But there’s a reason for this—the date is not just about EF, it’s about a wave of products that are all being released at or around the same time. Nobody on the EF team is in a position to know what strategic implications there are for these releases, or how the marketing and messaging is being handled at a high level. If somebody on the EF team leaked the date for all this it could undermine lots of careful planning and have massive implications for Microsoft way beyond EF.
<h3>Pre-releases</h3>
Okay, so that’s why we can’t give you dates for RTM releases that are part of waves. What about pre-release versions like <a href="https://docs.microsoft.com/archive/blogs/adonet/ef6-beta-1-available">EF6 beta 1</a> which was released yesterday? Surely we must know in advance when these will be released.

The answer to this is that yes, we do know, but we only know the exact date when we get very close to that date—usually we don’t know the exact date until the day we actually release. The process we have goes something like this:
<ul>
	<li>We tentatively plan for a pre-release timeframe based on where we are, what features are coming in, and other factors. At this stage in the process the degree of precision is something like “late May”.</li>
	<li>As we get closer to this tentative timeframe it may be moved out if we decide that users would be best served by delaying a week or so to get something else in, or polish an experience, or any number of other things.</li>
	<li>At some point we put a stake in the ground and say, “We’re going to release on Tuesday, May 28.” We then create a candidate build and the team does testing on that build. Less testing for an alpha, more testing for a beta, the most testing for RC’s and RTMs. (Of course, we’re testing all the time we develop as well; this is additional integration-level testing done in addition to normal testing.)</li>
	<li>If we find a ship-stopping bug, then we may cut another candidate build and push out the date. This happened for beta 1, which is why it was released on Thursday instead of Tuesday.</li>
	<li>Once the team is happy that the release is ready to go we ship it, usually the same day. Put another way, as I said above, the first time we know the exact date for a pre-release is when we arrive at that date.</li>
</ul>
We could, theoretically, at any point in this process provide you with what we know. We generally don’t do this because things change all the time and experience has shown that it gets very confusing if people read “early May”, which then turns into “late May”, which then becomes “May 28”, which then becomes “May 30”, and so on.

Not only can this be confusing, it is also often perceived negatively. For example, it is often seen as “missing a date” when it is instead just a normal part of how we plan in an agile environment where we expect and embrace change.
<h3>What I can tell you</h3>
As explained above, I can’t give exact dates. However, I can tell you that:
<ul>
	<li><a href="https://docs.microsoft.com/archive/blogs/adonet/2013/05/30/ef6-beta-1-available.aspx">EF6 beta 1</a> was released yesterday <img class="wlEmoticon wlEmoticon-smile" alt="Smile" src="http://oneunicorn.files.wordpress.com/2013/05/wlemoticon-smile.png" /></li>
	<li>We will make a release of EF6 with a “go-live” license before the end of the year</li>
	<li>EF6 will RTM at the same time as the next version of Visual Studio</li>
	<li>
<ul>
	<li><strong>Update: </strong>As mentioned in the update above, Visual Studio 2013 is now announced for “this year” which means EF6 will also be released “this year”.</li>
</ul>
</li>
</ul>
