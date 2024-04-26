---
title: Get Asset Discoveryjobs
date: 2024-04-26 17:00:00 +0200
categories: [PowerShell, N-Central]
tags: [powershell, n-central, api, soap]
---
Asset Discoveryjobs are an important part of N-Central, to keep your asset information up to date and possibly find any new assets you might want to add to you monitoring.
Keeping up with all changes in the subnets you need to scan of all your managed networks might be a cumbersome task to do manually.
N-Central has a great tool for this called Discovery Jobs.
These Discovery Jobs will keep any asset you already manage up to date and it will also discovery any new asset you might want to monitor, but keeping all those Discovery Jobs managed might be to much. Therefore we can export these jobs using the NCentral Soap API by using the [PS-NCentral](https://www.powershellgallery.com/packages/PS-NCentral/1.5) module written by Adriaan Sluis and the [ImportExcel](https://www.powershellgallery.com/packages/ImportExcel/7.8.6) module written by Doug Finke.

We start of by checking if the PS-NCentral module is installed and if installed, then load it, else we install it and then immediately load it.
```powershell
$CheckImportPSNCentral = Get-Module -ListAvailable -Name PS-NCentral
if (!$CheckImportPSNCentral) {
    Install-Module PS-NCentral -Scope CurrentUser
    Import-Module PS-NCentral
}
else {
    Import-Module PS-NCentral
}
```

Now lets do the exact same for the ImportExcel module
```powershell
$CheckImportExcelModule = Get-Module -ListAvailable -Name ImportExcel
if (!$CheckImportExcelModule) {
    Install-Module ImportExcel -Scope CurrentUser
    Import-Module ImportExcel
}
else {
    Import-Module ImportExcel
}
```

Next, we'll setup 2 variables to hold the NCentral server URL and JWT Token.
```powershell
$NCServer = 'https://yourncentralserver.domain.tld'
$JWTToken = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c' # fake JWT token
```

Let's connect to N-Central
```powershell
Write-Host -ForegroundColor Yellow "Connecting to NCentral"
$NCSession = New-NCentralConnection -ServerFQDN $NCServer -JWT $JWTToken
```

In this example we get all asset discovery jobs for our customer with ID 50 (system level) and we will filter all jobs we retrieve to only include jobs with a type 'Asset Discovery Task'
```powershell
$Jobs = Get-NCJobStatusList -CustomerID 50 | where-object -Property jobtype -Like "Asset Discovery Task"
```

Next we need to do some magic foreach loop to process all the data we got in our `$Jobs` variable.
```powershell
$AssetScanJobs = @()
ForEach ($Job in $Jobs) {
    Write-Progress -Activity "Processing Job $($Jobs.IndexOf($Job)) of $($Jobs.Count)" -Status "Progress:" -PercentComplete ($Jobs.IndexOf($Job)/$($Jobs.Count)*100)
    $AssetScanJob = New-Object PSObject
    $AssetScanJob | Add-Member -type NoteProperty -Name 'Customer' -Value $Job.customername
    $AssetScanJob | Add-Member -type NoteProperty -Name 'Site' -Value $Job.sitename
    $AssetScanJob | Add-Member -type NoteProperty -Name 'Job Name' -Value $Job.jobname
    $AssetScanJob | Add-Member -type NoteProperty -Name 'Scheduled Time' -Value $Job.scheduledtime.DateTime
    $AssetScanJob | Add-Member -type NoteProperty -Name 'Job Target' -Value $Job.jobtarget
    $AssetScanJob | Add-Member -type NoteProperty -Name 'Last Completion Time' -Value $Job.lastcompletiontime.DateTime
    $AssetScanJob | Add-Member -type NoteProperty -Name 'Status' -Value $Job.status
    $AssetScanJobs += $AssetScanJob
}
Write-Progress -Activity "Processing Job" -Completed
```

In this case we need to use a custom object to hold all of our data we want exported.
The `Write-Progress` line is to see how many jobs we have retrieved and processed. this might be usefull if you have many jobs on your SO level, so this might take a while to process.

If you want you asset scan sorted before you export it, you can do so by including the next line.
This will sort your asses scans on Customer, then on Site and then on Scheduled Time to get a complete ordered list of all your scans
```powershell
$AssetScanJobs = $AssetScanJobs | Sort-Object -Property Customer, Site, 'Scheduled Time'
```

The last thing we need to do is export it to excel using the Export-Excel function of the ImportExcel module we installed and imported.
```powershell
Export-Excel -Path ".\Exports\DiscoveryJobs.xlsx" -WorksheetName (Get-Date -Format "dd-MM-yyyy HH.mm") -InputObject $AssetScanJobs -AutoSize -AutoFilter -FreezeTopRow -NoNumberConversion * -TableStyle Medium10
```

We can reuse the same excel file, since we create a new worksheet with the current date and time (in EU formatting).
We make the columns Autosizing, we enable the AutoFilter, we Freeze the top row, we disable number conversion to prevent converting some values from a number to a date and we add the Medium10 styole to our table.