---
layout: post
title: Setting up a test database with Entity Framework core is ridiculously easy
---

For years I've been advocating that you deploy your database in your integration tests.

It gives your team the confidence that everything is running as expected without some wolly _mocked_ or 
_faked_ data access layer.

Recently I've been working with `Entity Framework Core` and once again came the task of automating
a database deployment for my integration test run.

And for once it couldn't be easier!

{% highlight csharp %}
[TestFixture]
public class FooTests
{
    private const string DatabaseName = "My.Service.TestRun";
    private static string ConnectionString => $@"Server=(localdb)\mssqllocaldb;Database={DatabaseName};Trusted_Connection=True;MultipleActiveResultSets=true";

    [OneTimeSetUp]
    public async Task SetUp()
    {
        var optionsBuilder = new DbContextOptionsBuilder<MyContext>();
        optionsBuilder.UseSqlServer(ConnectionString);

        var dbContext = new MyContext(optionsBuilder.Options);
        await dbContext.Database.EnsureDeletedAsync();
        await dbContext.Database.EnsureCreatedAsync();
    }
}
{% endhighlight %}

_NB: using NUnit syntax._

Kudos to the EF Core team on this one :D

_See also: [Using localdb with VSTS](/2018/02/09/using-local-db-with-vsts)_ 

&mdash; Dan