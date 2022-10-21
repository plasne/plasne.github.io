---
layout: post
title: API Versioning
---

This document will describe the typical approach to versioning an API and why that might not be the best approach (particularly for microservices) while offering an alternative.

This guidance generally refers to major versions (ie. versions that break functionality). While this approach could be used with minor versions and bug fixes, that is generally not necessary.

## Traditional Approach

Commonly developers will create a project for their API. Then create controllers (or similar) in that API with a route that includes a version (ex. /api/v1/my-controller). When they have breaking functionality, they might create a separate controller (or similar) with a route and new version number (ex. /api/v2/my-controller).

There are a number of issues with this approach, particularly for microservices solutions, including:

- __Bloat__: Using this approach, we now have a lot more code in our API. More code means: (a) longer compile times, (b) longer unit test times, (c) a greater chance for unintended consequences due to change, (d) larger deployment packages, (e) more resources consumed on deployment targets, and so on.

- __Complex Code Reuse__: Typically not all the code will be contained in the controller - there will likely be classes of functionality that could be reused (for example, a load or validation component). Some of that code may be reusable across version. However, as soon as there is any change in functionality, the code has to be branched or duplicated to ensure that prior versions of the code operate as they did before the change. It doesn't take many versions of code before this can become a complicated mess.

- __More Unit Tests__: Having multiple versions of code in the same API also means that the number of unit tests must continue to grow such that each version has a full set of tests.

## Recommended Approach

Contrast the above with a very simple microservice that serves one purpose and has only the latest version of the code available. We can support this vision by:

- __Tagging__: Whenever a version is released, it should be tagged in Git. In this way, it is easy to see what code is in a particular version and it is possible to redeploy a specific version if needed. Ref: <https://git-scm.com/book/en/v2/Git-Basics-Tagging>.

- __Multiple Deployments__: It is likely that a service may be deployed multiple times with different versions. This is particularly true of a microservices architecture whereby other services may be taking a dependency on a specific version of this service. Therefore, you might have serviceA.v1, serviceA.v2, etc all deployed at the same time.

- __Routing__: With multiple deployments (each a different version), it becomes necessary for services to make calls to specific versions of other services. There are multiple options:

  - Each service could be named with the version, ex. serviceA-v1.

  - If path-based routing is supported by your proxy, you could use the version in the path, ex. /v1/serviceA.

  - You could use a service mesh to route traffic using more advanced rules. For example, using Istio and Destination Rules, you could specify a destination that used a version label. Ref: <https://istio.io/latest/docs/reference/config/networking/destination-rule/>.

  - Azure API Management (APIM) or a similar solution could be used to route to the appropriate version of a service. Ref: <https://learn.microsoft.com/en-us/azure/api-management/api-management-versions>.

- __Deprecation__: Once all services have upgraded their dependencies to newer versions, older deployments of this API can simply be removed from the environment.

There are significant improvements to the developer experience with this approach. To write a new version of an API, the developer just needs to:

- Ensure the prior version is tagged in Git.

- Modify all code and tests as desired.

- Remove any code and tests that are no longer used.

- Write the manifest to deployment and route to this new version.

- Tag this new version in Git.
