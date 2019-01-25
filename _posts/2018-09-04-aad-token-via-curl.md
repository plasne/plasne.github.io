---
layout: post
title: AAD Token via cURL
---

Someone sent me some code this week on using CURL to get an access token via Azure AD. The format is definately useful so I have included it here.

RESULTS = $(curl -s -X POST https://login.windows.net/${TENANT_ID}/oauth2/token -F grant_type=client_credentials -F client_id=${CLIENT_ID} -F client_secret=${CLIENT_SECRET} -F resource=https://vault.azure.net -F username=${AZURE_USER})

-s specifies that no progress bar is returned, just the JSON
-F sets Form data for the POST
