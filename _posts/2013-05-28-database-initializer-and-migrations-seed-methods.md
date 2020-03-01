---
layout: default
title: "Database initializer and Migrations Seed methods"
date: 2013-05-28 10:20
day: 28th
month: May
year: 2013
author: ajcvickers
permalink: 2013/05/28/database-initializer-and-migrations-seed-methods/
---

# Entity Framework 6.0
# Database initializer and Migrations Seed methods

Entity Framework contains two different methods both called Seed that do similar things but behave slightly differently. The first was introduced in EF 4.1 and works with database initializers. The second was introduced in EF 4.3 as part of Code First Migrations. This post describes how these two methods are used, when they are called, and how they differ from each other.
<h2>The basic idea</h2>
Regardless of the specific Seed method being used the general idea of a Seed method is the same. It is a way to get initial data into a database that is being created by Code First or evolved by Migrations. This data is often test data, but may also be reference data such as lists of known countries, states, etc.

In both cases the Seed method is a virtual <a href="http://en.wikipedia.org/wiki/Template_Method">Template Method</a> that is overridden by application code to write seed data into the database.
<h2>Database initializers</h2>
Starting with EF 4.1 Code First and DbContext can be used to create a database for you. This behavior is encapsulated in objects called “database initializers” that implement the IDatabaseInitializer interface. Database initializers run the first time that a DbContext is used and can do things like check if a database already exists and create a new database if needed.
<h3>DropCreateDatabaseIfModelChanges</h3>
Several IDatabaseInitializer implementations are included in the EntityFramework assembly. Let's take <a href="http://msdn.microsoft.com/en-us/library/gg679604(v=vs.103).aspx">DropCreateDatabaseIfModelChanges</a> as an example since it defines a <a href="http://msdn.microsoft.com/en-us/library/gg679410(v=vs.103).aspx">Seed method</a> and is the most interesting initializer with regards to this discussion. This initializer does the following:
<ul>
	<li>Checks whether or not the target database already exists</li>
	<li>If it does, then the current Code First model is compared with the model stored in metadata in the database</li>
	<li>The database is dropped if the current model does not match the model in the database</li>
	<li>The database is created if it was dropped or didn't exist in the first place</li>
	<li>If the database was created, then the initializer Seed method is called</li>
</ul>
This initializer and others like it are intended to be used during initial development of an application where no real data exists yet, or for tests that run against a test database that can be dropped and re-created at will. You can find many uses of this initializer in the open source EF tests on <a href="https://entityframework.codeplex.com/">CodePlex</a>.
<h3>Database initializer Seed</h3>
With respect to Seed the important thing to notice is that Seed is only ever called immediately after a new, empty database has just been created. Seed is never called for an existing database that might already have data in it. This has two important consequences:
<ul>
	<li>Database initializer Seed methods do not have to handle existing data. That is, new entities can be inserted without any need to check whether or not the entities already exist in the database.</li>
	<li>The Seed method will not be called when the application is run if the database already exists and the model has not changed since the last run. We'll come back to this point later.</li>
</ul>
<h2>Enter Migrations</h2>
EF 4.3 introduced <a href="http://msdn.microsoft.com/en-us/data/jj591621">Code First Migrations</a>. Migrations provide a way for the database to be evolved without needing to drop and recreate the entire database. Use of Migrations commonly involves using PowerShell commands to manage updates to the database explicitly. That is, database creation and updates are usually handled during development from PowerShell and do not happen automatically when the applications runs. (See <em>The Migrations initializer </em>below for how this can be changed.)
<h3>Migrations Seed</h3>
Migrations introduced its own <a href="http://msdn.microsoft.com/en-us/library/hh829453(v=vs.103).aspx">Seed method</a> on the <a href="http://msdn.microsoft.com/en-us/library/hh829093(v=vs.103).aspx">DbMigrationsConfiguration class</a>. This seed method is different from the database initializer Seed method in two important ways:
<ul>
	<li>It runs whenever the Update-Database PowerShell command is executed. Unless the Migrations initializer is being used the Migrations Seed method will <strong>not </strong>be executed when your application starts.</li>
	<li>It must handle cases where the database already contains data because Migrations is evolving the database rather than dropping and recreating it.</li>
