---
layout: default
title: "Code First Migrations: Making __MigrationHistory not a system table"
date: 2012-02-27 21:40
day: 27th
month: February
year: 2012
author: ajcvickers
permalink: 2012/02/27/code-first-migrations-making-__migrationhistory-not-a-system-table/
---

# Entity Framework 4.3
# Code First Migrations
# Making __MigrationHistory not a system table

<p><a href="https://docs.microsoft.com/archive/blogs/adonet/ef-4-3-released">Code First Migrations</a> uses a table called <a href="/2012/01/13/ef-4-3-beta-1-what-happened-to-that-edmmetadata-table/">__MigrationHistory</a> as a place to store metadata about the migrations that have been applied to the database. Code First creates this table when it creates a database or when migrations are enabled. In addition, when running against Microsoft SQL Server, Code First will also mark the table as a system table. However, several times recently the question has come up of how to make the table a normal user table instead of a system table. This is pretty easy to do and this post describes how.</p><p>Migrations doesn't actually care whether or not __MigrationHistory is a system table. Indeed, with some providers, such as SQL Server Compact, the table is never marked as a system table. The reason it is marked as a system table on SQL Server is simply to keep it out of the way such that it doesn't clutter up the view of your normal tables.</p>  <p>However, sometimes having __MigrationHistory as a system table can be a problem. For example, current versions of Web Deploy don't deal well with system tables. The Web Deploy team are working on supporting Migrations but until this work is released you may want to make __MigrationHistory a normal user table.</p>  <h3>For new databases</h3>  <p>One way to make sure that __MigrationHistory is not created as a system table is to override the Migrations SQL generator for SQL Server. This only works if you do it before the table has been created, since Code First only tries to mark the table as a system table as part of creating the table. In other words, this method is only usually suitable for new databases where you haven't yet performed any migrations.</p>  <p>To override the SQL generator, create a class that derives from <a href="http://msdn.microsoft.com/en-us/library/system.data.entity.migrations.sql.sqlservermigrationsqlgenerator(v=vs.103).aspx">SqlServerMigrationSqlGenerator</a> and override the <a href="http://msdn.microsoft.com/en-us/library/system.data.entity.migrations.sql.sqlservermigrationsqlgenerator.generatemakesystemtable(v=vs.103).aspx">GenerateMakeSystemTable</a> method so that it does nothing. For example:</p>  

``` c#
public class NonSystemTableSqlGenerator : SqlServerMigrationSqlGenerator
{
    protected override void GenerateMakeSystemTable(
        CreateTableOperation createTableOperation)
    {
    }
}
```
<p>Now set an instance of this new class in your Migrations Configuration:</p> 

``` c#
public Configuration()
{
    AutomaticMigrationsEnabled = false;
    SetSqlGenerator("System.Data.SqlClient", new NonSystemTableSqlGenerator());
}
```
  
<h3>For existing databases</h3>  <p>If you have an existing __MigrationHistory table and want to make it non-system, then you'll have to do some work in SQL. The following worked for me although there are plenty of other ways to write the SQL that would have the same end result:</p>  

``` sql
SELECT *
INTO [TempMigrationHistory]
FROM [__MigrationHistory]

DROP TABLE [__MigrationHistory]

EXEC sp_rename 'TempMigrationHistory', '__MigrationHistory'
``` 
  
<p>And that's it—you don't have to have __MigrationHistory as a system table if you don't want.</p>  
