---
title: Remove old Powershell Modules
date: 2024-10-09 07:50
categories: [PowerShell]
tags: [powershell]
---
When are are using Powershell alot, you sometimes will come to a point where you have several versions of the same module installed.
These multiple versions can sometimes hold you back from using the correct version of the module.
These duplicates can be easily removed using the following very small powershell script.
```powershell
$Latest = Get-InstalledModule 
foreach ($module in $Latest) { 
  Write-Verbose -Message "Uninstalling old versions of $($module.Name) [latest is $( $module.Version)]" -Verbose
  Get-InstalledModule -Name $module.Name -AllVersions | Where-Object {$_.Version -ne $module.Version} | Uninstall-Module -Verbose -force
}
```

Happy cleanup of your old modules