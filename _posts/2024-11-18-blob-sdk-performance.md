---
layout: post
title: Blob SDK Performance
---

Recently I discovered a significant performance difference using the C# Blob SDK (Azure.Storage.Blobs 12.22.2).

Reading blob this way...

```csharp
BlobServiceClient blobServiceClient = new BlobServiceClient(new Uri(blobServiceEndpoint), cred);
BlobContainerClient containerClient = blobServiceClient.GetBlobContainerClient(containerName);
BlobClient blobClient0 = containerClient.GetBlobClient(blobName0);
await blobClient0.DownloadToAsync(downloadFilePath0);
```

...was around 50ms on all calls over the first (which included getting an access token). However, reading the blob this way...

```csharp
BlobClient blobClient0 = new BlobClient(new System.Uri($"{blobServiceEndpoint}/{containerName}/{blobName0}"), cred);
await blobClient0.DownloadToAsync(downloadFilePath0);
```

...was around 425ms on all calls over the first.

In both cases, permissions were obtained using DefaultAzureCredential, as shown here...

```csharp
var cred = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    ExcludeAzureDeveloperCliCredential = true,
    ExcludeAzurePowerShellCredential = true,
    ExcludeEnvironmentCredential = true,
    ExcludeInteractiveBrowserCredential = true,
    ExcludeManagedIdentityCredential = true,
    ExcludeSharedTokenCacheCredential = true,
    ExcludeVisualStudioCodeCredential = true,
    ExcludeVisualStudioCredential = true,
    ExcludeWorkloadIdentityCredential = true
});
```
