---
layout: post
title: Unable to Verify connection to Service Fabric cluster.
---

This one caught me out today and it took a little while to diagnose the issue so I thought I'd share my experiences.

A little background:

My team are using **Service Fabric** to host our applications as containers and as the size of our system continues to grow
having a way of locally deploying pre-built applications has become more of a *need to have* rather than a *nice to have*.

Always one to try my hand at some hackery I gave writing a local build and deploy `powershell` module a go.

All was going swimmingly until my local deployment script stumbled across the following issue:

```
> Unable to Verify connection to Service Fabric cluster.
```

The local build tools I had written would recursively loop through a list of applications, jump in and out of a few functions
and dynamically load and run more scripts where needed.

So for brevity this is something similar to what I had

{% highlight bash %}
Import-Module "$ModuleFolderPath\ServiceFabricSDK.psm1"
Write-Host "Establish local cluster connection"
[void](Connect-ServiceFabricCluster)

(Get-Content "${ENV:PROJECT_PATH}/manifest.json") `
    | ConvertFrom-Json `
    | ForEach-Object {
		$project = $_.Name
		$version = $_.Value
        
		Write-Host "Deploying ${project} version ${version}"
		
		$PublishParameters = @{
			# lots of parameters
		}

		Publish-NewServiceFabricApplication @PublishParameters
    }
{% endhighlight %}

When the `Publish-NewServiceFabricApplication` is called, the script throws the error **Unable to Verify connection to Service Fabric cluster.**

In fact, the line of code which breaks can be found in the `Publish-NewServiceFabricApplication.ps1` file found under
`C:\Program Files\Microsoft SDKs\Service Fabric\Tools\PSModule\ServiceFabricSDK`.

Scroll down to `line 170` ish and you'll see the script attempt (and fail) to execute the following function

> Test-ServiceFabricClusterConnection

This had me stumped for a while as the code I was using to deploy had been largly copied and pasted from the given
`Visual Studio` deployment template.

After some learning about `powershell` variable scope and some source code digging it looks like the `$clusterConnection`
variable is incorrectly scoped.

To fix the issue we just need to amend the code above slightly. 

{% highlight bash %}
[void](Connect-ServiceFabricCluster)
$global:clusterConnection = $clusterConnection
{% endhighlight %}

Now when we call `Publish-NewServiceFabricApplication` everything should spring to life as if by magic!

&mdash; Dan