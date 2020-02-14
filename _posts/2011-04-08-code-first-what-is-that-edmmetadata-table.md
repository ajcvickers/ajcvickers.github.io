---
layout: post
title: Code First: What is that EdmMetadata table?
date: 2011-04-08 22:04
author: ajcvickers
comments: true
categories: [Code First, Database Initializers, DbContext, DbContext API, EdmMetadata, Entity Framework]
---
<p>This is one of those posts that may only be useful for a short time since we already have a group of people working on migrations support for Code First. That being said I’ve been asked twice in the corridors of Building 18 recently how to manually create the model hash that Code First stores in the EdmMetadata table so I thought a few words about that strange EdmMetadata thing might be generally useful.</p><!--more--><p>The EdmMetadata table is a simple way for Code First to tell if the model used to create a database is the same model that is now being used to access the database. As of <a href="http://blogs.msdn.com/b/adonet/archive/2011/03/15/ef-4-1-release-candidate-available.aspx">EF 4.1</a> the only thing stored in the table is a single row containing a hash of the SSDL part of the model used to create the database.</p>  <p>(Geek details: when you look in an EDMX file, the SSDL is the part of that file that represents the database (store) schema. This means that the EdmMetadata model hash only changes if the database schema that would be generated changes; changes to the conceptual model (CSDL) or the mapping between the conceptual model and the database (MSL) will not affect the hash.)</p>  <h2>Checking if the model hash matches</h2>  <p>You might want to <a href="http://blog.oneunicorn.com/2011/03/31/configuring-database-initializers-in-a-config-file/">write your own database initializer</a> that checks whether or not the hash stored in the database matches that of the current model. The easiest way to do this is to use the CompatibleWithModel method on Database. For example, you could do something similar to DropCreateDatabaseIfModelChanges with code like this:</p>  <div style="display:inline;float:none;margin:0;padding:0;" id="scid:C89E2BDB-ADD3-4f7a-9810-1B7EACF446C1:ae004add-3c80-44b5-8912-48b583e7c5cf" class="wlWriterEditableSmartContent"><pre>
[sourcecode language="csharp"]
public void InitializeDatabase(TContext context)
{
    if (context.Database.Exists())
    {
        if (context.Database.CompatibleWithModel(throwIfNoMetadata: true))
        {
            return;
        }

        context.Database.Delete();
    }

    // Database didn't exist or we deleted it, so we now create it again.
    context.Database.Create();

    Seed(context);
    context.SaveChanges();
}
[/sourcecode]
</pre>
</div>

<p>Note that by default the model hash is written to the database any time that Database.Create() is called.</p>

<h2>Reading the raw hash value </h2>

<p>When the EdmMetadata table is in use it is mapped to the EdmMetadata class in your Code First model. This means that you can query for the value in the table using something like this:</p>

<div style="display:inline;float:none;margin:0;padding:0;" id="scid:C89E2BDB-ADD3-4f7a-9810-1B7EACF446C1:92ceb60f-7478-4a00-9a19-544ec6b4e6d3" class="wlWriterEditableSmartContent"><pre>
[sourcecode language="csharp"]
public static class EdmMetadataExtensions
{
    public static string QueryForModelHash(this DbContext context)
    {
        return context.Set&lt;EdmMetadata&gt;().AsNoTracking().Single().ModelHash;
    }
}

[/sourcecode]
</pre>
</div>

<p>This code assumes that the table exists and contains data; if this might not be the case then you should add some more checking that the query succeeds and returns valid data.</p>

<p>Of course, you could instead write raw SQL (<a href="http://blogs.msdn.com/b/adonet/archive/2011/02/04/using-dbcontext-in-ef-feature-ctp5-part-10-raw-sql-queries.aspx">using Database.SqlQuery</a>) to access this data if you wanted to.</p>

<h2>Getting the hash of the current model</h2>

<p>Sometimes you might want more control over the model hash than Database.CompatibleWithModel or Database.Create gives you. For example, you might want to write the model hash into a database that was created or modified by some means other than the Database.Create method. You could do this using code something like this:</p>

<div style="display:inline;float:none;margin:0;padding:0;" id="scid:C89E2BDB-ADD3-4f7a-9810-1B7EACF446C1:fe326cbb-cb0f-4d8b-9f90-928abdbcc95f" class="wlWriterEditableSmartContent"><pre>
[sourcecode language="csharp"]
    public static void SaveModelHashToDatabase(this DbContext context)
    {
        // Create an EdmMetadata entity containing the model hash for
        // the given context.
        var edmMetadata = new EdmMetadata
        {
            ModelHash = EdmMetadata.TryGetModelHash(context)
        };
        
        // Insert the model hash into the database.
        context.Set&lt;EdmMetadata&gt;().Add(edmMetadata);
        context.SaveChanges();
    }
}

[/sourcecode]
</pre>
</div>

<p>This code uses the static TryGetModelHash method on the EdmMetadata type to get the model hash of the given context. (Note that you can only get a model hash for a context that has been created using Code First; getting a hash is currently not supported for Database First or Model First contexts, although you could create an SHA-256 hash of the SSDL in the EDMX file yourself if you really wanted it.)</p>

<p>The code above then sets the model hash into an EdmMetadata object and this object is saved to the database. Depending on what you then want to do with the context you might want to detach the EdmMetadata entity from the context; if you’re using this code in a database initializer this is probably not necessary since the initializer effectively does this for you.</p>

<p>Again, you could choose to save the data to the table using raw SQL instead of the mapped EdmMetadata entity type.</p>

<h2>But I don’t want EdmMetadata at all!</h2>

<p>In the CTP4 versions of what is now EF 4.1 the mapping of the EdmMetadata table by default could be quite problematic, especially when using Code First against an existing database. Since CTP5 this has been much less of a problem, but there may be times when you still don’t want this entity added to your model. You can prevent DbContext from adding it by removing IncludeMetadataConvention in your OnModelCreating method. For example:</p>

<div style="display:inline;float:none;margin:0;padding:0;" id="scid:C89E2BDB-ADD3-4f7a-9810-1B7EACF446C1:af238d1a-49ef-4c67-b6fb-c1f2daf7e59f" class="wlWriterEditableSmartContent"><pre>
[sourcecode language="csharp"]
protected override void OnModelCreating(DbModelBuilder modelBuilder)
{
    modelBuilder.Conventions.Remove&lt;IncludeMetadataConvention&gt;();
}
[/sourcecode]
</pre>
</div>

<p>So there you are, probably more details on EdmMetadata than you ever wanted. Hope some people find it useful.</p>

<p>Thanks for reading!
  <br />Arthur</p>
