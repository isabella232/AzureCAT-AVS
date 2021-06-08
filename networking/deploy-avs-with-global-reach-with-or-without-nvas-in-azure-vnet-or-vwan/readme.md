# Deploy AVS with Global Reach with or without NVAs in Azure using VNETs or vWAN

This solution is the easiest way of deploying AVS.

You can follow the guide available on the public website, [here](https://docs.microsoft.com/en-us/azure/azure-vmware/concepts-networking) are all the details.

## General description of this solution

Connectivity from On Premises to AVS is going directly to AVS without passing any Azure VNET. The feature for this scenario is Global Reach.

Connectivity from On Premises to Azure native VM is going through the ExpressRoute Gateway deployed in a VNET or in vWAN.

Connectivity from Azure native VMs to AVS is going through the same ExpressRoute Gateway as from On Premisis to Azure native VMs.

## Limitations

* There is no firewall inspection possible before you enter AVS
* you can deploy a firewall in AVS between the Tier0 and the Tier1 router
* if you use Global Reach all routes announces to AVS will also be distributes through Global Reach to other AVS regions connected using Global Reach