</ul>
This second point is really important and is the reason why the <a href="http://msdn.microsoft.com/en-us/library/hh846521(v=vs.103).aspx">AddOrUpdate extension method</a> is included with Migrations. This method can check whether or not an entity already exists in the database and then either insert a new entity if it doesn't already exist or update the existing entity if it does exist.
<h2>Seeding when the model hasn't changed</h2>
In the section on database initializers I mentioned that the initializer Seed method will not be called if the database already exists and the model has not changed. This often turned out to be quite an inconvenience. Consider adding a new entity to the model and then running the application without remembering to update the Seed method. The database is dropped and recreated with a table for the new entity. However the new table is empty. So now you update the Seed method and run again…but the table is still empty because the model has not changed since the last run.

People would usually work around this by either:
<ul>
	<li>Making a temporary artificial change to the model</li>
	<li>Switching to DropCreateDatabaseAlways, with the consequence that the database is often dropped and recreated when it is not needed</li>
	<li>Manually deleting the database</li>
</ul>
<h3>The Migrations situation</h3>
So what should happen if you are using Migrations in a similar situation? The analogous case is that a new entity is added, a migration is created for it, and Update-Database is used to apply the migration without remembering to update the Seed method. As before, the new table is empty and you realize this, so now you update the Migrations Seed method to AddOrUpdate data into the new table. You now run Update-Database again; should the Seed method run?

If we were following the database initializers pattern then the Seed method would not run because the model has not changed since Update-Database was called last time. In other words, there is no new migration to apply. You would then need to create some sort of artificial migration just to get the Seed method to run—note that just deleting the database doesn't work in this case since the database is being evolved by Migrations.

However, since the Seed method must be able to handle existing data anyway why not just run the Seed method when Update-Database is executed regardless of whether or not there is a migration to apply? This is indeed what happens and it means that Seed can be updated and run at anytime without a change to the model being needed.
<h2>The Migrations initializer</h2>
The two worlds of database initializers and Migrations come together with the <a href="http://msdn.microsoft.com/en-us/library/hh829293(v=vs.103).aspx">MigrateDatabaseToLatestVersion initializer</a>. This is an IDatabaseInitializer implementation that uses your DbMigrationsConfiguration to programmatically run Update-Database when the application starts.

Since Update-Database causes the DbMigrationsConfiguration.Seed method to be called it follows that using this initializer causes that Seed method to be called. And given that Seed is always called when Update-Database is executed it means that Seed will be called every time that the application is started, regardless of whether or not any migrations were actually applied. So when using the Migrations initializer you never need to do anything artificial to get Seed to run.
<h3>Long Seed methods</h3>
As discussed above, the Migrations Seed method must handle a database that already contains data and the AddOrUpdate method is often used for this purpose. However, the AddOrUpdate method makes an intentional trade-off between ease-of-use in a Seed method and efficiency. This is because it queries the database each time it is called to get any already existing entity.

This behavior is fine for short Seed methods or even for Seed methods that are only run manually when Update-Database is executed in PowerShell. However, it can become a problem when a long running Seed method is used with the Migrations initializer since the Seed method is run every time the application starts.

The best way to handle this is usually to not use AddOrUpdate for every entity, but to instead be more intentional about checking the database for existing data using any mechanisms that are appropriate. For example, Seed might check whether or not one representative entity exists and then branch on that result to either update everything or insert everything.

There is also this <a href="https://entityframework.codeplex.com/workitem/843">CodePlex work item</a> to allow the Migrations Seed method to be run only when a migration is actually applied. This can help with long running Seed methods, but keep in mind that, once this is implemented, switching it on will likely lead to situations where something special has to be done to run Seed because Seed changed but the model did not.
<h2>Summary</h2>
Both database initializers and Migrations have Seed methods which can be used to seed a database with initial data. However, the original initializer Seed methods only run immediately after database creation, whereas the Migrations Seed method runs anytime Update-Database is used.
