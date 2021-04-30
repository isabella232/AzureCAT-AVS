# Connecting from On Premises to Azure VMware Solution in regions where no Global Reach feature is availble using Cisco CSR1000V

## Intro

Azure VMware Soltuion is physical hardware running in Azure datacenters managed by Microsoft.

To connect physical components to the Azure backbone we are using a technology called ExpressRoute.

Learn more about ExpressRoute [here](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-introduction).

As the Virtual Network Gateways connected to an ExpressRoute circuit can't transit traffic between two circuits (one circuit is going to on premisis, one is going to Azure VMware Solution) Microsoft uses the Global Reach feature to directly connect the on premisis circuit to AVS.
To learn more about Global Reach click [here](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-global-reach).

In certain cases the Global Reach feature is not available in all regions. Please look [here](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-global-reach#availability) to see where Global Reach is available.

This guide describes how to use virtual machines and the new [Azure Route Server](https://docs.microsoft.com/en-us/azure/route-server/overview) to connect your on premisis connection to Azure VMware Solution in cases where Global Reach is not available.

You can also use this guide if you want to route the traffic from on premises to AVS through NVAs.

---

## Getting Started

To get started create four virtual machines for a highly available implementation.
Look [here](https://docs.microsoft.com/en-us/azure/virtual-machines/sizes-compute) to learn more about Virtual Machines and their network throughput, in this guide all VMs where D4s_v3 instances with 2000 MBit/s, so roughly 200 MByte/s.

This guide used Cisco CSR1000V AX enabled images available in Azure marketplace.

To learn more about Cisco CSR1000V on Azure visit [this](https://www.cisco.com/c/en/us/td/docs/routers/csr1000/software/azu/b_csr1000config-azure.html) website.

The guide shows all IP addresses used, please update those to fit your scenario.

### Create Hub vNET

The Hub vNET connects to on premises and contains the hub VMs.

* hub-vnet (address space 10.90.0.0/22)

    incl following subnets

  * default (10.90.0.0/25) for test VMs
  * GatewaySubnet (10.90.1.0/27) for vNET Gateway
  * RouteServerSubnet (10.90.1.32/27) for Azure Route Server
  * Routing (10.90.2.0/27) to connect to Route Server
  * Routing-Internal (10.90.3.0/27) to connect to VMs running in other vNET

* avs-vnet (address space 10.91.0.0/22)

    incl following subnets

  * default (10.91.0.0/25) for test VMs
  * GatewaySubnet (10.91.1.0/27) for vNET Gateway
  * RouteServerSubnet (10.91.1.32/27) for Azure Route Server
  * Routing (10.91.2.0/27) to connect to Route Server
  * Routing-Internal (10.91.3.0/27) to connect to VMs running in other vNET

* add Virtual Network Gateway to hub-vnet

    For best performance use "Ultra Performance Gateway" or "ErGw3AZ" to enable FastPath feature.

* add Virtual Network Gateway to avs-vnet

    For best performance use "Ultra Performance Gateway" or "ErGw3AZ" to enable FastPath feature.

* peer hub-vnet and avs-vnet
* create a User Defined Route with "Propagate gateway routes" set to No
* apply the User Defined Route (UDR) to both "Routing-Internal" Subnets
* deploy Azure Route Server in both vNETs, to access Azure Route Server in portal use this [link](https://aka.ms/routeserver)

* deploy four VMs (this guide used [Cisco CSR1000V 17.3.3 AX](https://ms.portal.azure.com/#create/cisco.cisco-csr-1000v17_3_3-payg-ax) VMs), use Availability Sets to separate the VMs on different hosts

  * hub11 (connected to Routing Subnet in hub-vnet)
  * hub12 (connected to Routing Subnet in hub-vnet)
  * route11 (connected to Routing Subnet in avs-vnet)
  * route12 (connected to Routing Subnet in avs-vnet)

* stop all VMs
* after your deployed the VMs please make following changes to each VM

  * add a second network card that is connected to the Routing-Internal Subnet
  * enable IP forwarding on each NIC (8 NICs), see [here](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-network-interface#enable-or-disable-ip-forwarding) how to enable/disable IP forwarding
  * be sure that "Accelerated Networking" is enabled on all NICs, look [here](https://docs.microsoft.com/en-us/azure/virtual-network/create-vm-accelerated-networking-cli#create-a-linux-vm-with-azure-accelerated-networking) for help
  * to enable [Serial Console Access](https://docs.microsoft.com/en-us/troubleshoot/azure/virtual-machines/serial-console-overview) change the Diagnostic Settings of each VM to use a selfdefined storage account

* enable Branch-to-Branch configuration in both Route Servers
* create peers in Azure Route Server for each instance

  * hub Route Server to hub11 and hub12
  * avs Route Server to route11 and route12

* start all VMs, log in and remove not needed configurations, like VRF and outbound NAT

  ```cisco
  conf t
  no vrf definition GS
  no interface VirtualPortGroup0
  no ip access-list standard GS_NAT_ACL

  int gi 1
    no ip nat outside
  ```

* add new config incl
  * ip routing
  * a vxlan-gpe tunnel between the CSR1000V instances
  * MTU 1550 on Interface Gigabit 2 to achieve MTU 1500 on Tunnel
  * BGP config
    * neighbors to the Route Server IP addresses
    * connect to peer on Tunnel interface
  * ip route for interconnect (10.91.3.0/27)
  * ip route to route servers (10.90.1.32/27)
  * route-map for all connections
  * the route-map to the peer removes the ASN 65515 for Route Server and the ASN 12076 from Microsoft Network

  ```cisco
  ip routing

  interface Tunnel1
  ip address 10.92.0.1 255.255.255.252
  tunnel source GigabitEthernet2
  tunnel mode vxlan-gpe ipv4
  tunnel destination 10.91.3.4
  !
  interface GigabitEthernet1
  ip address dhcp
  negotiation auto
  no mop enabled
  no mop sysid
  !
  interface GigabitEthernet2
  mtu 1550
  ip address dhcp
  negotiation auto
  no mop enabled
  no mop sysid
  !
  router bgp 65101
  bgp log-neighbor-changes
  neighbor 10.90.1.36 remote-as 65515
  neighbor 10.90.1.36 ebgp-multihop 255
  neighbor 10.90.1.37 remote-as 65515
  neighbor 10.90.1.37 ebgp-multihop 255
  neighbor 10.92.0.2 remote-as 65103
  !
  address-family ipv4
    neighbor 10.90.1.36 activate
    neighbor 10.90.1.36 route-map from-routeserver in
    neighbor 10.90.1.36 route-map to-routeserver out
    neighbor 10.90.1.37 activate
    neighbor 10.90.1.37 route-map from-routeserver in
    neighbor 10.90.1.37 route-map to-routeserver out
    neighbor 10.92.0.2 activate
    neighbor 10.92.0.2 route-map from-peer in
    neighbor 10.92.0.2 route-map to-peer out
  exit-address-family
  !
  ip route 10.90.1.32 255.255.255.224 10.90.2.1
  ip route 10.91.3.0 255.255.255.224 10.90.3.1
  !
  route-map to-peer permit 10
    set as-path replace 12076
    set as-path replace 65515
  !
  route-map from-peer permit 10
  !
  route-map from-routeserver permit 10
  !
  route-map to-routeserver permit 10
  ```

* test connectivity on vxlan interface

  ```none
  hub11#ping 10.92.0.2
  Type escape sequence to abort.
  Sending 5, 100-byte ICMP Echos to 10.92.0.2, timeout is 2 seconds:
  !!!!!
  Success rate is 100 percent (5/5), round-trip min/avg/max = 4/4/4 ms
  hub11#
  ```

* Check BGP status

  ```none
  hub11#show ip bgp su
  BGP router identifier 10.92.0.1, local AS number 65101
  BGP table version is 41, main routing table version 41
  13 network entries using 3224 bytes of memory
  17 path entries using 2312 bytes of memory
  4/4 BGP path/bestpath attribute entries using 1152 bytes of memory
  4 BGP AS-PATH entries using 128 bytes of memory
  1 BGP community entries using 24 bytes of memory
  0 BGP route-map cache entries using 0 bytes of memory
  0 BGP filter-list cache entries using 0 bytes of memory
  BGP using 6840 total bytes of memory
  BGP activity 22/9 prefixes, 26/9 paths, scan interval 60 secs
  13 networks peaked at 19:04:14 Apr 29 2021 UTC (13:50:21.808 ago)

  Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
  10.90.1.36      4        65515    1102    1060       41    0    0 15:54:14        4
  10.90.1.37      4        65515    1092    1061       41    0    0 15:54:04        4
  10.92.0.2       4        65103       6       6       32    0    0 00:00:15        9

  hub11#show ip bgp
  BGP table version is 41, local router ID is 10.92.0.1
  Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
                r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
                x best-external, a additional-path, c RIB-compressed,
                t secondary path, L long-lived-stale,
  Origin codes: i - IGP, e - EGP, ? - incomplete
  RPKI validation codes: V valid, I invalid, N Not found

      Network          Next Hop            Metric LocPrf Weight Path
  *    10.90.0.0/22     10.90.1.37                             0 65515 i
  *>                    10.90.1.36                             0 65515 i
  *>   10.91.0.0/22     10.92.0.2                              0 65103 i
  *>   10.120.0.0/26    10.92.0.2                              0 65103 65103 65103 398656 ?
  *>   10.120.0.64/26   10.92.0.2                              0 65103 65103 65103 398656 ?
  *>   10.120.1.0/25    10.92.0.2                              0 65103 65103 65103 398656 ?
  *>   10.120.1.128/25  10.92.0.2                              0 65103 65103 65103 398656 ?
  *>   10.120.2.0/25    10.92.0.2                              0 65103 65103 65103 398656 ?
  *>   10.120.3.0/26    10.92.0.2                              0 65103 65103 65103 398656 ?
  *>   10.120.5.0/24    10.92.0.2                              0 65103 65103 65103 398656 ?
  *>   10.120.6.0/24    10.92.0.2                              0 65103 65103 65103 398656 ?
  *    10.220.0.0/24    10.90.1.37                             0 65515 12076 64631 i
  *>                    10.90.1.36                             0 65515 12076 64631 i
  *    10.220.9.0/30    10.90.1.37                             0 65515 12076 64631 i
  *>                    10.90.1.36                             0 65515 12076 64631 i
  *    10.220.9.4/30    10.90.1.37                             0 65515 12076 64631 i
  *>                    10.90.1.36                             0 65515 12076 64631 i
  hub11#

  ```

  The route table shows all required entries, in this sample

  ```none
  10.220.0.* connects to on premises
  10.120.*.* connects to AVS
  ```

* Check routing tables for route server

  show learned routes from on premise

  ```none
  C:\>az network routeserver peering list-advertised-routes -n hub11 --routeserver azcat-avs-hub-routeserver -g azcat-avs-no-globalreach --query "RouteServiceRole_IN_0" -o table
  LocalAddress    Network          NextHop     Origin      AsPath                                Weight
  --------------  ---------------  ----------  ----------  ------------------------------------  --------
  10.90.1.36      10.90.0.0/22     10.90.1.36  Igp         65515                                 0
  10.90.1.36      10.220.9.4/30    10.90.1.36  Igp         65515-12076-64631                     0
  10.90.1.36      10.220.0.0/24    10.90.1.36  Igp         65515-12076-64631                     0
  10.90.1.36      10.220.9.0/30    10.90.1.36  Igp         65515-12076-64631                     0

  C:\>
  ```
  
  show advertised routes from hub11 VM

  ```none
  C:\>az network routeserver peering list-learned-routes -n hub11 --routeserver azcat-avs-hub-routeserver -g azcat-avs-no-globalreach --query "RouteServiceRole_IN_0" -o table
  LocalAddress    Network          NextHop    SourcePeer    Origin    AsPath                          Weight
  --------------  ---------------  ---------  ------------  --------  ------------------------------  --------
  10.90.1.36      10.91.0.0/22     10.90.2.4  10.90.2.4     EBgp      65101-65103                     32768
  10.90.1.36      10.120.0.0/26    10.90.2.4  10.90.2.4     EBgp      65101-65103-65103-65103-398656  32768
  10.90.1.36      10.120.0.64/26   10.90.2.4  10.90.2.4     EBgp      65101-65103-65103-65103-398656  32768
  10.90.1.36      10.120.1.0/25    10.90.2.4  10.90.2.4     EBgp      65101-65103-65103-65103-398656  32768
  10.90.1.36      10.120.1.128/25  10.90.2.4  10.90.2.4     EBgp      65101-65103-65103-65103-398656  32768
  10.90.1.36      10.120.2.0/25    10.90.2.4  10.90.2.4     EBgp      65101-65103-65103-65103-398656  32768
  10.90.1.36      10.120.3.0/26    10.90.2.4  10.90.2.4     EBgp      65101-65103-65103-65103-398656  32768
  10.90.1.36      10.120.5.0/24    10.90.2.4  10.90.2.4     EBgp      65101-65103-65103-65103-398656  32768
  10.90.1.36      10.120.6.0/24    10.90.2.4  10.90.2.4     EBgp      65101-65103-65103-65103-398656  32768

  C:\>
  ```

* Test connectivity from on premises to Azure VMware Solution, you can use tools like qperf to measure throughput and latency

  Run  qperf on AVS VM
  
  ```none
  qperf
  ```

  Run qperf on onprem VM

  ```none
  [root@onpremvm ~]# qperf 10.120.5.100 -t 10 tcp_bw tcp_lat
  tcp_bw:
      bw  =  222 MB/sec
  tcp_lat:
      latency  =  1.34 ms
  [root@onpremvm ~]#
  ```
