---
layout: post
title: Why is EF Core adding 1 to my column names?
---

Today I came across a really annoying issue which practically wiped out my whole day! >.<

I was undertaking an ambitious project whereby I'd map some pre-existing SQL tables to a new set of domain classes when I came across the following error...

<error>
    <small><b>System.Data.SqlClient.SqlException :</b></small>
    Invalid column name 'OwnerId1'.<br/>
   at System.Data.SqlClient.SqlCommand.<>c.<ExecuteDbDataReaderAsync>b__180_0(Task`1 result)<br/>
   at System.Threading.Tasks.ContinuationResultTaskFromResultTask`2.InnerInvoke()<br/>
   at System.Threading.Tasks.Task.Execute()
</error>

For some context the domain classes I wrote were abstract with navigation properties on both the abstract classes, as well as the derrived classes.

To save your sanity I have recreated a very basic similar set of classes for your eye balls.

{% highlight cs %}
public class Car : Vehicle
{
  public int NumberOfSeats { get; set; }
}

public class Motorbike : Vehicle
{
  public decimal CoolnessFactor { get; set; }
}

public abstract class Vehicle
{
  public Guid Id { get; set; }

  public Guid OwnerId { get; set; }
  public Owner Owner { get; set; }
}

public class Owner
{
  public Guid Id { get; set; }
  public string Name { get; set; }
}
{% endhighlight %}

And the following is how the above classes were **incorrectly** configured.

{% highlight cs %}
public class VehicleContext : DbContext
{
  private const string ConnectionString = @"Server=(localdb)\mssqllocaldb;Database=PropertyNameTest;Trusted_Connection=True;MultipleActiveResultSets=true";

  private static DbContextOptions Options => new DbContextOptionsBuilder<VehicleContext>()
    .UseSqlServer(ConnectionString)
    .Options;

  public VehicleContext()
    : base(Options)
  {
  }

  protected override void OnModelCreating(ModelBuilder modelBuilder)
  {
    modelBuilder.Entity<Vehicle>(vehicle =>
    {
      vehicle.HasKey(x => x.Id);

      vehicle.HasOne<Owner>()
        .WithMany()
        .HasForeignKey(x => x.OwnerId);

      vehicle.HasDiscriminator()
        .HasValue<Car>("CAR")
        .HasValue<Motorbike>("BIKE");
    });

    modelBuilder.Entity<Owner>(owner =>
    {
      owner.HasKey(x => x.Id);
    });
  }
}
{% endhighlight %}

So just to be clear this is what the SQL table would look if you were to allow Entity Framework to deploy the above domain model.

<img src="/images/posts/ef-column1.png"/>

I never created a property called `OwnerId1`?!

I tried all the _seemingly_ obvious things.

I added explicit column name fluent mappings. Nothing.

I added a reverse navigation property. Nothing.

So how do we fix this happy little miracle?

Well it turned out to be such a simple fix. The problem was with how my `DbContext` had been configured.

{% highlight cs %}
vehicle.HasOne<Owner>(x => x.Owner)
  .WithMany()
  .HasForeignKey(x => x.OwnerId);
{% endhighlight %}

facepalm.gif

In fact digging into the `EntityTypeBuilder` source code explicity outlines this case.

<info>
    <p><small><b>Summary:</b></small></p>
    <p>Configures a relationship where this entity type has a reference that points
    to a single instance of the other type in the relationship.</p>
    <p>Note that calling this method with no parameters will explicitly configure this
    side of the relationship to use no navigation property, even if such a property
    exists on the entity type. If the navigation property is to be used, then it
    must be specified.</p>
</info>

I'm not 100% sure why the navigation expression defaults to null but hey ho.

&mdash; Dan