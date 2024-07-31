---
layout: post
title: Dockerfile for AMD64
---

It is now commonplace to have ARM64 chips (M-series on Mac, Copilot PC, etc.) but still often necessary to build a .NET container that will run in AMD64.

Here is a Dockerfile that has been constructed for this purpose:

```dockerfile
# create the build container
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG TARGETARCH
LABEL stage=build
WORKDIR /ui
COPY ui .
WORKDIR /catalog
COPY catalog .
RUN apt-get update && \
    apt-get install -y curl && \
    curl -sL https://deb.nodesource.com/setup_20.x | bash - && \
    apt-get install -y nodejs && \
    apt-get clean
RUN ./refresh-ui.sh
RUN dotnet publish -c Release -o out -a $TARGETARCH

# create the runtime container
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /catalog/out .
COPY --from=build /catalog/wwwroot ./wwwroot
EXPOSE 80
ENTRYPOINT ["dotnet", "exp-catalog.dll"]
```

To build the image:

```bash
docker build -t plexchatcopilotpoceastusacr.azurecr.io/catalog:1.7.15 --platform linux/amd64 .
```

Let's break down what is different:

- When building the image the desired platform is specified.

- The build container uses $BUILDPLATFORM which is special Docker variable that will be "linux/arm64". In other words, the build container is running on the platform of your local computer (not the target).

- The runtime container has no such specification so it is running on the target platform of "linux/amd64".

- There is another special Docker variable of $TARGETARCH which is set by the `ARG TARGETARCH` command.

- All `dotnet` commands (such as `dotnet restore`, `dotnet build`, `dotnet publish`) need to include `-a $TARGETARCH`. This ensures the .NET libraries and compilation process is performed for the target platform of "amd64".

For more complex scenarios such as a multiple target platforms in the same image, see [buildx](https://github.com/docker/buildx).