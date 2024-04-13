---
title: Afas Status Page
date: 2024-04-14 01:00:00 +0200
categories: [PowerShell, N-Central]
tags: [powershell, n-central, status page]
---
When your company is a user of the software Afas, you want to get the status of it in your N-Central to more easily see if it is reporting any issues.
Afas Software publishes it's status on the page [https://afasstatus.nl](https://afasstatus.nl), however if you look at that page, you can't see any way to subscribe by mail (except mention in an article on the help pages) or rss feed.

When you start to dig deeper into the inner workings of this page, you can find a reference to a subpage [/json](https://afasstatus.nl/json)

If you visit that page you immediately see that this page returns a json object with it's current state, like for example

```json
{"typeId":"1","typeName":"goed","typeMsg":"Er is geen storing bekend.","eventId":null}
```

As you can see in this example, all we need is the typeId and the eventId.
Based on my own observations, if typeId is 1, all systems are green, when it returns a 2 or 3, a warning or maintenance is scheduled and when it returns 4, it's a major issue.

When you want to get the correct info from this json object, the code you need for this is as following:

```powershell
$BaseURI = "https://afasstatus.nl"
$AfasData = (Invoke-WebRequest -Uri "$BaseURI/json").Content
$AfasJSON = ConvertFrom-Json -InputObject $AfasData
$Status = $AfasJSON.typeId
if ($Status -eq 1)
{
    $URL = "$BaseURI"
}
else {
    $URL = "$BaseURI/details/event:$($AfasJSON.eventId)"
}
```

As you can see in the script above, i even know (based on my own observations of the workings when an outage, maintenance or warning is active) the details can be found on the URL that is created in the else section of the if statement.

Download AMP for direct import into N-Central: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Afas%20Status%20Page/Afas%20Status.amp)

Download PS1 file: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Afas%20Status%20Page/Afas%20Status.ps1)