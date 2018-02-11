---
layout: post
title: Using localdb with VSTS
---

Today I had my devops hat on and got to work with `Visual Studio Team Services` automated builds.

I was really hoping that the default `Hosted VS2017` build agent had a copy of `SQL Server` or `SQL Express` installed
so that my integration test library would be able to deploy and execute a number of tests.

<error>
    <small><b>ERROR</b></small>
    <p>System.Data.SqlClient.SqlException : Connection Timeout Expired.  The timeout period elapsed while attempting to consume the pre-login handshake acknowledgement.  This could be because the pre-login handshake failed or the server was unable to respond back in time.  The duration spent while attempting to connect to this server was - [Pre-Login] initialization=141279; handshake=79;</p>
    <p>System.ComponentModel.Win32Exception : The wait operation timed out</p>
</error>

After a moment of despair and a frantic Google I sussed that `SQL Express` should indeed be installed, so why the Connection Timeout?

Turns out `SQL Express` isnt always _started_ when your build agent starts.

I managed to fix this by introducing a `PowerShell` step into my VSTS `Build Definition`.

```
PowerShell
Version: 1.*
Display name: Start localdb
Type: inline script
Arguments:
Inline Script:

sqllocaldb start mssqllocaldb.

```

And hey presto - we have a `localdb`!

```
##[section]Starting: Start localdb
==============================================================================
Task         : PowerShell
Description  : Run a PowerShell script
Version      : 1.2.3
Author       : Microsoft Corporation
Help         : [More Information](https://go.microsoft.com/fwlink/?LinkID=613736)
==============================================================================
##[command]. 'D:\a\_temp\455ec5ea-0200-42c0-8ddc-78ca01be30bb.ps1' 
LocalDB instance "mssqllocaldb" started.
 
##[section]Finishing: Start localdb
```

Happy testing!

&mdash; Dan