---
title: Azure Automation Job Status
date: 2025-02-04 14:00:00 +0200
categories: [PowerShell, Azure]
tags: [powershell, azure]
image:
  path: /assets/images/logos/Microsoft_Azure-Logo.svg
  alt: Microsoft Azure Logo.
---
Sometimes you are using an Azure Automation to get things done using the Microsoft methods because you don't have another choice.
The problem when using N-Central is that you can't monitor these Automations from within N-Central. For this purpose you need to logon to Azure and check the status of the runbook to get the result. It is much nicer to see in N-Central if a runbook has failed for some reason.

To be able to monitor such a job, we need the following data:
- Resource Group Name. This is the name of the resourcegroup in Azure, on which your job is linked to.
- Automation Account Name. This is the name of the Automation account the resource group is linked to.
- Runbook Name. This is the name of the runbook linked to the Automation Account specified above.

We will also need to have the app setup as explained in [this post]({{ site.baseurl }}{% link _posts/2024-04-14-Azure-Monitoring-Requirements.md %})

Next we need to install a few specific versions of powershell modules, because of some bugs in later releases of these modules.
To install these specific versions, execute the following lines of powershell
```powershell
Uninstall-Module -name Az.Automation -AllVersions
Install-Module -Name Az.Automation -RequiredVersion 1.9.1
Install-Module -Name Az.Accounts -RequiredVersion 2.12.1
```

Now that we are setup, we can start by setting up some variables which hold our needed data:
```powershell
$TenantID = "<TenantID>"
$ClientID = "<ClientID>"
$CertificateThumbprint = "<CertificateThumbprint>"
$ResourceGroupName = "<ResourcegroupName>"
$AutomationAccountName = "<AutomationAccountName>"
$RunbookName = "<RunbookName>"
```

We also need to setup some return variables with some default values. For easy of use, I have the type mentioned next to it
```powershell
$Status = ""        #String
$LastRun = ""       #Date
$RunTime = 0       #Integer (Minutes)
$ErrorCount = 0    #Integer
```

Let's check if we have all required data setup properly
```powershell
if (($TenantID -match "<") -or ($ClientID -match "<") -or ($CertificateThumbprint -match "<") -or ($ResourceGroupName -match "<") -or ($AutomationAccountName -match "<") -or ($RunbookName -match "<"))
{
    $Status = "Failure, not all parameters are filled in correctly"
    Exit
}
```

Do a quick check of the required modules and import them
```powershell
If (Get-Module -ListAvailable -Name AZ.Accounts)
{
    Import-Module AZ.Accounts
}
else
{
    $Status = "Please install the AZ.Accounts Powershell module on the executing server"
    Exit
}

If (Get-Module -ListAvailable -Name AZ.Automation)
{
    Import-Module AZ.Automation
}
else
{
    $Status = "Please install the AZ.Automation Powershell module on the executing server"
    Exit
}
```

Let's connect to Azure and force using a TLS1.2 connection
```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Connect-AzAccount -ServicePrincipal -ApplicationId $ClientID -Tenant $TenantID -CertificateThumbprint $CertificateThumbprint
```

Now, let's get the job data and make sure to only use the most recent run of the job
```powershell
$Job = (Get-AzAutomationJob -ResourceGroupName $ResourceGroupName -AutomationAccountName $AutomationAccountName -RunbookName $RunbookName)[0]
```

We also want to get the Error stream of the most recent job, so that we can add the errorcount in the output if any error has been output
```powershell
$JobErrors = (Get-AzAutomationJobOutput -Id $Job.JobId -ResourceGroupName $ResourceGroupName -AutomationAccountName $AutomationAccountName -stream "Error")
```

We no longer need a connection with Azure, so let's disconnect it.
```powershell
Disconnect-AzAccount
```

Our last step is to convert any data so that N-Central can interpret it correctly.
```powershell
$Status = $Job.Status
$LastRun = $Job.StartTime.Date
if ($Status -eq "Running") {
    $RunTime = 0
    $Status += ", not Completed yet!"
}
else {
    $RunTime = [math]::Round(($Job.EndTime - $Job.StartTime).TotalMinutes)
}
$ErrorCount = $JobErrors.Count
```
In the snippet above, we check if the job has a Running status and if it does, we make sure to include the word Completed. That way we can use the Status result in N-Central to check if the word Completed is contained in the output and report a Normal state when the job is still not finished. This prevents a false positive if a job hasn't finished yet.

To eliviate some stress on your server you are using to monitor these jobs, i have set it on a schedule to only check every hour and not every 5 minutes, but this ofcourse all deepends on your specific situation.

# Downloads
Download AMP for direct import into N-Central: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Azure%20Monitoring/Azure%20Automation%20Job%20Status/Azure%20Automation%20Job%20Status.amp)

Download PS1 file: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Azure%20Monitoring/Azure%20Automation%20Job%20Status/AzureAutomationJobStatus.ps1)