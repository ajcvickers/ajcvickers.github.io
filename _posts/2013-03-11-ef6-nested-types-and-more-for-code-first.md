---
layout: post
title: EF6: Nested types and more for Code First
date: 2013-03-11 10:39
author: ajcvickers
comments: true
categories: [Code First, EF6, Entity Framework, POCO]
---
Ever since POCO support was introduced in EF4 there have been two limiting restrictions on the types that can be mapped:
<ul>
	<li>Types nested inside other types were not supported</li>
	<li>Types were matched by simple names only</li>
</ul>
Some recent check-ins to the EF6 code base have lifted these restrictions to some degree when using Code First. This post describes what is now supported, what restrictions still exist, and provides some background about why things are the way they are.<!--more-->

Note that these changes did not make it into EF6 alpha 3, but they are in recent <a href="http://entityframework.codeplex.com/wikipage?title=Nightly%20Builds">nightly builds</a>.
<h2>What is now supported?</h2>
In a nutshell, the changes described below mean that:
<ul>
	<li>Types not used in a Code First model but with the same simple name as types in the model will not cause EF to throw. For example, you can now have DTO types and entity types with the same names in the same assembly.</li>
	<li>Nested types can be used.</li>
</ul>
Note that “simple name” means the name of the type excluding its namespace or any outer type names. The simple name of “My.Namespace.MyType” is “MyType” and the simple name of the nested type “My.Namespace.OuterType+InnerType” is “InnerType”.
<h2>What restrictions still exist?</h2>
Every type used in a given Code First model must still have a unique simple name. That is, using “My.Namespace1.Product” and “My.Namespace2.Product” in the same Code First model is not supported. Both these types can exist in the same assembly, and both can be used in different Code First models, or one can be used by Code First and the other not, but they can’t both be used by the <em>same </em>Code First model.

Also, these changes only apply to Code First. At the time of writing Database First and Model First still have the restrictions on types that they always had.

See <em>Future Plans </em>below for ideas on how we plan to remove these remaining restrictions in a future release of EF.
<h2>Background</h2>
Consider starting an EF application that contains a model in an EDMX file. EF reads the model from the EDMX file and then must find the CLR types that should be used with this model. For POCO entities this is done by scanning through assemblies that might contain the types and looking for matches. Matches are made by simple type name only—that is, there is no concept of matching namespaces or nested types. An exception is thrown if EF finds two types that have the same simple name and are therefore both potential matches.

This is clearly not a very good way of doing things even though arguments were made that it had to be this way. I won’t say any more about the decision to do it this way; right or wrong it is what it is.
<h2>Code First arrives</h2>
With Code First the CLR types that should be used do not need to be discovered—they are the “code” in Code First from which the model is created. So, theoretically, the above process should not have been needed for Code First. The problem was that until EF6 Code First was in a separate assembly to core EF with no access to internals. It therefore worked by creating an EDMX in memory and pushing that to core EF, which then loaded the EDMX and was forced to discover types again using the process described above.
<h2>What changed?</h2>
With the move out of the .NET Framework for EF6, Code First and the core code are now all in EntityFramework.dll. This has allowed what we refer to as “metadata unification”. That is, Code First now doesn’t create an EDMX in memory, but rather creates runtime metadata directly. (At the time of writing the MSL part of the EDMX is still created as XML because its validation at runtime is tightly coupled to reading this data from XML.)

This metadata unification was a very significant and non-trivial task, but now that it has been done it opens the door for other changes. (One of these is stored proc mapping in Code First—one of the reasons this is coming late in the EF6 cycle is because it makes use of this metadata unification.)

With metadata unification Code First now annotates the model it builds with CLR type information. This type information can then be used directly by the runtime so that for Code First the process described above is no longer needed. This means that:
<ul>
	<li>Nested types work with Code First because the discovery mechanism that doesn’t understand them is no longer used</li>
	<li>Types not used in the Code First model don’t have any impact because core EF no longer needs to try to find the correct types to use</li>
</ul>
<h2>Future plans</h2>
As stated above, every type used in a given model must still have a unique simple name. This is because every type in the conceptual model (the CSDL part of an EDMX) is still included in the same conceptual model namespace. The conceptual model (and in particular the CSDL XML schema) is not setup to deal well with types in the same model but in different namespaces. However, this is something we want to tackle post EF6 such that EF can use any number of types with the same simple name so long as they have different namespaces or outer types.

We also plan to change the way type discovery happens for Database First and Model First when an EDMX file is loaded. It’s not clear exactly how this will be done, but the most promising approach seems to be to use DbSet/ObjectSet entity type information from the context coupled with the model shape in order to discover the CLR types to use. This should effectively remove the assembly scanning and simple name matching that Database First and Model First currently use. (And we can then hopefully dump this code entirely from EntityFramework.dll.)

It’s worth noting that using DbSet/ObjectSet information could break those using Database First or Model First without using DbSets or ObjectSets. It’s not clear that anybody is doing this, but if you are now would be a great time to contact the EF team!
