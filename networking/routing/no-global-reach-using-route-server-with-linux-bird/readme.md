# Connecting from On Premises to Azure VMware Solution in regions where no Global Reach feature is availble

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

The guide shows all IP addresses used, please update those to fit your scenario.

The repository also contains the configuration files for each device.

### Network Diagram

![network architecture](network-architecture.jpg)

You can download an editable version of the diagram [here](network-architecture.vsdx)

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

* deploy four VMs (this guide used [Ubuntu 18.04 LTS](https://portal.azure.com/#create/Canonical.UbuntuServer1804LTS-ARM) VMs), use Availability Sets to separate the VMs on different hosts

  * hub-11 (connected to Routing Subnet in hub-vnet)
  * hub-12 (connected to Routing Subnet in hub-vnet)
  * route-11 (connected to Routing Subnet in avs-vnet)
  * route-12 (connected to Routing Subnet in avs-vnet)

* stop all VMs
* after your deployed the VMs please make following changes to each VM

  * add a second network card that is connected to the Routing-Internal Subnet
  * enable IP forwarding on each NIC (8 NICs), see [here](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-network-interface#enable-or-disable-ip-forwarding) how to enable/disable IP forwarding
  * be sure that "Accelerated Networking" is enabled on all NICs, look [here](https://docs.microsoft.com/en-us/azure/virtual-network/create-vm-accelerated-networking-cli#create-a-linux-vm-with-azure-accelerated-networking) for help
  * to enable [Serial Console Access](https://docs.microsoft.com/en-us/troubleshoot/azure/virtual-machines/serial-console-overview) change the Diagnostic Settings of each VM to use a selfdefined storage account

* start all VMs and run these command's to install bird, strongswan and ifupdown.

  By default the image is using netplan to manage interfaces, but netplan as of now doesn't support vxlan which is used to interconnect the VMs.

  ```bash
  apt-get update
  apt-get upgrade
  apt install -y bird strongswan ifupdown
  ```
  
  This guide is using [**BIRD**](https://bird.network.cz/) as BGP software.

* create /etc/network/interfaces file, here a sample file.

    eth0 is connected to Route Server
    eth1 is used to connect the VMs, MTU is increased to 1550 to have vxlan interfaces with MTU 1500.
    The route on eth1 is added to send traffic to VMs in other vNET through eth1.

    ```none
        10.92.0.0/30  used for hub-11 (10.92.0.1) to route-11 (10.92.0.2)
        10.92.0.4/30  used for hub-11 (10.92.0.5) to route-12 (10.92.0.6)
        10.92.0.8/30  used for hub-12 (10.92.0.9) to route-11 (10.92.0.10)
        10.92.0.12/30 used for hub-12 (10.92.0.13) to route-12 (10.92.0.14)
    ```

    here the content for hub-11.

    ```none
    source /etc/network/interfaces.d/*

    # The loopback network interface
    auto lo
    iface lo inet loopback

    auto eth0
    iface eth0 inet dhcp

    auto eth1
    iface eth1 inet dhcp
        post-up /sbin/ifconfig eth1 mtu 1550; route add -net 10.91.3.0 netmask 255.255.255.224 gw 10.90.3.1

    auto vxlan1
    iface vxlan1 inet static
        mtu 1500
        pre-up ip link add vxlan1 type vxlan id 1 remote 10.91.3.4 dev eth1 || true
        up ip link set vxlan1 up
        down ip link set vxlan1 down
        post-down ip link del vxlan1 || true
        address 10.92.0.1
        netmask 255.255.255.252

    auto vxlan2
    iface vxlan2 inet static
        mtu 1500
        pre-up ip link add vxlan2 type vxlan id 2 remote 10.91.3.5 dev eth1 || true
        up ip link set vxlan2 up
        down ip link set vxlan2 down
        post-down ip link del vxlan2 || true
        address 10.92.0.5
        netmask 255.255.255.252
    ```

* change system config from netplan to ifupdown

  ```none
  ifdown --force eth0 lo && ifup -a
  systemctl unmask networking
  systemctl enable networking
  systemctl restart networking

  systemctl stop systemd-networkd.socket systemd-networkd networkd-dispatcher systemd-networkd-wait-online
  systemctl disable systemd-networkd.socket systemd-networkd networkd-dispatcher systemd-networkd-wait-online
  systemctl mask systemd-networkd.socket systemd-networkd networkd-dispatcher systemd-networkd-wait-online
  apt-get --assume-yes purge nplan netplan.io

  rm /etc/netplan/50-cloud-init.yaml
  ```

* enable IP forwarding on all Linux VMs by adding these entries to /etc/sysctl.conf

  ```none
  net.ipv4.ip_forward=1
  net.ipv4.conf.all.forwarding=1
  net.ipv4.conf.default.forwarding=1

  net.ipv4.conf.all.rp_filter = 0
  net.ipv4.conf.default.rp_filter = 0
  net.ipv4.conf.eth0.rp_filter = 0
  net.ipv4.conf.eth1.rp_filter = 0
  net.ipv4.conf.lo.rp_filter = 0
  net.ipv4.conf.vxlan1.rp_filter = 0
  net.ipv4.conf.vxlan2.rp_filter = 0
  ```

* reboot each system

  ```bash
  reboot
  ```

* test connectivity on vxlan interface

  ```none
  root@hub-11:~# ping 10.92.0.2
  PING 10.92.0.2 (10.92.0.2) 56(84) bytes of data.
  64 bytes from 10.92.0.2: icmp_seq=1 ttl=64 time=1.50 ms
  64 bytes from 10.92.0.2: icmp_seq=2 ttl=64 time=0.259 ms
  64 bytes from 10.92.0.2: icmp_seq=3 ttl=64 time=0.276 ms
  ^C
  --- 10.92.0.2 ping statistics ---
  3 packets transmitted, 3 received, 0% packet loss, time 2027ms
  rtt min/avg/max/mdev = 0.259/0.679/1.504/0.583 ms
  root@hub-11:~#
  ```

* enable Branch-to-Branch configuration in both Route Servers
* create peers in Azure Route Server for each instance

  * hub Route Server to hub-11 and hub-12
  * avs Route Server to route-11 and route-12

* Configure BIRD on each VM by editing /etc/bird/bird.conf file

  ASN 65101 for hub vNET

  ASN 65103 for avs vNET

  ```none
  log syslog { debug, trace, info, remote, warning, error, auth, fatal, bug };

  # local router id
  router id 10.90.2.4;

  protocol device {
    scan time 10;
  }

  # do not import routes from kernel but send learned routes to kernel for routing
  protocol kernel {
    export all;
    import none;
  }

  protocol direct {
    disabled;
  }

  # create static routes for vxlan interfaces and to route server subnet,
  # required by bird even if default route would also work
  protocol static {
    route 10.90.1.32/27 via 10.90.2.1;
    route 10.92.0.0/30 via "vxlan1";
    route 10.92.0.4/30 via "vxlan2";
  }

  # inbound from route server replace the AS path with own ASN
  # required as both route servers have same ASN 65515
  filter from_routeserver {
    bgp_path.empty;
    bgp_path.prepend(65101);
    accept;
  }

  filter to_routeserver {
    # Drop long prefixes
    if ( net ~ [ 0.0.0.0/0{30,32} ] ) then { reject; }
    else accept;
  }

  # template for route server and route VMs
  template bgp PEERS {
    local as 65101;
    multihop;
  }

  # template for internal BGP session
  template bgp ibgp {
    local as 65101;
    next hop self;
  }

  # connection to route server IP 1
  protocol bgp routeserver1 from PEERS {
    neighbor 10.90.1.36 as 65515;
    import filter from_routeserver;
    export filter to_routeserver;
  }

  # connection to route server IP 2
  protocol bgp routeserver2 from PEERS {
    neighbor 10.90.1.37 as 65515;
    import filter from_routeserver;
    export filter to_routeserver;
  }

  # connection to hub12 through iBGP
  protocol bgp hub12 from ibgp {
    neighbor 10.90.2.5 as 65101;
  }

  # connection to route11 through vxlan
  protocol bgp route11 from PEERS {
    neighbor 10.92.0.2 as 65103;
    import all;
    export all;
  }

  # connection to route12 through vxlan
  protocol bgp route12 from PEERS {
    neighbor 10.92.0.6 as 65103;
    import all;
    export all;
  }
  ```

* Start BIRD on all VMs

  ```none
  systemctl enable bird
  systemctl start bird
  ```

* Start birdc to see status

  ```none
    root@hub-11:~# birdc
    BIRD 1.6.3 ready.
    bird> show route
    10.92.0.12/30      via 10.92.0.6 on vxlan2 [route12 08:38:53] * (100/0) [AS65103i]
    10.92.0.8/30       via 10.92.0.2 on vxlan1 [route11 08:33:48] * (100/0) [AS65103i]
    10.90.0.0/22       via 10.90.2.1 on eth0 [routeserver1 08:31:51 from 10.90.1.36] * (100/?) [AS65101i]
                    via 10.90.2.1 on eth0 [routeserver2 08:31:47 from 10.90.1.37] (100/?) [AS65101i]
    10.220.9.4/30      via 10.90.2.1 on eth0 [routeserver1 08:31:51 from 10.90.1.36] * (100/?) [AS65101i]
                    via 10.90.2.1 on eth0 [routeserver2 08:31:47 from 10.90.1.37] (100/?) [AS65101i]
    10.91.0.0/22       via 10.92.0.2 on vxlan1 [route11 08:33:48] * (100/0) [AS65103i]
                    via 10.92.0.6 on vxlan2 [route12 08:38:54] (100/0) [AS65103i]
    10.92.0.4/30       dev vxlan2 [static1 08:31:46] ! (200)
                    via 10.92.0.6 on vxlan2 [route12 08:38:53] (100/0) [AS65103i]
    10.220.9.0/30      via 10.90.2.1 on eth0 [routeserver1 08:31:51 from 10.90.1.36] * (100/?) [AS65101i]
                    via 10.90.2.1 on eth0 [routeserver2 08:31:47 from 10.90.1.37] (100/?) [AS65101i]
    10.92.0.0/30       dev vxlan1 [static1 08:31:46] ! (200)
                    via 10.92.0.2 on vxlan1 [route11 08:33:48] (100/0) [AS65103i]
    10.220.0.0/24      via 10.90.2.1 on eth0 [routeserver1 08:31:51 from 10.90.1.36] * (100/?) [AS65101i]
                    via 10.90.2.1 on eth0 [routeserver2 08:31:47 from 10.90.1.37] (100/?) [AS65101i]
    10.90.1.32/27      via 10.90.2.1 on eth0 [static1 08:31:46] * (200)
    10.91.1.32/27      via 10.92.0.2 on vxlan1 [route11 08:33:48] * (100/0) [AS65103i]
                    via 10.92.0.6 on vxlan2 [route12 08:38:53] (100/0) [AS65103i]
    10.120.2.0/25      via 10.92.0.2 on vxlan1 [route11 08:33:48] * (100/0) [AS65103?]
                    via 10.92.0.6 on vxlan2 [route12 08:38:54] (100/0) [AS65103?]
    10.120.3.0/26      via 10.92.0.2 on vxlan1 [route11 08:33:48] * (100/0) [AS65103?]
                    via 10.92.0.6 on vxlan2 [route12 08:38:54] (100/0) [AS65103?]
    10.120.0.64/26     via 10.92.0.2 on vxlan1 [route11 08:33:48] * (100/0) [AS65103?]
                    via 10.92.0.6 on vxlan2 [route12 08:38:54] (100/0) [AS65103?]
    10.120.0.0/26      via 10.92.0.2 on vxlan1 [route11 08:33:48] * (100/0) [AS65103?]
                    via 10.92.0.6 on vxlan2 [route12 08:38:54] (100/0) [AS65103?]
    10.120.1.0/25      via 10.92.0.2 on vxlan1 [route11 08:33:48] * (100/0) [AS65103?]
                    via 10.92.0.6 on vxlan2 [route12 08:38:54] (100/0) [AS65103?]
    10.120.1.128/25    via 10.92.0.2 on vxlan1 [route11 08:33:48] * (100/0) [AS65103?]
                    via 10.92.0.6 on vxlan2 [route12 08:38:54] (100/0) [AS65103?]
    10.120.6.0/24      via 10.92.0.2 on vxlan1 [route11 08:33:48] * (100/0) [AS65103?]
                    via 10.92.0.6 on vxlan2 [route12 08:38:54] (100/0) [AS65103?]
    10.120.5.0/24      via 10.92.0.2 on vxlan1 [route11 08:33:48] * (100/0) [AS65103?]
                    via 10.92.0.6 on vxlan2 [route12 08:38:54] (100/0) [AS65103?]
    bird>
  ```

  The route table shows all required entries, in this sample

  ```none
  10.220.0.* connects to on premises
  10.120.*.* connects to AVS
  ```

* Check routing tables for route server

  show learned routes from on premise

  ```none
  C:\>az network routeserver peering list-advertised-routes -n hub-11 --routeserver hub-routeserver -g avs-no-globalreach --query "RouteServiceRole_IN_0" -o table
  LocalAddress    Network        NextHop     Origin    AsPath             Weight
  --------------  -------------  ----------  --------  -----------------  --------
  10.90.1.36      10.90.0.0/22   10.90.1.36  Igp       65515              0
  10.90.1.36      10.220.0.0/24  10.90.1.36  Igp       65515-12076-64631  0
  10.90.1.36      10.220.9.0/30  10.90.1.36  Igp       65515-12076-64631  0
  10.90.1.36      10.220.9.4/30  10.90.1.36  Igp       65515-12076-64631  0
  
  C:\>
  ```
  
  show advertised routes from hub-11 VM

  ```none
  C:\>az network routeserver peering list-learned-routes -n hub-11 --routeserver hub-routeserver -g avs-no-globalreach --query "RouteServiceRole_IN_0" -o table
  LocalAddress    Network          NextHop    SourcePeer    Origin    AsPath             Weight
  --------------  ---------------  ---------  ------------  --------  -----------------  --------
  10.90.1.36      10.90.1.32/27    10.90.2.4  10.90.2.4     EBgp      65101              32768
  10.90.1.36      10.91.1.32/27    10.90.2.4  10.90.2.4     EBgp      65101-65103        32768
  10.90.1.36      10.91.0.0/22     10.90.2.4  10.90.2.4     EBgp      65101-65103-65103  32768
  10.90.1.36      10.120.2.0/25    10.90.2.4  10.90.2.4     EBgp      65101-65103-65103  32768
  10.90.1.36      10.120.3.0/26    10.90.2.4  10.90.2.4     EBgp      65101-65103-65103  32768
  10.90.1.36      10.120.0.64/26   10.90.2.4  10.90.2.4     EBgp      65101-65103-65103  32768
  10.90.1.36      10.120.0.0/26    10.90.2.4  10.90.2.4     EBgp      65101-65103-65103  32768
  10.90.1.36      10.120.1.0/25    10.90.2.4  10.90.2.4     EBgp      65101-65103-65103  32768
  10.90.1.36      10.120.1.128/25  10.90.2.4  10.90.2.4     EBgp      65101-65103-65103  32768
  10.90.1.36      10.120.6.0/24    10.90.2.4  10.90.2.4     EBgp      65101-65103-65103  32768
  10.90.1.36      10.120.5.0/24    10.90.2.4  10.90.2.4     EBgp      65101-65103-65103  32768

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
      bw  =  223 MB/sec
  tcp_lat:
      latency  =  1.34 ms
  [root@onpremvm ~]#
  ```
