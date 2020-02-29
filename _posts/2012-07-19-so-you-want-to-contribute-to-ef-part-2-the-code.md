---
layout: post
title: So you want to contribute to EF? Part 2: The code
date: 2012-07-19 09:20
author: ajcvickers
comments: true
categories: [EF6, Entity Framework, Open Source, OSS]
---
This is the second part of a <a href="/2012/07/19/so-you-want-to-contribute-to-ef-part-1-introduction/">series</a> providing some background to those who may want make contributes to the Entity Framework. This post will cover the high-level organization of the code describing the assemblies created and some of the namespaces used.
<h2>What is the code base like?</h2>
The Entity Framework has been developed over a number of years by many developers at a big company. Some of the code dates back to the WinFS days. The result of this is a code base that is, to say the least, not as clean as it could be. Feel free to complain about this if you want. Better yet be constructive in suggesting ways to improve the code or submit contributions that make those improvements.
<h2>What gets built?</h2>
The process for getting the EF source code and building it can be found on the <a href="http://entityframework.codeplex.com/documentation">EF CodePlex site</a>. The build generates several assemblies and NuGet packages.
<h3>Product assemblies</h3>
<ul>
	<li>EntityFramework.dll
<ul>
	<li>This is the main EF assembly and contains most of the runtime code including most of the code that was previously part of System.Data.Entity.dll in the .NET Framework (see below).</li>
</ul>
</li>
	<li>EntityFramework.SqlServer.dll
<ul>
	<li>This is the EF provider for Microsoft SQL Server. Previous to EF6 this code was part of System.Data.Entity.dll but it has been split out into a separate assembly for EF6 because it is independent of the rest of the runtime code and only used when connecting to SQL Server.</li>
</ul>
</li>
	<li>EntityFramework.SqlServerCompact.dll
<ul>
	<li>This is the EF provider for Microsoft SQL Server Compact Edition. Previous to EF6 the SQL Compact provider for EF was shipped with the associated ADO.NET provider. In EF6 the ADO.NET and EF providers are less tightly coupled which allows the EF provider to be included in the open source code and built into a separate assembly.</li>
</ul>
</li>
</ul>
<h3>Tooling assemblies</h3>
<ul>
	<li>EntityFramework.PowerShell.dll
<ul>
	<li>This assembly contains most of the code used by EF when integrating with PowerShell and NuGet. In particular, it contains the PowerShell code for the Migrations commands and for manipulating config files on EF NuGet package installation.</li>
</ul>
</li>
	<li>EntityFramework.PowerShell.Utility.dll
<ul>
	<li>This is a small shim assembly that provides a mechanism for binding to different versions of other assemblies from PowerShell. This is needed because sometimes PowerShell is using the .NET 4 version of assemblies and sometimes it is using the .NET 4.5 version.</li>
</ul>
</li>
	<li>Migrate.exe\Migrate.x86.exe
<ul>
	<li>This executable is the command-line façade for using Migrations.</li>
</ul>
</li>
</ul>
<h3>Test assemblies</h3>
<ul>
	<li>EntityFramework.UnitTests.dll</li>
	<li>EntityFramework.FunctionalTests.dll</li>
	<li>EntityFramework.VBTests.dll</li>
</ul>
See part 3 of this series for information about the tests.
<h3>NuGet packages</h3>
The build combines the product and tooling assemblies into two NuGet packages:
<ul>
	<li>EntityFramework
<ul>
	<li>The main EF NuGet package containing EntityFramework.dll and EntityFramework.SqlServer.dll plus all the PowerShell and Migrations integration.</li>
</ul>
</li>
	<li>EntityFramework.SqlServerCompact
<ul>
	<li>A NuGet package that installs EntityFramework.SqlServerCompact.dll and sets EF up to use the SQL Server Compact provider by default.</li>
</ul>
</li>
</ul>
<h2>Code organization</h2>
There are two things that effect the EF code organization more than anything else. The first is the historical split between core code and the DbContext/Code First code; the second is underpinning of the EF core code by the Entity Data model (EDM).

I have written more details about the high-level architecture of EF and its EDM underpinnings in part 5 of this series, so I'm not going to cover that here. Instead I'm going to talk here about the split between the core code and the DbContext/Code First code.
<h3>The EF 4.1 code split</h3>
The first two versions of EF were released as part of the .NET Framework. At about the time that the second version of EF (called EF4 to align with .NET 4) was released <a href="/2011/04/12/a-brief-history-of-ef-4-1/">a group of people were already working on features that became Code First and the DbContext APIs</a>. These features formed an out-of-band (OOB) release called EF 4.1. Significantly, this code was never included in the .NET Framework.

