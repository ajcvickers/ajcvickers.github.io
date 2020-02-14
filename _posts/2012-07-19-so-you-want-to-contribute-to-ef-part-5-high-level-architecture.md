---
layout: post
title: So you want to contribute to EF? Part 5: High-level architecture
date: 2012-07-19 10:18
author: ajcvickers
comments: true
categories: [EF6, Entity Framework, Open Source, OSS]
---
This is the fifth part of a <a href="http://blog.oneunicorn.com/2012/07/19/so-you-want-to-contribute-to-ef-part-1-introduction/">series</a> providing some background to those who may want make contributes to the Entity Framework. In this post I’ll give an extremely high-level overview of the EF architecture.<!--more-->
<h2>The Entity Data Model</h2>
The first thing to understand is that the EF is based on the concept of an Entity Data Model (EDM). Very roughly, this means that the EF is making use of several different representations (models) and much of the work it does is to move the data between those models using defined mappings.
<h3>The three models</h3>
The three fundamental models are:
<ul>
	<li>The store model, which matches how the data is represented in a relational database.</li>
	<li>The conceptual model, which provides a conceptual representation of the data not tied to any particular use. The conceptual model could be used for an OData feed, the generation of reports, or, as is usually the case with EF, as the basis for mapping to .NET objects.</li>
	<li>The object model, which are your .NET classes.</li>
</ul>
You can think of these three models as forming three layers. The EF then uses mappings to transform between the layers:
<ul>
	<li>C/S mapping handles how data is transformed between the store and conceptual models.</li>
	<li>O/C mapping handles how data is transformed between the conceptual and object models.</li>
</ul>
So, for example, when you change a property in an object and send that change to the database EF will (in principal):
<ul>
	<li>Use the change to the object model (the property change) together with O/C mapping to update the conceptual model with appropriate changes.</li>
	<li>Use the changes to the conceptual model together with C/S mapping to update the store model with appropriate changes.</li>
	<li>Send the store model changes to the database.</li>
</ul>
I say "in principal” because EF does not necessarily store multiple copies of the data, but rather passes the data through pipelines that dynamically transform it as it passes through.
<h3>Hiding the EDM</h3>
The Entity Framework is an object/relational mapper (O/RM). However, the original motivation for creating the Entity Framework was not to create an O/RM. It was instead a means to promote and facilitate the use of the EDM as a common way of representing and sharing domain models.

Most developers who use EF don’t care about the EDM, which is fine. Indeed, from EF 4.1 onwards the EDM is intentionally abstracted away from most uses of EF. It’s still there and can enable some powerful scenarios when needed, but most of the time as a user of EF you don’t have to think about it.

As an EF developer it is important to recognize and understand the role of EDM but, in the spirit described in Part 4 of this series, it is often desirable to keep the EDM abstracted away from the common developer experience.
<h2>Main sub-systems</h2>
Using EDM as a foundation the EF code is divided into several sub-systems.
<h3>Object services</h3>
The object services code is responsible for most of the public API used by applications to work with the EF in conjunction with the application’s object model. For example, the context (DbContext/ObjectContext) and set (DbSet/ObjectSet) classes are part of object services. Adding new entities to the context or creating a new query are the kinds of things that object services is responsible for.
<h3>State management</h3>
The state management code takes care of storing objects that have been queried or added to the context and tracking changes made to these objects. The main class responsible for state management is the ObjectStateManager. Object services makes use of the ObjectStateManager for many of the interactions it mediates between the application and the EF.

For example, when an object is returned from a query it is stored in the ObjectStateManager. Let’s assume that a change is then made to that object and the application (explicitly or implicitly) asks EF to detect that change. The change detection is performed by the state management code and the change is recorded in the ObjectStateManager. When SaveChanges is called the update pipeline asks the ObjectStateManager for the changes that have been made in order to construct updates to be sent to the database.
<h3>Metadata</h3>
The metadata code is responsible for creating representations of the object, conceptual, and store models, and the o/c and c/s mappings between these models. The principal metadata class is the MetadataWorkspace which takes care of loading and parsing the models and providing an API surface to access metadata about the models and mappings.

