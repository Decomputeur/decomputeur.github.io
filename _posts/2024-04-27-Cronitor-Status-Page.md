---
title: Cronitor Status Page
date: 2024-04-27 20:23:00 +0200
categories: [PowerShell, N-Central]
tags: [powershell, n-central, status page]
image:
  path: /assets/images/logos/cronitor.png
  alt: Cronitor Statuspage Logo.
---
Many cloud providers will have a public status page to see their availability and scheduled maintenance.
One of those public status page providers is from Cronitor.
An example of a company using Cronitor as status page is [Workspace365](https://status.workspace365.net/)

To see if a status page is using Cronitor, just look at the bottom right corner. It will show Powered by Cronitor.

To get data from this page, we need to parse the resulting HTML content, but this has been made easy for us as the required data resides inside a script section wich houses a json object that contains all info we need.

Let's start by setting our statuspage url.
```powershell
$StatusPageURL= "https://status.workspace365.net/"
```

Let's define our default output values.
```powershell
$MaintenanceScheduled = 0
$LinkToMaintenance = "."
$IncidentActive = 0
$LinkToIncident = "."
```

Grab the page
```powershell
$StatusHTML = Invoke-RestMethod -Method Get -Uri "$($StatusPageURL)"
```

Convert to required script block to JSON
```powershell
$StatusJSON = ConvertFrom-Json $StatusHTML.html.body.script.'#text'
```

If you want to grab and convert in a single line, you could replace the above two lines with this single line, but I prefer the double line to make it more readable
```powershell
$StatusJSON = ConvertFrom-Json (Invoke-RestMethod -Method Get -Uri "$($StatusPageURL)").html.body.script.'#text'
```

Now we extract the genera; status from the JSON object.
```powershell
$Status = $StatusJSON.props.pageProps.data.status.state
```

Grab all scheduled maintenance windows (if there are any scheduled) and get the first one if there are multiple, to only get the maintenance windows that comes up as first.
```powershell
$ScheduledMaintenances = $StatusJSON.props.pageProps.data.maintenance_windows
$MaintenanceID = ""
if ($ScheduledMaintenances.Count -ne 0)
{
    ForEach ($ScheduledMaintenance in $ScheduledMaintenances)
    {
        if ($MaintenanceID -eq "")
        {
            $MaintenanceID = $ScheduledMaintenance
        }
        else
        {
            if (($MaintenanceID.start).Date -ge ($ScheduledMaintenance.start).Date)
            {
                $MaintenanceID = $ScheduledMaintenance
            }
        }
    }
}
```

Now let's do the same for the incidents that are possibly active.
```powershell
$Incidents = $StatusJSON.props.pageProps.data.incidents
$IncidentID = ""
if ($Incidents.Count -ne 0)
{
    ForEach ($Incident in $Incidents)
    {
        if ($IncidentID -eq "")
        {
            $IncidentID = $Incident
        }
        else
        {
            if (($IncidentID.started_at).Date -ge ($Incident.started_at).Date)
            {
                $IncidentID = $Incident
            }
        }
    }
}
```

Finally we grab the page name and make all data presentable if not already done so.
```powershell
$PageName = $StatusJSON.props.pageProps.data.name
if ($MaintenanceID)
{
    $LinkToMaintenance = $StatusPageURL
    $MaintenanceScheduled = 1
}
if ($IncidentID)
{
    $LinkToIncident = $StatusPageURL
    $IncidentActive = 1
}
```

Download AMP for direct import into N-Central: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Cronitor%20Status%20Page/Cronitor%20Status%20Page.amp)

Download PS1 file: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Cronitor%20Status%20Page/Cronitor%20Status%20Page.ps1)