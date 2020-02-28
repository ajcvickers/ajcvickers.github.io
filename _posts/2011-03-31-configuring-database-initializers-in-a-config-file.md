---
layout: default
title: Configuring Database Initializers in a config file
date: 2011-03-31 21:14
day: 31st
month: March
year: 2011
author: ajcvickers
permalink: 2011/03/31/configuring-database-initializers-in-a-config-file/
---

# Entity Framework 4.1
# Configuring Database Initializers in a config file

<a href="https://docs.microsoft.com/archive/blogs/adonet/ef-4-1-release-candidate-available">Entity Framework 4.1</a> introduced the concept of database initializers as a way for your application or tests to perform some actions before your database is used for the first time. On the team blog and in demos we commonly show setting a database initializer through a call to the Database.SetInitializer method. This is certainly an easy way to set an initializer, but it also possible to decouple initializer configuration from your application code by setting them in your app.config or web.config file.

In this post we’ll look at database initializer basics and then show how they can be configured in either code or your config file.

<strong>Update:</strong> Please note that starting with EF 4.3 there is a new syntax for setting initializers in the config file. We introduced this new syntax because as we needed to add more items to the config it became cleaner to create an EntityFramework section rather than adding more key/value pairs to app data. You can still use the syntax described in this post, but you may find the EF 4.3 syntax cleaner. For details see <a href="https://docs.microsoft.com/archive/blogs/adonet/ef-4-3-configuration-file-settings">https://docs.microsoft.com/archive/blogs/adonet/ef-4-3-configuration-file-settings</a>.
<h2>What are database initializers?</h2>
A database initializer is a strategy object that can be registered in the app-domain and will automatically be run the first time that your context is used against a database in that app-domain. They are typically used to check for existence and compatibility of a database, create a database automatically if needed, and/or seed a database with data.

A database initializer must implement the IDatabaseInitializer<TContext> interface where the TContext generic type specifies the context type with which the initializer will be used. This means that if your app or tests have multiple different contexts then you can register an initializer for each of these context types.
<h2>The built-in initializers</h2>
EF 4.1 comes with some pre-defined initializers. These are all contained in the System.Data.Entity namespace in EntityFramework.dll.
<h3>CreateDatabaseIfNotExists</h3>
This is the default initializer for Code First development. It checks if the database referred to by your connection exists and creates it if it does not. If the database exists and contains an EdmMetadata table, then a check is made to determine if the model hash contained in this table matches the current model hash. If this check fails then an exception is thrown letting you know that you probably need to update your database to match the model.
<h3>DropCreateDatabaseIfModelChanges</h3>
This initializer works in basically the same way as CreateDatabaseIfNotExists except that it requires that the EdmMetadata table exists and if the model hashes don’t match then the database is automatically dropped, recreated, and re-seeded. This is initializer is likely to become much less used when full database migration support is added to Code First.
<h3>DropCreateDatabaseAlways</h3>
This initializer drops the database if it exists and always creates a new one the first time that the context is used in the app-domain. This can be useful to ensure that your app or tests always start with a known state. Note that the database is not recreated each time that your context is used, but rather only the first time that the context is used in the app-domain.
<h2>Custom initializers</h2>
It is common to derive from one of the built-in initializer classes and then override the Seed method in your derived initializer. The Seed method is called after the database has been created by each of these initializers and can be used to populate the newly created database with data. For example, the BlogsContextInitializer from the <a href="/2011/03/28/extra-lazy-collection-count-with-ef-4-1-part-2/">Extra-lazy Count post</a> does this.

The Seed method can also be used to perform additional manipulation of the database schema after it has been created. For example, context.Database.ExecuteSqlCommand can be used to add indexes to tables.

Rather than deriving from one of the built-in initializers you can also choose to implement IDatabaseInitializer yourself. This provides complete control over the database initialization process. The methods on context.Database can be very useful in this case to implement your own logic for creating and manipulating the database.
<h2>Setting an initializer in code</h2>
As mentioned above you can use the static SetInitializer method on the Database class to set an initializer.  For example:

``` c#
Database.SetInitializer<BlogContext>(new BlogsContextInitializer());
```
Note that SetInitializer is a generic method and the generic type defines the context type for which the initializer is being set. Often the generic type for the SetInitializer method can be inferred from the generic type of the initializer itself such that you can write:

``` c#
Database.SetInitializer(new BlogsContextInitializer());
```

You can also completely disable the database initializer for a given context type by passing null to the SetInitializer method:

``` c#
Database.SetInitializer<BlogContext>(null);
```

I often call SetInitializer as part of my test fixture setup code such that whatever test(s) I choose to run I know that the appropriate initializer will have been set.
<h2>Setting an initializer in app.config/web.config</h2>
Calling SetInitializer can be okay for tests, but what if I’m building an app and want one initializer in my development environment, a different one for my test environment, and initializers disabled entirely when in production? You could do this by making sure that different code gets compiled/run in each case, but that can get cumbersome and error-prone. A better solution is to leave the code unchanged but setup different config files for each environment, just like you might do for connection strings to development, test, and production databases.

This is the basic syntax for setting an initializer in your config file:

``` xml
<appSettings>
<add key="DatabaseInitializerForType MyAssemblyQualifiedContextType"
value="MyAssemblyQualifedInitializerType" />
</appSettings>"
<add key="DatabaseInitializerForType
            LazyUnicornTests.Model.BlogContext, LazyUnicornTests"
value="LazyUnicornTests.Model.BlogsContextInitializer, LazyUnicornTests" />
</appSettings>
```

This sets the initializer “LazyUnicornTests.Model.BlogsContextInitializer" in assembly “LazyUnicornTests" for the context “LazyUnicornTests.Model.BlogContext" also in assembly “LazyUnicornTests".

You can also disable the initializer for a context type in the config file. For example:

``` xml
<appSettings>
<add key="DatabaseInitializerForType
            LazyUnicornTests.Model.BlogContext, LazyUnicornTests"
value="Disabled" />
</appSettings>
```

If you like you can use the empty string “" instead of the string “Disabled". Both do the same thing.
<h2>Summary</h2>
In this post we looked at the basics of the database initializer concept introduced in EF 4.1 and saw how to set or disable initializers in code or in the application config file.
