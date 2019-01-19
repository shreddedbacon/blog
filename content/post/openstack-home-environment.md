---
title: "Openstack Home Environment"
date: 2019-01-18T22:27:28+11:00
---

I want to be able to play with BOSH at home better. Leverage similar components that I can get from AWS/Azure/GCP like load balancing, network file shares, s3 compatable buckets, etc... I need Openstack.

I originally deployed Openstack onto a single Dell R720 and used it to deploy some custom BOSH releases I had been working on. But the electricity and heat it produced was too high. Electricity is not cheap in Australia. So I turned it off.

<!--more-->
Recently, while browsing AliExpress, I came across some really cheap MiniPC/NUC style computers. They only have an Intel Celeron J1900 processor, 4 cores (no HT). But they have VT-x, and dual NIC. Downside, only support max 8GB RAM.

I bought 3 of them, they will be the compute nodes. I also found one that had 4 NICs, same processor and specs as the others. I got one of them, it is the controller/network node.

![RackShot](/img/rack-deployment.jpg)
I should really maybe get some different colours for the types of networks instead of all blue..

## How to deploy it?
I was going to go down the path of running the Openstack Undercloud/Overcloud setup, but it became too complicated to try and make work on these 4 little nodes, so I ended up using RDO. RDO is really pretty simple to use, it uses puppet for anyone that wants to get into the nitty gritty of it. (http://rdoproject.org/)

Originally, I set out to deploy my stack with ALL OF THE THINGS. But it turns out, the J1900 and 8GB just isn't enough. I ended up running with just the following services:

* Cinder (using a Synology as the backend)
* Swift
* Glance
* Nova
* Neutron
* Horizon
* LBaaSV2 (Not Octavia, but good enough for home environment)

I had to remove these ones because they were too taxing on resources, although I could probably get Manila running properly

* Sahara
* Trove
* Magnum
* Heat
* Manila
* Ceilometer

## Networking
I have a rack at home where all of this lives, and in there is my Firewall/Gateway, a Fortinet Fortigate. I have seperate VLANs for different components of my home network (Overkill? Probably.). I needed to be able to support VLAN provider networks (or "external" network, were the floating addresses/vips will live), so I can make sure things inside each network that need to talk, don't have to pass through the firewall interfaces and clog it up with useless traffic.

![Rack Lights](/img/rack-lights.jpg)

Tennant networks need to be isolated completely, not even touching the firewall. So my little old Dell PowerConnect switch was set up to have trunks for the tennant VLANs and trunks for the provider VLANs, but also setting a native VLAN for the main nodes IP addresses... yea.. it gets a bit much.

This took some time to figure out, but in the end it was SUPER simple. In the `answers.txt` file that RDO generates, I had to configure the following
```
CONFIG_NEUTRON_ML2_TYPE_DRIVERS=flat,vlan
CONFIG_NEUTRON_ML2_TENANT_NETWORK_TYPES=vlan
CONFIG_NEUTRON_ML2_VLAN_RANGES=physnet1:800:1000,physnet0:100:400
```
Breaking this down, `physnet0` is my "provider" network, the one that has interfaces in my firewall/gateway. VLANs 100-400 are for my home network, but only 4 of them are actually used...

The next one `physnet1` is the tennant network, this will use VLANs 800-1000 when creating tennant networks inside of the stack.

Each MiniPC is configured with bond0 and bond1 interfaces. Bond0 is the management/control interface on each unit. On the controller/network node it has 2 interfaces associated to it, on each compute node only 1 interface is in the bond. Bond1 is the tennant network interface, on the control/network node it has the other 2 interfaces, and on each compute node only 1 interface. Adding single interfaces to the bonds just makes it easier if I decide to upgrade later on. The answer-file also benefits from this.
```
CONFIG_NEUTRON_OVS_BRIDGE_MAPPINGS=physnet1:br-eth1,physnet0:br-ex
CONFIG_NEUTRON_OVS_BRIDGE_IFACES=br-eth1:bond1,br-ex:bond0
CONFIG_NEUTRON_OVS_BRIDGES_COMPUTE=br-eth1
CONFIG_NEUTRON_OVS_EXTERNAL_PHYSNET=physnet0
```

## Deploy it
With all of that done, deploying was as easy as telling the answer-file which nodes are my compute, which is network and which is controller, follow the RDO project guide (https://www.rdoproject.org/install/packstack/) and boom.

Done. I have a small openstack deployment.

![Horizon Hypervisors](/img/horizon-hypervisors.png)

I can create load balancers that can be associated floating IP addresses for any of my core home networks, deploy into private tennancies like in AWS and associate instances to the load balancer pools. A lot nicer than deploying to a BOSH lite.

Thanks for sticking through it.
