---
title: Atlassian Status Page
date: 2024-04-15 09:30:00 +0200
categories: [PowerShell, N-Central]
tags: [powershell, n-central, status page]
image:
  path: /assets/images/logos/logo-gradient-blue-statuspage.png
  alt: Microsoft Azure Logo.
---
Many cloud providers will have a public status page to see their availability and scheduled maintenance.
One of those public status page providers is from Atlassian.
Examples of companies using Atlassian as status page are:
- [Atlassian](https://metastatuspage.com)
- [1Password](https://1password.statuspage.io/)
- [Carbonite](https://status.carbonite.com/)
- [Teamviewer](https://status.teamviewer.com/)

Fortunatly, the Atlassian Status page has an easy api available. This api is available [here](https://metastatuspage.com/api).

the quick and dirty way to connect to this api is to append the URL of the statuspage with /api/v2/summary.json to get the summary information in a json formatted text.

If you view the json text, you will get the following example result:
```json
{
  "page": {
    "id": "y2j98763l56x",
    "name": "Atlassian Statuspage",
    "url": "http://metastatuspage.com",
    "updated_at": "2024-03-27T17:57:01-07:00"
  },
  "status": {
    "description": "Partial System Outage",
    "indicator": "major"
  }
  "components": [
    {
      "created_at": "2014-05-03T01:22:07.274Z",
      "description": null,
      "id": "b13yz5g2cw10",
      "name": "API",
      "page_id": "y2j98763l56x",
      "position": 1,
      "status": "partial_outage",
      "updated_at": "2014-05-14T20:34:43.340Z"
    },
    {
      "created_at": "2014-05-03T01:22:07.286Z",
      "description": null,
      "id": "9397cnvk62zn",
      "name": "Management Portal",
      "page_id": "y2j98763l56x",
      "position": 2,
      "status": "major_outage",
      "updated_at": "2014-05-14T20:34:44.470Z"
    }
  ],
  "incidents": [
    {
      "created_at": "2014-05-14T14:22:39.441-06:00",
      "id": "cp306tmzcl0y",
      "impact": "critical",
      "incident_updates": [
        {
          "body": "Our master database has ham sandwiches flying out of the rack, and we're working our hardest to stop the bleeding. The whole site is down while we restore functionality, and we'll provide another update within 30 minutes.",
          "created_at": "2014-05-14T14:22:40.301-06:00",
          "display_at": "2014-05-14T14:22:40.301-06:00",
          "id": "jdy3tw5mt5r5",
          "incident_id": "cp306tmzcl0y",
          "status": "identified",
          "updated_at": "2014-05-14T14:22:40.301-06:00"
        }
      ],
      "monitoring_at": null,
      "name": "Unplanned Database Outage",
      "page_id": "y2j98763l56x",
      "resolved_at": null,
      "shortlink": "http://stspg.co:5000/Q0E",
      "status": "identified",
      "updated_at": "2014-05-14T14:35:21.711-06:00"
    }
  ],
  "scheduled_maintenances": [
    {
      "created_at": "2014-05-14T14:24:40.430-06:00",
      "id": "w1zdr745wmfy",
      "impact": "none",
      "incident_updates": [
        {
          "body": "Our data center has informed us that they will be performing routine network maintenance. No interruption in service is expected. Any issues during this maintenance should be directed to our support center",
          "created_at": "2014-05-14T14:24:41.913-06:00",
          "display_at": "2014-05-14T14:24:41.913-06:00",
          "id": "qq0vx910b3qj",
          "incident_id": "w1zdr745wmfy",
          "status": "scheduled",
          "updated_at": "2014-05-14T14:24:41.913-06:00"
        }
      ],
      "monitoring_at": null,
      "name": "Network Maintenance (No Interruption Expected)",
      "page_id": "y2j98763l56x",
      "resolved_at": null,
      "scheduled_for": "2014-05-17T22:00:00.000-06:00",
      "scheduled_until": "2014-05-17T23:30:00.000-06:00",
      "shortlink": "http://stspg.co:5000/Q0F",
      "status": "scheduled",
      "updated_at": "2014-05-14T14:24:41.918-06:00"
    },
    {
      "created_at": "2014-05-14T14:27:17.303-06:00",
      "id": "k7mf5z1gz05c",
      "impact": "minor",
      "incident_updates": [
        {
          "body": "Scheduled maintenance is currently in progress. We will provide updates as necessary.",
          "created_at": "2014-05-14T14:34:20.036-06:00",
          "display_at": "2014-05-14T14:34:20.036-06:00",
          "id": "drs62w8df6fs",
          "incident_id": "k7mf5z1gz05c",
          "status": "in_progress",
          "updated_at": "2014-05-14T14:34:20.036-06:00"
        },
        {
          "body": "We will be performing rolling upgrades to our web tier with a new kernel version so that Heartbleed will stop making us lose sleep at night. Increased load and latency is expected, but the app should still function appropriately. We will provide updates every 30 minutes with progress of the reboots.",
          "created_at": "2014-05-14T14:27:18.845-06:00",
          "display_at": "2014-05-14T14:27:18.845-06:00",
          "id": "z40y7398jqxc",
          "incident_id": "k7mf5z1gz05c",
          "status": "scheduled",
          "updated_at": "2014-05-14T14:27:18.845-06:00"
        }
      ],
      "monitoring_at": null,
      "name": "Web Tier Recycle",
      "page_id": "y2j98763l56x",
      "resolved_at": null,
      "scheduled_for": "2014-05-14T14:30:00.000-06:00",
      "scheduled_until": "2014-05-14T16:30:00.000-06:00",
      "shortlink": "http://stspg.co:5000/Q0G",
      "status": "in_progress",
      "updated_at": "2014-05-14T14:35:12.258-06:00"
    }
  ]
}
```
The most important parts from this is the incidents and scheduled_maintenance.
When a scheduled_maintenance is planned or an incident is active, the now empty array's for those fields will be filled with the correct corresponding information.

The information we need is scheduled_for for when it is a scheduled maintenance or started_at when it is an incident.

to put it all into a script, you get the following.
```powershell
$MaintenanceScheduled = 0
$LinkToMaintenance = "."
$IncidentActive = 0
$LinkToIncident = "."

$StatusJSON = Invoke-RestMethod -Method Get -Uri "$($StatusPageURL)/api/v2/summary.json"
$ScheduledMaintenances = $StatusJSON.scheduled_maintenances
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
            if (($MaintenanceID.scheduled_for).Date -ge ($ScheduledMaintenance.scheduled_for).Date)
            {
                $MaintenanceID = $ScheduledMaintenance
            }
        }
    }
}

$Incidents = $StatusJSON.incidents
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

$PageName = $StatusJSON.page.name
if ($MaintenanceID)
{
    $LinkToMaintenance = $MaintenanceID.shortlink
    $MaintenanceScheduled = 1
}
if ($IncidentID)
{
    $LinkToIncident = $IncidentID.shortlink
    $IncidentActive = 1
}
```
Download AMP for direct import into N-Central: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Atlassian%20Status%20Page/Atlassian%20Status%20Page.amp)

Download PS1 file: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Atlassian%20Status%20Page/AtlassianStatusPage.ps1)