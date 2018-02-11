---
layout: post
title: Restoring nuget packages to a local Packages folder again! 
---

I'm a really big fan of the new slim `csproj` files but bundled in with this super sleek `XML` markup is a sneaky change
to the way Nuget packages are loaded.

You'll probably start by noticing that the dreaded `packages.config` file has disappeared! Hooray!

Personally I had always taken issue with this file. The amount of times I'd seen my team use the Visual Studio Nuget GUI
tool and inadvertently install a plethora of unintended packages was frightening.

Part of me doesn't blame them. While the command line syntax isn't particilarly difficult, it doesn't seem to be encouraged. 

So the replacement is the `<PackageReference />` section of your `csproj` file.

I feel this is a much nicer way of working and really lowers the barrier to entry for developers.

I particularly like the way your packages are restored on save. I've already witnessed my team opting to modify their `csproj`
files to include packages, check versions and references over using the GUI tool.

But for all this nice stuff one thing seemed a little odd.

### Why aren't my nuget packages restoring to my Packages folder?

The new location for nuget packages to get restored to is under `%USERPROFILE%\.nuget\packages`. I guess the reasoning
behind this makes sence. I mean, why do you need to have multiple copies of the same packages across your development
machine?

Maybe I've just been burned too many times in the past but I sort of got to like the local `/Packages` cache. The exercise
of deleting your `/bin`, `/obj` and `/Packages` folders when something's playing up all of a sudden becomes just that
little bit more difficult.

Turns out you can restore this functionality if you wish to. If you're familiar with `nuget.config` files then you'll be
pleased to hear it's as simple as adding the new config key `globalPackagesFolder`.

For those of you who aren't, the `nuget.config` file is a simple file you can place along side your `*.sln` file to
specify a number of configuration for your team project.

The below file will restore packages from both the old `packages.config` and the new `<PackageReference />` formats to 
your `.sln` file's parent directory (when placed along side your `.sln` file).

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <config>
    <add key="repositoryPath" value="..\Packages" />
	<add key="globalPackagesFolder" value="..\Packages" />
  </config>
</configuration>
{% endhighlight %}

_NB: you dont need to include this file in your .sln project in any way._NB

More info can be found in the [NuGet.Config reference docs.](https://docs.microsoft.com/en-us/nuget/reference/nuget-config-file)

&mdash; Dan