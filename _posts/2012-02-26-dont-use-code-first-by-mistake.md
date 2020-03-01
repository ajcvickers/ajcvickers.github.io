---
layout: default
title: "Don't use Code First by mistake"
date: 2012-02-26 16:29
day: 26th
month: February
year: 2012
author: ajcvickers
permalink: 2012/02/26/dont-use-code-first-by-mistake/
---

# Entity Framework 4.3
# Don't use Code First by mistake

<p><a href="https://docs.microsoft.com/archive/blogs/adonet/ef-4-2-code-first-walkthrough">EF Code First</a> is great and I use it all the time, even when <a href="https://docs.microsoft.com/archive/blogs/adonet/when-is-code-first-not-code-first">mapping to existing databases</a>. However, if your intention is to use a Database First flow, then it's important that you don't start to use Code First unintentionally. If you do, then you might end up with exceptions like those described in <a href="http://stackoverflow.com/questions/9441892/entity-framework-4-3-looking-for-singular-name-instead-of-plural-when-entity">this Stack Overflow question</a>.</p><p>Code First can be used both <a href="https://docs.microsoft.com/archive/blogs/adonet/ef-4-2-code-first-walkthrough">when you want the database to be created from your entity</a> classes and <a href="https://docs.microsoft.com/archive/blogs/adonet/when-is-code-first-not-code-first">when mapping your entity classes to an existing database</a>. But EF also supports another way of mapping an existing database to an object model—this is usually referred to as the Database First approach or workflow. The EF designer in Visual Studio takes this approach when the new Entity Data Model wizard is used and “Generate from database” is chosen. There is a <a href="https://docs.microsoft.com/en-us/archive/blogs/adonet/ef-4-2-code-first-walkthrough">walkthrough for this on the EF team blog</a> for EF 4.2—the same walkthrough can also be used for EF 4.3.</p>  <h3>How Database First works</h3>  <p>These two approaches differ fundamentally in the way DbContext behaves when the application is run. With the Database First approach an EDMX file is created by the EF Designer and (usually) embedded in the application assembly. This EDMX file contains all the information required to map between the entity classes and the database. For example, if one of the entity classes is called “User” but the corresponding table in the database is called “t_userdata”, then this mapping is included in the EDMX file. Such mapping is usually configured using the EF Designer but can also be added through editing the XML in the EDMX file directly.</p>  <p>When a Database First application is run the DbContext must load this EDMX file so that it knows how to map between entity classes and the database. This is done through a special <a href="http://msdn.microsoft.com/en-us/library/cc716756(v=vs.100).aspx">EF connection string</a> which is created for you and added to your config file by the EF Designer. The connection string will look something like this:</p>

``` xml
<connectionStrings>
  <add name="MyEntities"
  connectionString="metadata=res://*/MyModel.csdl|res://*/MyModel.ssdl|res://*/MyModel.msl;
    provider=System.Data.SqlClient;provider connection string=&quot;
    datasource=.\sqlexpress;initial catalog=MyEntities;
    integrated security=True;multipleactiveresultsets=True;
    App=EntityFramework&quot;"
  providerName="System.Data.EntityClient" />
</connectionStrings>
```

Notice that this connection string contains references to “CSDL”, “MSL”, and “SSDL” metadata. This is the contents of the EDMX that has been embedded in the application assembly.</p>  <h3>How Code First works</h3>  <p>With the Code First approach there is no EDMX file. There is nothing in the VS project that contains the mapping information from entity classes to database…except for the code itself.</p>  <p>When a Code First application is run the DbContext <a href="/2011/04/15/code-first-inside-dbcontext-initialization/">constructs the mapping to the database the first time that the context is used</a>. It does this through a combination of convention, configuration by data annotations, and configuration using the Code First DbModelBuilder fluent API. Taking the example above again, by convention Code First will map the User class to a table called “Users” in the database. If the table is instead called “t_userdata” then this must be specified either using the <a href="http://msdn.microsoft.com/en-us/library/system.componentmodel.dataannotations.tableattribute(v=VS.103).aspx">Table</a> data annotation or through a call to the <a href="http://msdn.microsoft.com/en-us/library/gg679488(v=VS.103).aspx">ToTable</a> method.</p>  <h3>How it can go wrong</h3>  <p>Imagine you are using the Database First approach and have an EDMX containing all your mapping information. Now consider what happens if you run your application and it fails to find this EDMX but instead uses Code First to create mappings by convention. There won't be anything in your code to map the User class to the t_userdata table. So the model created by Code First will assume that the table is called “User” and will attempt (but fail) to run queries against that table.</p>  <h3>How to prevent it going wrong</h3>  <p>When you use the Database First approach with the <a href="http://visualstudiogallery.msdn.microsoft.com/7812b04c-db36-4817-8a84-e73c452410a2">DbContext T4 templates</a> two things are setup to protect against things going wrong in this way.</p>  <p>First, the generated context class makes a call to the base DbContext constructor specifying the name of this connection string. For example: </p>  

``` c#
public MyEntities()
: base("name=MyEntities")
{
}
```

<p>This tells DbContext to find and use the "MyEntities" connection string in the config—i.e. the one created by the designer as described above.</p>  <p>Using "name=" means that DbContext will throw if it doesn't find the connection string—it won't just go ahead and create a connection by convention. Be very careful if you change this call to the base constructor. Make sure that whatever change you make DbContext is still able to find the correct connection string containing the information from your EDMX. If DbContext finds a non-EF connection string or creates a connection string by convention then it will start using Code First to create a model for that connection.</p>  <p>The second thing that happens is that the <a href="http://msdn.microsoft.com/en-us/library/system.data.entity.dbcontext.onmodelcreating(v=vs.103).aspx">OnModelCreating</a> is overridden in the generated context and made to throw: </p>  


``` c#
protected override void OnModelCreating(DbModelBuilder modelBuilder)
{
    throw new UnintentionalCodeFirstException();
}
```

<p>To see why this happens consider how OnModelCreating is used; OnModelCreating is a way of making calls to the Code First DbModelBuilder fluent API. In other words, it's a way of setting up Code First mappings. This means that OnModelCreating <em>will never be called when using the Database First approach</em>. It will never be called because all the mappings already exist in the EDMX and so Code First and the DbModelBuilder are never used.     <br /></p>  <p>The message from the exception reads:</p>  <blockquote>   <p>Code generated using the T4 templates for Database First and Model First development may not work correctly if used in Code First mode. To continue using Database First or Model First ensure that the Entity Framework connection string is specified in the config file of executing application. To use these classes, that were generated from Database First or Model First, with Code First add any additional configuration using attributes or the DbModelBuilder API and then remove the code that throws this exception.</p> </blockquote>  <p>This is message is an attempt to distil the contents of this post into an exception message. A hard job!</p>  <h3>But what if I intended to use Code First?</h3>  <p>Now, it might be that you want to use Code First to map to an existing database. As I stated at the top of this post, using Code First in this way is a great pattern and fully supported. However, you will need to make sure mappings are setup appropriately for this to work. Don't just delete the code that throws this exception and expect things to work. You'll also need to make sure that you setup any mappings to the database correctly, using data annotations and/or the fluent API.</p>
