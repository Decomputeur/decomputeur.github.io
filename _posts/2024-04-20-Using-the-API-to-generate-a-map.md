---
title: Using the API to generate a map
date: 2024-04-20 22:00:00 +0200
categories: [PowerShell, N-Central]
tags: [powershell, n-central, api]
image:
  path: /assets/images/map.png
  alt: A map with pins.
---
Sometimes you have the need to plot all your customers onto a map and see how widespread in the area they are.
To do this, you can use Google Maps (for example) to create a manual map of all Customers by address, but how fun would it be if you could use PowerShell for this?

In this article i'm going to show you how you can generate the map, like the one above with your own customers and sites from data pulled from N-Central using the API and the [PS-NCentral](https://www.powershellgallery.com/packages/PS-NCentral/1.5) module written by Adriaan Sluis.

To prepare, we need to create 2 custom properties in N-Central. These custom properties will contain the latitude and longitude for the location of the customer. We do this to prevent an overload to the OpenStreetmap API each time we run this Powershell script.
![Custom Properties Latitude and Longitude](/assets/images/custom_properties_lat_lon.png)

Next, make sure to have the Details tab of your customers and sites filled in with the correct address for that customer and/or site. If this isn't filled, it will not get included into the map.

As usual, you need an API user with a generated API token with enough access to be able to write the latitude and longitude back to N-Central.

First we need to check if the PS-NCentral module is installed and if not install then import the module for use.
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

Next, we'll setup 2 variables to hold the NCentral server URL and JWT Token.
```powershell
$NCServer = 'https://yourncentralserver.domain.tld'
$JWTToken = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c'
```

now, let's connect to N-Central and get our customers data
```powershell
Write-Host -ForegroundColor Yellow "Connecting to NCentral"
$NCSession = New-NCentralConnection -ServerFQDN $NCServer -JWT $JWTToken
Write-Host -ForegroundColor Yellow "Getting Customers"
$Customers = Get-NCCustomerList -NcSession $NCSession
```

Now we have a PowerShell variable named $Customers that will hold all our customers data.
Let's initialize 2 variables for internal script usage.
```powershell
$CustomerArray = @()
$i = 1
```

The first variable named $CustomerArray will contain the array with all customers in a nice orderly fashion.
The second variable $i is used as a counter to display a progressbar during execution of the script.

