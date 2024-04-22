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

The first thing we start to do is create 2 variables for checking later output and writeback:

```powershell
# To enable debugging messages, remove the comment from the line below
# $DebugPreference = "continue"

# To enable writing of Latitude and Longitude back to N-Central, set value below to $true, otherwise to disable, set to $false
$WriteBack = $true
```

Next we need to check if the PS-NCentral module is installed and if not install then import the module for use.
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
$JWTToken = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c' # fake JWT token
```

now, let's connect to N-Central and get our customers data
```powershell
Write-Host -ForegroundColor Yellow "Connecting to NCentral"
$NCSession = New-NCentralConnection -ServerFQDN $NCServer -JWT $JWTToken
Write-Host -ForegroundColor Yellow "Getting Customers"
$Customers = Get-NCCustomerList -NcSession $NCSession
```

Now we have a PowerShell variable named $Customers that will hold all our customers data.
Let's initialize a variables for internal script usage.
```powershell
$CustomerArray = @()
```

The variable named $CustomerArray will contain the array with all customers in a nice orderly fashion.

We need to process the $Customers data into the $CustomerArray so that we have a nice base to generate the final array from which we need to display the markers from.
To do this, we loop through every customer in the $Customers array.
```powershell
foreach ($Customer in $Customers)
{
    Write-Progress -Activity "Processing Customer $($Customers.IndexOf($Customer)) of $($Customers.Count)" -Status "Progress:" -PercentComplete ($Customers.IndexOf($Customer)/$($Customers.Count)*100)
    $lat = 0
    $lon = 0
    $CustomerName = [System.Net.WebUtility]::HtmlEncode($Customer.customername)
    $CustomerStreet1Web = [System.Net.WebUtility]::HtmlEncode($Customer.street1)
    $CustomerCustomProperties = Get-NCCustomerPropertyList -CustomerIDs $Customer.customerid
    $CustomerActiveIssues = (Get-NCActiveIssuesList -CustomerID $Customer.customerid).count
    if (($CustomerStreet1Web -ne "") -and ($CustomerCustomProperties.Latitude -eq ""))
    {
        $OpenStreetMapUri = ("https://nominatim.openstreetmap.org/search?q={0}+{1}+{2}&format=xml" -f $CustomerStreet1Web,$Customer.city,$Customer.county)
        $OpenStreetMapResult = Invoke-WebRequest -uri $OpenStreetMapUri
        $OpenStreetMapResultXML = [xml]$OpenStreetMapResult
        write-debug $CustomerName
        write-debug $OpenStreetMapUri
        if ($OpenStreetMapResultXML.searchresults.place -is [System.Array])
        {
            write-debug "Array"
            $lat = $OpenStreetMapResultXML.searchresults.place[0].lat
            $lon = $OpenStreetMapResultXML.searchresults.place[0].lon
        }
        else
        {
            write-debug "NOT-Array"
            $lat = $OpenStreetMapResultXML.searchresults.place.lat
            $lon = $OpenStreetMapResultXML.searchresults.place.lon
        }
        if ($WriteBack)
        {
            Set-NCCustomerProperty -CustomerIDs $Customer.customerid -PropertyLabel Latitude -PropertyValue $lat
            Set-NCCustomerProperty -CustomerIDs $Customer.customerid -PropertyLabel Longitude -PropertyValue $lon
        }
        write-debug $lat
        write-debug $lon
        write-debug "--"
        $CustomerCustomProperties.Latitude = $lat
        $CustomerCustomProperties.Longitude = $lon
    }
    $item = [pscustomobject][ordered]@{
        CustomerID = $Customer.customerid
        CustomerParentID = $Customer.parentid
        CustomerName = $CustomerName
        CustomerStreet1 = $CustomerStreet1Web
        CustomerStreet2 = $Customer.street2
        CustomerZipCode = $Customer.postalcode
        CustomerCity = $Customer.city
        CustomerStateProvince = $Customer.stateprov
        CustomerCountry = $Customer.county
        CustomerLat = $CustomerCustomProperties.Latitude
        CustomerLon = $CustomerCustomProperties.Longitude
        CustomerActiveIssues = $CustomerActiveIssues
    }
    $CustomerArray += $item
}
Write-Progress -Activity "Processing Customer" -Completed
```

As you can see on line 3 above, we create a progressbar to show our progress while executing our script. The last line marks the completion and removal of this progress bar.

Line 6 and 7 are special lines. This encodes the name of the customer and the Street1 to be HTML safe. Using this, you can prevent that special characters break the HTML.

On line 8, I get all customer properties set for this customer. This will include our new (and now still empty) Latitude and Longitude parameters.

On line 9 we retrieve all active issues for this customer to be displayed on the map as well.

Line 10 will do a comparison if the Street1 field is filled and latitude field isn't filled. Only if both of these conditions are met, then we will do a fetch of the street, city and country via the OpenStreetmap API on line 23.

On line 14 we convert the result from OpenStreetMap to an XML object.

Line 17 will check if the searchresults.place is an array or not and if it is an array, we need to use the first item starting with id 0.

On lines 31 and 32 we will write the fetched Latitude and Longitude to the N-Central custom properties we created at the start if we enabled the variable to do so.

Lines 40 to 54 we create a custom object containing all data we need later and on line 63 we add that custom object to our $CustomerArray.

The several `Write-Debug` commands used throughout the code above will only dispay it's output if you removed the comment for the `$DebugPreference = "continue"` line at the start.

Now that we have an array named $CustomerArray which holds all data we need, we will process this to a large string to be used in the HTML of the page we create for the map.
```powershell
$PlotList = @()
foreach ($SingleCustomer in $CustomerArray)
{
    if ($SingleCustomer.CustomerLat)
    {
        $PlotProps=@{}
        $PlotProps.lon = $SingleCustomer.CustomerLon
        $PlotProps.lat = $SingleCustomer.CustomerLat
        $PlotProps.ActiveIssues = $SingleCustomer.CustomerActiveIssues
        if ($SingleCustomer.CustomerParentID -eq 50)
        {
            $PlotProps.Customer = $SingleCustomer.CustomerName
            $PlotProps.Location = ""
        }
        else
        {
            $PlotProps.Customer = ($CustomerArray | Where-Object -Property CustomerID -EQ $SingleCustomer.CustomerParentID).CustomerName
            $PlotProps.Location = $SingleCustomer.CustomerName
        }
        $PlotList += [PSCustomObject]$PlotProps
    }
}
```

We start once again with the initialization of an empty variable named $PlotList. This variable will hold all data we need to plot the customer on the map.

We loop through each Customer in $CustomerArray, we check if the customer has a Latitude not equal to 0 and isn't empty to only include customers for which the coordinates have been found using the OpenStreetMap API.

Next we create another array to hold our customer data which we use to plot the markers with and start by population the required fields.

Then if the CustomerParentID is equal to 50, we only write the CustomerName and write an ampy value to the Location field.

If the CustomerParentID isn't 50, we use a `Where-Object` function to locate the correct parent customer and get the CustomerName to use as CustomerName in the plot and use the normal CustomerName from the loop as the location, since we are currently on a site instead of a customer.

The last part is to generate the HTML data and save it to a file.
```powershell
$HTMLExport = @"
<!DOCTYPE html>
<html>
<head>
  <title>N-Central Customer Map</title>
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
    let markers = $(ConvertTo-Json $PlotList -Compress);
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
On line 48 you find the small bit of powershell `$(ConvertTo-Json $PlotList -Compress)`. This will convert the $PlotList to a JSON Object to use as markers.

The last line is to export it all to a HTML file to display in your browser.

When you put this all together, you will get the result of the file you can download below.

If you execute this script, you will get an interactive map where all customers are plotted onto an OpenStreetMap map.

When you hover on a marker, you will see what customer it is including it's site name if it is a site of a customer.

# Downloads
Download PS1 file: [GitHub](https://github.com/eagle00789/N-Central/blob/master/OpenStreetMap%20Customers/OpenStreetMap%20Customers.ps1)

# Updates
- 22-04-2024 - Updated code and text (with some input from Adriaan Sluis)