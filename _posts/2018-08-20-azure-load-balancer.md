There are a number of ways to load balance a workload in Azure (2x layer-4 LBs and 1x layer-7 LB for 1st party load balancers). However, it turns out there are some interesting cavaets about how you can access the frontends and/or spread your backends across networks. I made this chart to help:

| Scenario | LB Basic (layer-4) | LB Standard (layer-4) | App GW (layer-7) |
|----------|--------------------|-----------------------|------------------|
| Backends in a single region | yes | yes | yes |
| Backends in a single region across peered VNETs | no | no | yes |
| Backends in multiple zones in a single region and VNET | no | yes | yes |
| Backends in multiple regions across global peered VNETs | no | no | yes |
| Access load balancer frontend across local peered VNETs | yes | yes | yes |
| Access load balancer frontend across global peered VNETs | no | no | yes |
| Access load balancer frontend across an ExpressRoute (on-premises to Azure VNET) | yes | yes | yes |
| Access load balancer frontend across an ExpressRoute (Azure VNET to Azure VNET) | yes | yes | yes |