We need to process the $Customers data into the $CustomerArray so that we have a nice base to generate the final array from which we need to display the markers from.
To do this, we loop through every customer in the $Customers array.
```powershell
foreach ($Customer in $Customers)
{
    Write-Progress -Activity "Processing Customer $i of $($Customers.Count)" -Status "Progress:" -PercentComplete ($i/$($Customers.Count)*100)
    $lat = 0
    $lon = 0
    $CustomerID = $Customer.customerid
    $CustomerName = [System.Net.WebUtility]::HtmlEncode($Customer.customername)
    $CustomerStreet1 = $Customer.street1
    $CustomerStreet2 = $Customer.street2
    $CustomerCity = $Customer.city
    $CustomerStateProvince = $Customer.stateprov
    $CustomerZipCode = $Customer.postalcode
    $CustomerCountry = $Customer.county
    $CustomerParentID = $Customer.parentid
    $CustomerStreet1Web = $CustomerStreet1.Replace(" ", "+")
    $CustomerCustomProperties = Get-NCCustomerPropertyList -CustomerIDs $CustomerID
    $CustomerLat = $CustomerCustomProperties.Latitude
    $CustomerLon = $CustomerCustomProperties.Longitude
    $CustomerActiveIssues = (Get-NCActiveIssuesList -CustomerID $CustomerID).count
    if (($CustomerStreet1Web -ne "") -and ($CustomerLat -eq ""))
    {
        $OpenStreetMapUri = "https://nominatim.openstreetmap.org/search?q=$CustomerStreet1Web+$CustomerCity+$CustomerCountry&format=xml"
        $OpenStreetMapResult = Invoke-WebRequest -uri $OpenStreetMapUri
        $OpenStreetMapResultXML = [xml]$OpenStreetMapResult
        $CustomerName
        $OpenStreetMapUri
        $Places = $OpenStreetMapResultXML.searchresults.place
        if ($OpenStreetMapResultXML.searchresults.place -is [System.Array])
        {
            write-host "Array"
            $lat = $OpenStreetMapResultXML.searchresults.place[0].lat
            $lon = $OpenStreetMapResultXML.searchresults.place[0].lon
            Set-NCCustomerProperty -CustomerIDs $CustomerID -PropertyLabel Latitude -PropertyValue $lat
            Set-NCCustomerProperty -CustomerIDs $CustomerID -PropertyLabel Longitude -PropertyValue $lon
        }
        else
        {
            write-host "NOT-Array"
            $lat = $OpenStreetMapResultXML.searchresults.place.lat
            $lon = $OpenStreetMapResultXML.searchresults.place.lon
            Set-NCCustomerProperty -CustomerIDs $CustomerID -PropertyLabel Latitude -PropertyValue $lat
            Set-NCCustomerProperty -CustomerIDs $CustomerID -PropertyLabel Longitude -PropertyValue $lon
        }
        $lat
        $lon
        write-host "--"
        $CustomerLat = $lat
        $CustomerLon = $lon
    }
    $item = New-Object PSObject
    $item | Add-Member -type NoteProperty -Name 'CustomerID' -Value $CustomerID
    $item | Add-Member -type NoteProperty -Name 'CustomerParentID' -Value $CustomerParentID
    $item | Add-Member -type NoteProperty -Name 'CustomerName' -Value $CustomerName
    $item | Add-Member -type NoteProperty -Name 'CustomerStreet1' -Value $CustomerStreet1
    $item | Add-Member -type NoteProperty -Name 'CustomerStreet2' -Value $CustomerStreet2
    $item | Add-Member -type NoteProperty -Name 'CustomerZipCode' -Value $CustomerZipCode
    $item | Add-Member -type NoteProperty -Name 'CustomerCity' -Value $CustomerCity
    $item | Add-Member -type NoteProperty -Name 'CustomerStateProvince' -Value $CustomerStateProvince
    $item | Add-Member -type NoteProperty -Name 'CustomerCountry' -Value $CustomerCountry
    $item | Add-Member -type NoteProperty -Name 'CustomerLat' -Value $CustomerLat
    $item | Add-Member -type NoteProperty -Name 'CustomerLon' -Value $CustomerLon
    $item | Add-Member -type NoteProperty -Name 'CustomerActiveIssues' -Value $CustomerActiveIssues
    $CustomerArray += $item
    $i++
}
Write-Progress -Activity "Processing Customer" -Completed
```

As you can see on line 3 above, we create a progressbar to show our progress while executing our script. The last line marks the completion and removal of this progress bar.

Line 7 is a special line. this encodes the name of the customer to be HTML safe. Using this, you can prevent that special characters break the HTML.

Then on line 15, i replace every space with a + sign to make it possible to use as search parameter.

On line 16, I get all customer properties set for this customer. This will include our new (and now still empty) Latitude and Longitude parameters.

On line 19 we retrieve all active issues for this customer to be displayed on the map as well.

Line 20 will do a comparison if the Street1 field is filled and latitude field isn't filled. Only if both of these conditions are met, then we will do a fetch of the street, city and country via the OpenStreetmap API on line 23.

On line 24 we convert the result to an XML object.

Line 28 will check if the searchresults.place is an array or not and if it is an array, we need to use the first item starting with id 0.

On lines 33 and 34 or 41 and 42 we will write the fetched Latitude and Longitude to the N-Central custom properties we created at the start.

Lines 50 to 62 we create a custom object containing all data we need later and on line 63 we add that custom object to our $CustomerArray.

With line 63, we add 1 to our loop array to progress our progressbar we started on line 3.

Now that we have an array named $CustomerArray which holds all data we need, we will process this to a large string to be used in the HTML of the page we create for the map.
```powershell
$CustomerPlots = ""
foreach ($SingleCustomer in $CustomerArray)
{
    if (($SingleCustomer.CustomerLat -ne 0) -and ($SingleCustomer.CustomerLat -notlike ""))
    {
        if ($SingleCustomer.CustomerParentID -eq 50)
        {
            $CustomerPlots += "{'lon': " + $SingleCustomer.CustomerLon + ", 'lat': " + $SingleCustomer.CustomerLat + ",'Customer': '" + $SingleCustomer.CustomerName + "', 'Location':'" + $SingleCustomer.CustomerName + "','ActiveIssues':'" + $SingleCustomer.CustomerActiveIssues + "'},"
        }
        else
        {
            for($i=0;$i-le $CustomerArray.length-1;$i++)
            {
                if ($CustomerArray[$i].CustomerID -eq $SingleCustomer.CustomerParentID)
                {
                    $CustomerPlots += "{'lon': " + $SingleCustomer.CustomerLon + ", 'lat': " + $SingleCustomer.CustomerLat + ",'Customer': '" + $CustomerArray[$i].CustomerName + "', 'Location':'" + $CustomerArray[$i].CustomerName + " - " + $SingleCustomer.CustomerName.Replace("'","\'") + "','ActiveIssues':'" + $SingleCustomer.CustomerActiveIssues + "'},"
                }
            }
        }
    }
}
```