The conceptual and store models and the c/s mappings are read from CSDL, SSDL, and MSL files respectively. The object model and o/c mappings are created (usually by convention) by Reflecting over application assemblies.

The MetadataWorkspace API surface is notoriously difficult to use; it is very hard to figure out how to obtain the metadata you need. This is mostly due to excessive use of inheritance and generic methods meaning that you will often need to cast and/or call methods with a specific generic type to get what you need. Often the best way to figure this out is to run the code and browse the data structures in the VS debugger until you find where the information is stored. You can then work back from that to figure out how to get at the data using the APIs.
<h3>View generation</h3>
The Entity Framework handles queries from the database using “query views” and handles updates to the database using “update views”. These views are generated from the EDM and form the basis of transforming data from the store model to the conceptual model (query views) or the conceptual model to the store model (update views).

The view generation code is responsible for creating these views. This usually happens dynamically the first time a query or update is requested. This can be a slow process and so it is also possible to pre-generate views based on given conceptual and store models and c/s mappings.

View generation code not only creates the views but also validates their correctness at the same time. This is mostly useful for ensuring that data integrity is preserved when data is read from the database and then written back to the database; this is usually referred to as round-tripping.
<h3>Query pipeline</h3>
The query pipeline is responsible for taking a query defined against the object or conceptual models and generating a query tree against the store model. This store query is then given to the EF provider which translates it into SQL that can be directly executed against the database.

Queries are usually created using LINQ but may also come from other sources such as Entity SQL. The query pipeline starts with a query view for the entity set being queried and transforms it through a series of steps into a tree structure called a Canonical Query Tree (CQT). It is this CQT that is passed to the provider. (This is a very simplified view of the query pipeline; a lot of stuff goes on here!)
<h3>Materialization</h3>
Once the provider has executed a query against the database a stream of records will be returned. Sometimes these records are used directly but more often the records are first translated into the objects from the object model. This process of creating objects from records is called materialization.

The materialization code is one of the transition points between object services and the query pipeline (the other being LINQ translation). The query pipeline generates a <em>result assembly</em> along with the CQT that describes how the records should be interpreted to create objects. The materializer uses the result assembly and records to create objects which are then both returned to the application and (usually) passed to the ObjectStateManager for tracking.
<h3>Update pipeline</h3>
The update pipeline is executed when SaveChanges is called. It requests information about all changed objects from the ObjectStateManager and then uses this information together with update views to create the updates (inserts, updates, and deletes) that must be applied to the store model. These updates are passed to the provider which then generates and executes the required database commands.

The update pipeline is responsible for ordering updates to ensure that constraints in the database are not violated—for example, it will ensure that dependent entities are deleted before their principal is deleted. The update pipeline is also responsible for ensuring that the updates succeed and for detecting optimistic concurrency violations.
<h3>Entity client</h3>
The entity client is a mechanism for using EF without an object model. It makes use of the conceptual model and the Entity SQL language to query the database without the need to materialize the results to objects. The entity client is not very commonly used.
<h3>Provider</h3>
The last main EF sub-system is the provider. Most of the EF stack is not tied to any particular database implementation. For example, the query pipeline produces CQTs rather than T-SQL. The EF provider is responsible for taking database-agnostic structures such as CQTs and translating them into code that can be run against a database, such as T-SQL. Applications plug in the appropriate provider for the type of database in use.

The EF provider is also responsible for returning metadata about the database which can be used to create a store model. For example, the provider is responsible for telling EF which database type (e.g. nvarchar) should be used for a given conceptual model type (e.g. string).
<h2>Summary</h2>
In this post I gave a very high-level overview of the Entity Framework architecture. I skimmed over a lot of details but hopefully the information is useful as a starting point for further investigation or asking further questions.

If there is anything here that you need more information about don’t hesitate to contact me or others on the EF team.
