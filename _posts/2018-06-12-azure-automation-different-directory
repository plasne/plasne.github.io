---
layout: post
title: Create an Azure Automation Connection to Another Directory
---

My customer is a software ISV that deploys solutions for their customers in Azure. The customers have their own Azure AD directories that contain Applications and the ISV wants to know when those expire. They wrote a PowerShell script to run in Azure Automation that connected to Azure AD like this...

```powershell
$conn = Get-AutomationConnection -Name "AzureRunAsConnection"

Connect-AzureAD `
    -TenantId $conn.TenantId `
    -ApplicationId $conn.ApplicationId `
    -CertificateThumbprint $conn.CertificateThumbprint
```

...but of course, that gives them the Application information for their directory, not the customer's directory. This post describes the process to create an account in the customer's directory that can be used by the ISV's Automation Account.

# Create a Self-Signed Certificate

You can use these PowerShell commands to create a certificate in PFX and CER format (both will be needed)...

```powershell
$cert = New-SelfSignedCertificate -DnsName "Customer01" -CertStoreLocation cert:\LocalMachine\My `
        -KeyExportPolicy Exportable -Provider "Microsoft Enhanced RSA and AES Cryptographic Provider" `
        -NotAfter (Get-Date).AddMonths(12) -HashAlgorithm SHA256

$password = ConvertTo-SecureString "mypassword" -AsPlainText -Force

Export-PfxCertificate -Cert ("Cert:\localmachine\my\" + $cert.Thumbprint) -FilePath "c:\temp\cert.pfx" -Password $password -Force | Write-Verbose

Export-Certificate -Cert ("Cert:\localmachine\my\" + $cert.Thumbprint) -FilePath "c:\temp\cert.cer" -Type CERT | Write-Verbose
```

# Create an Application

Next, a Web API Application will need to be created in the customer's directory. Sign into portal.azure.com with an account that has Global Administrator access for the customer's Azure AD directory:

1. Click on "Azure Active Directory"
2. Click on "App registrations"
3. Click on "New application registration"
4. Type a "Name" and use "http://fakeuri" for the "Sign-on URL"
5. Click on "Create"
6. Make note of the "Application ID", we will need this later
7. Click on "Settings"
8. Click on "Required permissions"
9. Add "Windows Azure Active Directory" or click on it if its already there
10. Check "Read directory data" and uncheck everything else
11. Click on "Save"
12. Click on "Keys"
13. Click on "Upload Public Key"
14. Select your "cert.cer" file created above
15. Click on "Save"

# Provide Administrative Consent

You might have noticed that all application rights, required consent from a Global Administrator of the directory, so we must now provide that.

You will craft a URL like this:

```
https://login.microsoftonline.com/{directory}.onmicrosoft.com/oauth2/authorize?response_type=code&client_id={appid}&redirect_uri=http%3A%2F%2Ffakeuri&state=not_needed&resource=https%3A%2F%2Fgraph.windows.net&prompt=admin_consent
```

Replace {directory} with the name of your Azure AD directory. Replace {appid} with the Application ID.

Go to that URL in a browser, sign-in with a Global Administrative account, and it should ask you for consent. You can provide that and then it will redirect to http://fakeuri, which will of course fail, but it doesn’t matter, the consent has already been provided.

# Create Connection

Sign into portal.azure.com with an account that has access to your Automation Account. Go to your Automation Account and do the following:

1. Click on "Certificates"
2. Click on "Add a certificate"
3. Name the certificate and select the "cert.pfx" file created above
4. Type the password and leave it not exportable
5. Click on "Create", make note of the thumbprint for step #12
6. Click on "Connections"
7. Click on "Add a connection"
8. Name your connection (remember it for later, ex. Customer01)
9. Select "AzureServicePrincipal" for the "Type"
10. Supply the Application ID (the Web API Application ID you created earlier)
11. Supply the Tenant ID for the customer's directory (ex. something.onmicrosoft.com)
12. Supply the Certificate Thumbprint (you should have seen it when you uploaded the pfx certificate)
13. Supply the Subscription ID of the customer's subscription (you can just type anything if they don't have one)
14. Click on "Create"

# Use the Connection

To use the connection, simply specify the connection you created instead of the RunAs account.

```powershell
$conn = Get-AutomationConnection -Name "Customer01"

Connect-AzureAD `
    -TenantId $conn.TenantId `
    -ApplicationId $conn.ApplicationId `
    -CertificateThumbprint $conn.CertificateThumbprint
```
