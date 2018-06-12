---
layout: post
title: Accessing Azure AD via Azure Automation
---

This post describes the method to use Azure AD commands in PowerShell inside an Azure Automation Account. There is a method to connect, provide authorization, and then provide consent.

# Connecting

You can use the following code in Azure Automation to connect to Azure AD:

```powershell
$conn = Get-AutomationConnection -Name "AzureRunAsConnection"

Connect-AzureAD `
    -TenantId $conn.TenantId `
    -ApplicationId $conn.ApplicationId `
    -CertificateThumbprint $conn.CertificateThumbprint
```

# Authorizing

You must also authorize the RunAs account to access Azure AD.

In your Automation Account in portal.azure.com:
1. Click on "Run as accounts"
2. Click on "Azure Run As Account"
3. Make note of the Display Name or Application ID

In portal.azure.com:
1. Click on "Azure Active Directory"
2. Click on "App registrations"
3. Click on your app registration, you can find it by the name or Application ID (step #3 above)
4. Click on "Settings"
5. Click on "Required permissions"
6. Click on "Add"
7. Click on "Select an API"
8. Click on "Windows Azure Active Directory (Microsoft.Azure.ActiveDirectory)"
9. Click on "Select"
10. Click on "Select permissions"
11. Select "Read directory data" (to read data, or whatever rights you need for what you are doing)
12. Click on "Save"
13. Back on the "Settings" pane, click on "Reply URLs"
14. Type a fake URL, use "http://fakeuri"
15. Click "Save"

# Providing Consent

You might have noticed that all application rights, required consent from a Global Administrator of the directory, so we must now provide that.

You will craft a URL like this:

https://login.microsoftonline.com/{directory}.onmicrosoft.com/oauth2/authorize?response_type=code&client_id={appid}&redirect_uri=http%3A%2F%2Ffakeuri&state=not_needed&resource=https%3A%2F%2Fgraph.windows.net&prompt=consent

Replace {directory} with the name of your Azure AD directory. Replace {appid} with the Application ID.

Go to that URL in a browser, sign-in with a Global Administrative account, and it should ask you for consent. You can provide that and then it will redirect to http://fakeuri, which will of course fail, but it doesn't matter, the consent has already been provided.
