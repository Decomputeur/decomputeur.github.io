---
title: Azure Application Certificates and Client Secrets
date: 2024-04-14 23:00:00 +0200
categories: [PowerShell, Azure]
tags: [powershell, azure]
image:
  path: /assets/images/Microsoft_Azure-Logo.svg
  alt: Microsoft Azure Logo.
---
One of the most important things to monitor in Microsoft Azure (for most customers) are the Application Certificates. This is because many connected Azure Application require such a certificate or client secret to communicate from the app to Azure (Entra) to authenticate, send mail, send teams message, read mail and do many other things via an Azure application.

To get if a certificate or client secret will expire, Microsoft has provided us with the powershell module named Microsoft.Graph.Applications.
This module has a dependency with Microsoft.Graph.Authentication, so first we need to make sure that both modules are installed. Microsoft, however made this part really easy for us.

Just run the following PowerShell command in a PowerShell console that has been starter as Administrator.
```powershell
Install-Module Microsoft.Graph.Applications
```

Now Powershell will install all the dependant modules for us. That means it will install both Microsoft.Graph.Applications and Microsoft.Graph.Authentication for us in a single command.

To connect to Azure using the authentication module we need to have the app setup as explained in [this post](posts/Azure-Monitoring-Requirements)

Now that we have all data we need, we can start by creating a few variables and running the command to authenticate using the app we created in the article above.
```powershell
$TenantID = "<TenantID>"
$ClientID = "<ClientID>"
$CertificateThumbprint = "<CertificateThumbprint>"
Connect-MgGraph -ClientId $ClientID -TenantId $TenantID -CertificateThumbprint $CertificateThumbprint -Environment Global
```
Just make sure to fill the variables above with the data from the app we created.

Now we will get the Certificate or Client Secret from the app we wish to monitor. Copy the Object ID from the app you wish to get the certificate or client secret from and past it as the value for the variable below.
```powershell
$ObjectID = "<ObjectID>"
$App = Get-MgApplicationById -Ids $ObjectID
```

Now the variable $App contains the app and information that we need buried in it's properties.
To get the certificate or client secret, we need to dig a little deeper in the $App variable.
```powershell
$AppProperties = $App.AdditionalProperties
$ClientKey = $AppProperties.keyCredentials
if (-not $ClientKey)
{
    $ClientKey = $AppProperties.passwordCredentials
}
```

The KeyCredentials is the certificate and the passwordCredentials is the Client Secret.
Because I have not found an app yet that has both the certificate and client secret set simultaneously, i first get the certificate and if there is no certificate set, i will retrieve the client secret.
Both of these have the same values we need, so to get expiration date and remaining days of validity, we can do the following small piece of powershell.
```powershell
$Now = Get-Date
$ClientKeyStartDate = [DateTime]$ClientKey.startDateTime
$ClientKeyEndDate = [DateTime]$ClientKey.endDateTime
$DaysRemaining = ($ClientKeyEndDate.Subtract($Now)).Days
```
If, for whatever reason, someone forgot to cleanup their old expired certificate or client secret, we can also first get the first certificate or client secret that hasn't been expired yet and then return the results.
This will result in the above script to be changed as following:
```powershell
$Now = Get-Date
ForEach ($Key in $ClientKey)
{
	$Date = [DateTime]$Key.endDateTime
	if ($Date.Subtract($Now).Days -ge 0)
	{
		$Index = $foreach.Current
		break
	}
}
if (-not $Index)
{
	$Index = $ClientKey[$ClientKey.Length-1]
}
$ClientKey = $Index

$ClientKeyStartDate = [DateTime]$ClientKey.startDateTime
$ClientKeyEndDate = [DateTime]$ClientKey.endDateTime
$DaysRemaining = ($ClientKeyEndDate.Subtract($Now)).Days
```

If we also want to retrieve the name of the application, we need to add the following line
```powershell
$AppDisplayName = $AppProperties.displayName.Trim()
```

Now when we all put this together in a single script, you would get the following as a result.
```powershell
$TenantID = "<TenantID>"
$ClientID = "<ClientID>"
$CertificateThumbprint = "<CertificateThumbprint>"
$ObjectID = "<ObjectID>"

$AppDisplayName = ""
$ClientSecretStartDate = ""
$ClientSecretEndDate = ""
$DaysRemaining = 0

$Now = Get-Date

if (($TenantID -match "<") -or ($ClientID -match "<") -or ($CertificateThumbprint -match "<") -or ($ObjectID -match "<"))
{
    $AppDisplayName = "Failure, not all parameters are filled in correctly"
    Exit
}

If (Get-Module -ListAvailable -Name Microsoft.Graph.Applications)
{
    import-module Microsoft.Graph.Applications
}
else
{
    $AppDisplayName = "Please install the Microsoft.Graph.Applications Powershell module on the executing server"
    Exit
}

[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Connect-MgGraph -ClientId $ClientID -TenantId $TenantID -CertificateThumbprint $CertificateThumbprint -Environment Global | Out-Null

$App = Get-MgApplicationById -Ids $ObjectID
$AppProperties = $App.AdditionalProperties
$ClientKey = $AppProperties.keyCredentials
if (-not $ClientKey)
{
    $ClientKey = $AppProperties.passwordCredentials
}

Disconnect-MgGraph

ForEach ($Key in $ClientKey)
{
	$Date = [DateTime]$Key.endDateTime
	if ($Date.Subtract($Now).Days -ge 0)
	{
		$Index = $foreach.Current
		break
	}
}
if (-not $Index)
{
	$Index = $ClientKey[$ClientKey.Length-1]
}

$ClientKey = $Index

$AppDisplayName = $AppProperties.displayName.Trim()
$ClientKeyStartDate = [DateTime]$ClientKey.startDateTime
$ClientKeyEndDate = [DateTime]$ClientKey.endDateTime
$DaysRemaining = ($ClientKeyEndDate.Subtract($Now)).Days
```

In the script above, I even added some error handling when the required powershell module isn't installed, or if some parameters aren't filled properly yet.

Download AMP for direct import into N-Central: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Azure%20Monitoring/Azure%20Application%20Certificate/Azure%20Application%20Certificate%20v2.amp)

Download PS1 file: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Azure%20Monitoring/Azure%20Application%20Certificate/Azure%20Application%20Certificate%20v2.ps1)