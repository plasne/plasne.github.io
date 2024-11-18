---
layout: post
title: GetBlobClient is faster than new BlobClient
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

## Results

```bash
plasne@Peters-MacBook-Pro test % dotnet run
Using new BlobClient...
Downloading blob to local file...
Blob downloaded to /Users/plasne/Documents/test/1100083-2024-01-04.json, after 6522 ms
Downloading blob to local file...
Blob downloaded to /Users/plasne/Documents/test/1100083-2024-01-05.json, after 684 ms
Downloading blob to local file...
Blob downloaded to /Users/plasne/Documents/test/1100083-2024-01-06.json, after 439 ms
Downloading blob to local file...
Blob downloaded to /Users/plasne/Documents/test/1100083-2024-01-07.json, after 427 ms
plasne@Peters-MacBook-Pro test % 
plasne@Peters-MacBook-Pro test % 
plasne@Peters-MacBook-Pro test % dotnet run
Using GetBlobClient...
Downloading blob to local file...
Blob downloaded to /Users/plasne/Documents/test/1100083-2024-01-04.json, after 1114 ms
Downloading blob to local file...
Blob downloaded to /Users/plasne/Documents/test/1100083-2024-01-05.json, after 61 ms
Downloading blob to local file...
Blob downloaded to /Users/plasne/Documents/test/1100083-2024-01-06.json, after 51 ms
Downloading blob to local file...
Blob downloaded to /Users/plasne/Documents/test/1100083-2024-01-07.json, after 53 ms
```