The obvious consequence of this is that since EF 4.1 the Entity Framework has consisted of two code bases—the code in the .NET Framework which we refer to as the core, and the OOB code for DbContext/Code First. The OOB code was dependent on the core, but the core didn't know anything about the OOB code.
<h3>The EF6 merge</h3>
With the move to open source for EF6 these two code bases have been brought together since all of EF6 is being released OOB and none of it is part of the .NET Framework. But when you look at the source code you will still see the historical separation. Generally speaking, code which comes from the core has been moved into namespaces under System.Data.Entity.Core.

The core code is not considered the primary public API going forward. In many situations it is treated as internal code that the DbContext and Code First APIs use for their implementation but not something that EF developers will work with directly. It remains public for those applications that make use of ObjectContext and the older APIs and because there are some scenarios that require dropping down to these lower-level APIs.

Most of the dependencies still go from the DbContext/Code First code to the core code and not the other way round. However, this separation is sometimes artificial and was forced by the model in which the releases were made. This means as we evolve the design of EF6 and beyond we are not trying to maintain this as a hard boundary, but rather we try to ensure that the abstractions and dependencies are appropriate going forward.
<h3>The namespaces</h3>
The main namespaces for EntityFramework.dll are:
<ul>
	<li>System.Data.Entity
<ul>
	<li>This is the top-level namespace for the main public APIs. Types go into this namespace if they are fundamental to the mainline scenarios for EF. We try to keep this namespace relatively uncluttered.</li>
</ul>
</li>
	<li>System.Data.Entity.Config
<ul>
	<li>This namespace contains the code-based configuration and dependency injection features that we are working on for EF6.</li>
</ul>
</li>
	<li>System.Data.Entity.Core and below
<ul>
	<li>As described above, this namespace contains most of the core EF code that was originally shipped with the .NET Framework.</li>
</ul>
</li>
	<li>System.Data.Entity.Edm and below
<ul>
	<li>This namespace contains the EDM object model used by Code First for building and manipulating the mappings between objects and the database. These types are currently internal but the intention is to make them public to support making Code First conventions public and to allow direct lower-level manipulation of the EDM model.</li>
</ul>
</li>
	<li>System.Data.Entity.Infrastructure
<ul>
	<li>This namespace contains the public types involved in less common DbContext scenarios or types that are often not used directly but are instead dotted through in the DbContext fluent APIs. A lot of application classes that use EF will not need to have a using directive for the infrastructure namespace while pretty much all will need a using directive for System.Data.Entity.</li>
</ul>
</li>
	<li>System.Data.Entity.Internal and below
<ul>
	<li>This namespace contains internal types used in the implementation of the DbContext APIs. The DbContext code keeps its internals here but in other parts of the code base types are organized by functional area rather than by accessibility and going forward we are tending to follow the latter pattern. In other words, don't feel that you have to put internal types  in Internal namespaces.</li>
</ul>
</li>
	<li>System.Data.Entity.Migrations and below
<ul>
	<li>This namespace contains the Migrations types.</li>
</ul>
</li>
	<li>System.Data.Entity.ModelConfiguration
<ul>
	<li>This namespace contains most of the types used by the Code First fluent API and those used for Code First configuration using data annotations. The work of building a Code First model happens here with the output being types from the Edm namespace described above.</li>
</ul>
</li>
	<li>System.Data.Entity.Spatial and below
<ul>
	<li>This namespace contains types from the core that are used for spatial types. These types are not in the <em>Core </em>namespace because they are used directly by DbContext-based applications even though they originated in the core assemblies of the .NET Framework.</li>
</ul>
</li>
	<li>System.Data.Entity.Utilities
<ul>
	<li>This namespace contains general internal utility classes, most of which are in the form of extension methods.</li>
</ul>
</li>
	<li>System.Data.Entity.Validation and below
<ul>
	<li>This namespace contains the types used for validation of entities based on data annotations.</li>
</ul>
</li>
</ul>
<h2>String resources</h2>
It must be possible to localize every string that the Entity Framework product assemblies use. This means that any time code uses a string (such as in an exception message) the English string must be placed in the appropriate Resources.resx file (which can be found in the Properties folder) and given a name.

The Resouces.tt file is a T4 template used to create some helper methods that can be used to access localized strings. After adding a string to the resx file right-click on Resources.tt and choose “Run Custom Tool” to generate the helper methods. Examples of how this all works are easy to find in the code.
<h2>Summary</h2>
The EF build generates the main EF runtime assembly (EntityFramework.dll) and two providers together with everything needed to create the EF NuGet packages and Migrations tooling.

If there is anything here that you need more information about don't hesitate to contact me or others on the EF team. In the next post I'll look at the EF tests.