We start once again with the initialization of an empty variable named $CustomerPlots. This variable will hold all data we need to plot the customer on the map.

We loop through each Customer in $CustomerArray, we check if the customer has a Latitude larger then 0 and isn't empty to only include customers for which the coordinates have been found using the OpenStreetMap API.

Next we'll do some trickery. If the customerParentID is 50 we'll add it to the list of $CustomerPlots.

If the CustomerParentID isn't 50, we need to loop through the entire array and get each site which is related to this CustomerID. Yes this isn't really efficient because it will loop many many times, even for customers which have already been added to the list, but it will group all customers and sites in the variable $CustomerPlots together and will make sure that each site will have the correct name included with each customer.

The last part is to generate the HTML data and save it to a file.
```powershell
$HTMLExport = @"
<!DOCTYPE html>
<html>
<head>
  <title>Detron N-Central Customer Map</title>
  <style>
	*{
		margin: 0;
		padding: 0;
	}
	#map{
		width: 100%;
		height: 100vh;
	}
	.leaflet-popup-content {
		margin: 5px;
	}
	.card {
		/*width: 450px;*/
		display: flex;
		flex-direction: column;
		flex-wrap: nowrap;
		align-items: left;
	}
	.card img{
		display: block;
		margin-right: 10px;
		border-radius: 5px 0 0 5px;
		-moz-border-radius: 5px 0 0 5px;
		-webkit-border-radius: 5px 0 0 5px;
	}
  </style>
  <link rel='stylesheet' href='https://unpkg.com/leaflet@1.7.1/dist/leaflet.css'>
</head>
<body>
  <div id="map"></div>
  <script src='https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/leaflet.js'></script>
  <script>
	//Default View
	let mapOptions = {
		center:[50.9, 6.19275],
		zoom:7
	}
	let map = new L.map('map', mapOptions);
	let layer = new L.TileLayer('http://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png');
	map.addLayer(layer);
    // Put your point-definitions here
    let markers = [
"@

$HTMLExport += $CustomerPlots.Substring(0,$CustomerPlots.Length-1)

$HTMLExport += @"
    ];
    let popupOption = {
      "closeButton":false
    }
    markers.forEach(element => {
        new L.Marker([element.lat,element.lon]).addTo(map)
      .on("mouseover",event =>{
        if (element.ActiveIssues > 0)
        {
          event.target.bindPopup('<div class="card"><img src="https://www.freeiconspng.com/uploads/red-not-ok-icon-22.png" width="20" height="20" alt="Error">   <h1>'+element.Customer+'</h1><h2>'+element.Location+'</h2><h3>Issues: '+element.ActiveIssues+'</h3></div>',popupOption).openPopup();
        }
        else
        {
          event.target.bindPopup('<div class="card"><img src="https://www.freeiconspng.com/uploads/green-ok-icon-2.png" width="20" height="20" alt="Normal">   <h1>'+element.Customer+'</h1><h2>'+element.Location+'</h2><h3>Issues: '+element.ActiveIssues+'</h3></div>',popupOption).openPopup();
        }
      })
      .on("mouseout", event => {
        event.target.closePopup();
      })
    });
    </script>
  </body>
  </html>
"@

$HTMLExport | Out-File "OpenStreetMap Customers.html"
```

When you put this all together, you will get the result of the file you can download below.

If you execute this script, you will get an interactive map where all customers are plotted onto an OpenStreetMap map.

When you hover on a marker, you will see what customer it is including it's site name if it is a site of a customer.

Download PS1 file: [GitHub](https://github.com/eagle00789/N-Central/blob/master/OpenStreetMap%20Customers/OpenStreetMap%20Customers.ps1)