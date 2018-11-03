---
layout: post
title: Akka.net - Actor system is now quarantined
---

Whilst developing earlier today I encountered an issue with my environment which left me a little scared...

```
[17:49:36 WRN] Association to [akka.tcp://ctrlaltdan@10.240.0.33:8081] having UID [333473583] is irrecoverably failed. UID is now quarantined and all messages to this UID will be delivered to dead letters. Remote actorsystem must be restarted to recover from this situation.
[17:49:36 INF] Shutting down remote daemon.
[17:49:36 INF] Remote daemon shut down; proceeding with flushing remote transports.
[17:49:36 WRN] Association with remote system akka.tcp://ctrlaltdan@10.240.0.33:8081 has failed; address is now gated for 5000 ms. Reason is: [Akka.Remote.EndpointDisassociatedException: Disassociated
```

What was scary is the state my application was now left in.

My API continued to be responsive even though the actor system was now "dead" or should I say "must be restarted" as per the error message.

I guess this scenario is far from ideal but realistically it's something to be expected. 

At this stage I'm not entirely sure why my actor system became `Disassociated` but I can appreciate that network blips, bad code, GC locks might all contribute to events like these in the future.

My real issue here isn't my actor system shutting down, it's my API continuing to accept requests.

So how can we hook into the cluster chat to determine when I've been quarantined?

Enter: the terminator.

{% highlight csharp %}
public class Terminator : ReceiveActor
{
    public Terminator()
    {
        Receive<QuarantinedEvent>(m => 
        {
            Actors.Instance.Shutdown();
        });
    }
}
{% endhighlight %}

Terminator in my case is a simple Actor ready to receive `QuarantinedEvent` messages delivered to my unhealthy cluster node.

Here I'm using a small wrapper around the ActorSystem called `Actors`. This object gives me some control around the actor system lifecycle and I use it to invoke the `CoordinatedShutdown` method as recommended by the Akka.net team.

Having a teminator in your code is all well and good but we need to make sure he's protecting our node from unwanted ~~Cyborg attacks~~ quarantined events. 

Let's add the following to our actor system setup.

{% highlight csharp %}
var arnold = system.ActorOf(Props.Create<Terminator>());
system.EventStream.Subscribe(arnold, typeof(QuarantinedEvent));
{% endhighlight %}

And that's all that's needed.

For clarity my `Program.cs` looks similar to this. You can see my application will exit if either API or the actor system exit.

Events are bound to both the cancel key press and assembly unloading to give cross windows development/docker (+ kubernetes) support.

{% highlight csharp %}
public class Program
{
    public static void Main(string[] args)
    {
        var host = CreateWebHostBuilder(args).Build();

        Actors
            .Bootstrap()
            .Build();

        Console.CancelKeyPress += (sender, eventArgs) => Shutdown(host);
        AssemblyLoadContext.Default.Unloading += _ => Shutdown(host);

        var webapiShutdown = host.RunAsync();
        var actorSystemShutdown = Actors.Instance.StayAlive();

        Task
            .WhenAny(webapiShutdown, actorSystemShutdown)
            .Wait();

        Shutdown(host);
    }
    
    private static void Shutdown(IWebHost host)
    {
        host.StopAsync(TimeSpan.FromSeconds(5)).Wait();
        Actors.Instance.Shutdown();
    }

    public static IWebHostBuilder CreateWebHostBuilder(string[] args) => WebHost
        .CreateDefaultBuilder(args)
        .UseKestrel()
        .UseStartup<Startup>();
}
{% endhighlight %}

&mdash; Dan