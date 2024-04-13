---
title: Windows Service Misbehaviour
date: 2024-04-13 23:00:00 +0200
categories: [PowerShell, N-Central]
tags: [powershell, n-central, windows service]
---
When you are used to working with N-Central, some obvious flaw in N-Central can easily be missed.

The default Windows Service service check in N-Central is able to detect if a service is starting, stopping or paused, but it is unable to report on it, as you can not set a threashold on it.
![N-Central Windows Service screenshot](/assets/images/N-Central-Windows-Service.png)

This issue has created an issue for a customer because his process was stuck on stopping and we didn't notice it.

Now, to fix this issue, we have the following simple Powershell Command to get all services that are currently not running or stopped:

```PowerShell
Get-Service | Where-Object {$_.Status -ne 'Stopped' -and $_.Status -ne 'Running'}
```
{: .nolineno }

When you run this simple one-liner, you get all services that are basically misbehaving and you would like to be reported on.
To get this into a simple script for N-Central, you get the following:

```PowerShell
$TwilightServiceNames = ""
$TwilightServices = Get-Service | Where-Object {$_.Status -ne 'Stopped' -and $_.Status -ne 'Running'}
if ($TwilightServices.Count -gt 0)
{
    foreach ($TwilightService in $TwilightServices)
    {
        $TwilightServiceNames += $TwilightService.DisplayName + "; "
    }
    $TwilightServiceNames = $TwilightServiceNames.Substring(0, $TwilightServiceNames.Length -2)
}
$TwilightServiceCount = $TwilightServices.Count
```

Download AMP for direct import into N-Central: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Windows%20Service%20Misbehaviour/Windows%20Services%20Misbehaviour.amp)

Download PS1 file: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Windows%20Service%20Misbehaviour/Windows%20Services%20Misbehaviour.ps1)