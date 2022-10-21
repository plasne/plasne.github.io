---
layout: post
title: Front End API
---

This document discusses a pattern whereby a collection of microservice APIs are only accessible via a private network and thus all external clients must contact a single frontend API which acts as a gateway to those private services. The role of this frontend API includes:

- Providing gateway features (access to internal APIs, SSL offload, change in auth - ex. mTLS).
- It can be aligned with functionality, whereas the backend APIs are typically aligned with the implementation.
- It can mash-up data from backend APIs, change protocol, etc.
- It can provide operations tailored to UIs so that they can concentrate on presentation instead of functionality.
- Enhancing, offloading, or changing authentication and authorization.

This approach can also provide some of these benefits:

- It simplifies security by reducing surface area.
- It allows for different tech stacks to be used  at different layers (ex. Node.js for Frontend API and C# for Backend API).

However, it also introduces these challenges:

- Code has to be written even for simple scenarios like reverse proxy.
- Some changes can require redeployment of both frontend and backend APIs.

There are some alternatives to consider to this approach:

- All microservice APIs could be given public IPs, but of course, this increases the surface area for attack.

- A reverse proxy, like NGINX, could be used as a reverse proxy, even if it doesn't provide all the capabilities of a frontend API.

- An Azure API Management gateway could be used as a reverse proxy and provide some advanced functionality via policies, even if it doesn't provide all the capabilities of a frontend API.

- All traffic could be routed through APIM to provide the illusion of a single API. The APIM could then route to both backend and frontend APIs to satify requests.
