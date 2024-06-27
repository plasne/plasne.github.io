---
layout: post
title: gRPC in Container Apps
---

gRPC requires HTTP/2. Container Apps have a transport setting that defaults to "auto" which should detect HTTP/1 or HTTP/2. However, seemingly this doesn't work with gRPC so you must set it to HTTP/2 manually in the ingress section.

![ingress transport](/assets/ingress-transport.png)

You must STOP and START the Container App before this setting takes effect.
