---
layout: post
title: Creating custom serilog enrichers
---

If like me you think Serilog is pretty fantastic then you might have wanted to extend your implementation with custom enrichers.

Below are a couple templates you can use to dynamically enrich your logs with whatever properties you need.

In my example I'm enriching my logs with the property `ReleaseNumber` given the environment variable `RELEASE_NUMBER`.

### 1. Create the log event enricher

This is the real meat when it comes to creating a custom Serilog enricher. The two important aspects are setting a constant `PropertyName` value
and providing a means to resolve the value of that property.

{% highlight csharp %}
public class ReleaseNumberEnricher : ILogEventEnricher
{
    LogEventProperty _cachedProperty;

    public const string PropertyName = "ReleaseNumber";

    /// <summary>
    /// Enrich the log event.
    /// </summary>
    /// <param name="logEvent">The log event to enrich.</param>
    /// <param name="propertyFactory">Factory for creating new properties to add to the event.</param>
    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        logEvent.AddPropertyIfAbsent(GetLogEventProperty(propertyFactory));
    }

    private LogEventProperty GetLogEventProperty(ILogEventPropertyFactory propertyFactory)
    {
        // Don't care about thread-safety, in the worst case the field gets overwritten and one property will be GCed
        if (_cachedProperty == null)
            _cachedProperty = CreateProperty(propertyFactory);

        return _cachedProperty;
    }

    // Qualify as uncommon-path
    [MethodImpl(MethodImplOptions.NoInlining)]
    private static LogEventProperty CreateProperty(ILogEventPropertyFactory propertyFactory)
    {
        var value = Environment.GetEnvironmentVariable("RELEASE_NUMBER") ?? "local";
        return propertyFactory.CreateProperty(PropertyName, value);
    }
}
{% endhighlight %}


### 2. Provide a means to add your enricher to your logger configuration

{% highlight csharp %}
public static class LoggingExtensions
{
    public static LoggerConfiguration WithReleaseNumber(
        this LoggerEnrichmentConfiguration enrich)
    {
        if (enrich == null)
            throw new ArgumentNullException(nameof(enrich));

        return enrich.With<ReleaseNumberEnricher>();
    }
}
{% endhighlight %}

### 3 Time to use your enricher!

There's two ways we can go about using a custom enricher. First is probably the simplest and most commonly adopted way.

#### Builder style

Along with any other setup, here we can add our own enricher to the configuration builder.

{% highlight csharp %}
var logger = new LoggerConfiguration()
  .Enrich.WithReleaseNumber()
  .CreateLogger();
{% endhighlight %}

#### Configuration style

The alternative way is to import your enricher using an `appsettings.json` file using the `Serilog.Settings.Configuration` package.

This package will scan all assemblies listed in the `Using` property (as well as any assemblies belonging to the `Serilog` namespace)
for methods extending the `LoggerEnrichmentConfiguration` object. If your enricher isn't being applied then double check the `Using` property
matches the `dll` where your extension method resides.  

{% highlight csharp %}
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Debug"
    },
    "WriteTo": [
      {
        "Name": "Console"
      }
    ],
    "Using": [ "Example.Assembly.Name" ],
    "Enrich": [ "FromLogContext", "WithReleaseNumber" ]
  }
}
{% endhighlight %}


&mdash; Dan