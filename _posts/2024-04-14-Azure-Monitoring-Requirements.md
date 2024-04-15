---
title: Azure Monitoring Requirements
date: 2024-04-14 12:00:00 +0200
categories: [PowerShell, Azure]
tags: [powershell, azure]
image:
  path: /assets/images/logos/Microsoft_Azure-Logo.svg
  alt: Microsoft Azure Logo.
---
> Visit this post regularly to check for updates.
{: .prompt-tip }

If you get to a point to monitor Microsoft Azure items, you can connect to Azure using a username and password with MFA exclusion, but you will also need an app registration to connect with. Why not go the additional extra step and also generate a certificate, so that you can use this certificate to authenticate to the tenant in Azure. Below you can find a detailed explanation how to setup this app including it's certificate.

1. Logon to [https://portal.azure.com](https://portal.azure.com)
2. Go to Microsoft Entra ID (formerly Azure Active Directory)
3. Go to App Registrations
4. Click on New Registration on the top of the screen
![Naamloos](/assets/images/Azure-1.png)
5. Give the application a clear name. When you create the application for use with NCentral please always use the name NCentral Azure Monitoring. That way we always know why an application was registered.
![Naamloos-1](/assets/images/Azure-2.png)
   The rest of the form must be left as-is
6. The base app is now created.
![Naamloos](/assets/images/Azure-3.png)
   Unfortunatly, the app currently doesn’t have any permissions to do anything by default.
7. Go to API Permissions on the left side and click on Add Permission
8. When adding the permissions, choose Microsoft Graph and then choose Application permissions
9. Search for the listed API permissions below and add a checkmark next to it.
    - Application.Read.All
    - AuditLog.Read.All
    - DeviceManagementConfiguration.Read.All
    - DeviceManagementServiceConfig.Read.All
    - Group.Read.All
    - IdentityRiskEvent.Read.All
    - IdentityRiskyUser.Read.All
    - Policy.Read.ConditionalAccess
    - Policy.Read.All
    - ServiceHealth.Read.All
    - ServiceMessage.Read.All
10. Click on Add permissions at the bottom right
11. Some tenants are locked down, so ask a global admin user for that tenant to grant the Admin Consent, or click the button yourself if you have the rights to do this.
12. After the admin consent has been given, this screen should look like this:
![Naamloos-1](/assets/images/Azure-4.png)
13. Now go to Certificates & secrets
14. Open the computer that is going to monitor the Azure services.
15. On the server open the services list.
16. Check which user is set as Logon As for the service Automation Manager Agent
![Naamloos](/assets/images/Azure-5.png)
17. Open Powershell on the computer that is going to monitor the specified services as the specified user above and execute the code below.
If the user is listed as Local System (like the example above) then open Take Control, go to the System Shell tab and type powershell before executing the commands below:
```powershell
$certname = "NCentral Azure Monitoring"
$cert = New-SelfSignedCertificate -Subject "CN=$certname" -CertStoreLocation "Cert:\CurrentUser\My" -KeyExportPolicy Exportable -KeySpec Signature -KeyLength 2048 -KeyAlgorithm RSA -HashAlgorithm SHA256
Export-Certificate -Cert $cert -FilePath "C:\Temp\$certname.cer"
``` 
18. In the C:\Temp folder on the computer that you used to generate this certificate, is now a .cer-file with the name NCentral Azure Monitoring.cer present. Upload this file to the Certificates & Secrets in Azure.
19. Go back to Microsoft Entra ID
20. Choose Roles and administrators
21. Search for the role Global Reader and click on the name
![Naamloos-1](/assets/images/Azure-6.png)
22. Click on Add assignments
![Naamloos](/assets/images/Azure-7.png)
23. Seach for NCentral Azure Monitoring, click the checkmark in front of it and click Add
![Naamloos-1](/assets/images/Azure-8.png)
24. Go to NCentral and make sure you are at the Managed Services level.
25. Search for the server that you are using to monitor the Azure services with, without going to customer or site level first:
![Naamloos](/assets/images/Azure-9.png)
26. In the example above, you can see that the server we are going to use, resides at the customer level, because no Site has been specified in this example.
27. In NCentral, now go to Administration > Customers (when the server resides at customer level) or go to the customer level and then go to Administration > Sites if the server resides at a site specific level.
28. Edit the Customer (or Site) and go to the tab Custom Properties. Here you can find 3 empty properties:
![Naamloos-1](/assets/images/Azure-10.png)
29. Fill these properties as following:
    - **Azure Certificate Thumbprint**: Go back to Azure and select the application **NCentral Azure Monitoring** and go to the option **Certificates & secrets**. There you will find the thumbprint. Just double click it, to select the entire Thumbprint (even though it isn’t completely visible, it will select the entire thumbprint) and copy it. Next past it in the **Azure Certificate Thumbprint** property in NCentral.
    - **Azure Client ID**: Go back to Azure and go to the tab Overview of the application **NCentral Azure Monitoring**. At the top you will find the **Application (client) ID**. Move the mouse over the Application (client) ID to make a copy button visible. Click the copy button and paste the value in the **Azure Client ID** field in NCentral.
    - **Azure Tenant ID**: Go back to Azure and go to the tab Overview of the application **NCentral Azure Monitoring**. At the top you will find the **Directory (tenant) ID**. Move the mouse over the **Directory (tenant) ID** to make a copy button visible. Click the copy button and paste the value in the **Azure Tenant ID** field in NCentral.
30. Hit Save at the bottom left to save the now filled custom properties