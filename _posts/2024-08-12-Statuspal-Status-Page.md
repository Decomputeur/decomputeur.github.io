---
title: Statuspal Status Page
date: 2024-08-12 13:25:00 +0200
categories: [PowerShell, N-Central]
tags: [powershell, n-central, status page]
image:
  path: /assets/images/logos/Statuspal.png
  alt: Statuspal Statuspage Logo.
---
Many cloud providers will have a public status page to see their availability and scheduled maintenance.
One of those public status page providers is from Statuspal.
An example of a company using Cronitor as status page is [Interconnect](https://status.interconnect-noc.nl/)

To see if a status page is using Statuspal, just click on the subscribe button in the top right corner and look at the text below the Subscribe button.
That should mention Statuspal.
If you don't have a subscribe button, view the page source. The first link element you see, should point to statuspal.io

Now that you have made sure that you are viewing a page using statuspal.io, you would like to monitor this page.
However, currently statuspal doesn't provide all the necessary information you need to monitor the status of the service you would like.
To get the required subdomain id, please send then an e-mail message at [statuspal](mailto:contact@statuspal.io) in which you request the subdomain id for the specific page you would like to monitor (this will change in the future)

As soon as you have the subdomain id, we can start monitoring the correct site.
Statuspal provides a public api for which we need the subdomain id, we requesed above, and browse to the following page:
```powershell
https://statuspal.io/api/v2/status_pages/$SubDomain/summary
```
This page provides us with a nice json formatted array of all the data we need.

Let's throw this into powershell.

First we initialize a few variables, to make sure they are empty:
```powershell
$Name = ""
$IncidentStatus = 0
$IncidentURL = ""
$MaintenanceStatus = 0
$MaintenanceURL = ""
```

Next, let's define the variable that will hold our subdomain and fill it with the required subdomain of the page we want to monitor:
```powershell
$Subdomain = "ic-noc"
```

Now we get the page source and grab the info we need from the resulting json array:
```powershell
$BaseURL = "https://statuspal.io/api/v2/status_pages/$SubDomain/summary"
try {
    $Status = Invoke-WebRequest -Uri $BaseURL
    $StatusObject = ConvertFrom-Json $Status
    $Name = $StatusObject.status_page.name
    if ($StatusObject.incidents.Count -gt 0) {
        $Incidents = $StatusObject.incidents
        $Incidents = $Incidents | Sort-Object -Property starts_at
        switch ($Incidents[0].type) {
            "major" {
                $IncidentStatus = 2
                $IncidentURL = $Incidents[0].url
            }
            "minor" {
                $IncidentStatus = 1
                $IncidentURL = $Incidents[0].url
            }
            "scheduled" {
                $MaintenanceStatus = 1
                $MaintenanceURL = $Incidents[0].url
            }
            Default {
                $IncidentStatus = 0
                $MaintenanceStatus = 0
            }
        }
    }
}
catch {
    $IncidentStatus = 2
    $IncidentURL = $_.Exception.Message
}
```

And that's it.

Download AMP for direct import into N-Central: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Statuspal%20Status%20Page/Statuspal%20Status%20Page.amp)

Download PS1 file: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Statuspal%20Status%20Page/Statuspal%20Status%20Page.ps1)