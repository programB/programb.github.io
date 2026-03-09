+++
date = '2026-03-03T17:01:22+01:00'
modified = '2026-03-03T17:01:22+01:00'
draft = false
title = 'Link aggregation on RouterOS'
tags = ["MikroTik"]
summary = "This post is part of series about Ethernet based link aggregation techniques. In this one I am showing how to configure LACP (Link Aggregation Control Protocol) on MikroTik's RouterOS."
series = ["Link aggregation"]
series_order = 2
+++

Here is a reminder of the topology we want to end up with.
![lab's topology](images/LACP_training_topology.png)
(LAG stands for Link Aggregation Group, ASWx - access switches, DSW - distribution switch, R - router)

> [!NOTE]
> I am simulating all devices in GNS3 using CHR based on RouterOS v7. I have not applied any "default configuration".

## Capabilities and terminology

In MikroTik terminology link aggregation is called "bonding" and all relevant configuration can be accessed through the `/interfaces/bonding` menu in the CLI.

Apart from the LACP, other **modes**, as MikroTik calls them, of creating an aggregated interface are also available. Although I am going to limit myself to LACP alone, here is a quick summary of what else is possible:

- 802.3ad
  - this is the LACP
  - it **IS NOT** the default
- balance-xor
  - similar to but not fully compatible with LACP. It uses layer 3 and 4 information to pick the physical link in the LAG
- balance-rr
  - it load balances the traffic by sending packets in a round-robin fashion through links in the LAG
  - if mode is not specified this **IS THE DEFAULT**
- active-backup
  - uses only one link in the LAG for the traffic the rest is on standby ready to take over if this one fails (redundancy)
- broadcast
  - replicates the traffic on all links in LAG (high noise/disruptions tolerance)
- balance-tlb
  - uses specific link for specific targets and can use links of different speeds (in contrast to LACP)
- balance-alb
  - similar to tlb but can overwrite MAC address in frames to present each target the same MAC regardless of actual path in the LAG

As mentioned in the first post in the series a packet transmitted through LAG is using one (or more) physical paths and that is decided by a combination packet's source and destination addresses. In RouterOS this is configured by specifying **transmit-hash-policy**.
For LACP we can choose:

- layer-2
  - packets sharing common source and destination MAC addresses will be send through the same physical link.
  - **this is the default policy**
- layer-2-and-3 - packets sharing common source and destination MAC addresses and source and destination IP addresses will be send through the same physical link.
  For non-LACP **modes** additional options can be used:
- layer-3-and-4
  - uses IP addresses and ports to assign packets to a physical link (It _may_ work with LACP).
- encap-2-and-3
  - similar to the layer-2-and-3 option but looks at internal IP addresses of tunnelled packets (eg. GRE or PPPoE) not the wrapping IP.
- encap-3-and-4
  - similar to previous but looks at ports as well.

> [!NOTE]
> Some high-end MikroTik devices support hardware offloading for aggregated interfaces but that comes with a restricted choice of modes and transmit policies that can be used - check official documentation.

The neighbors forming an aggregate interface between them will also monitor its state. There are two options:

- mii
  - monitors only the local interface state and is unaware of packets actually reaching their destination
  - this is **the default**
- arp
  - exchanges arp message to actively monitor the connection
  - only **available for some modes (not all)**

## Configuration

To accomplish the task for the switches we basically need to:

- group required interfaces using LACP into a LAG,
- create a bridge interface and add ports and LAGs as its members,
- decide which of those ports should be tagged/untagged and for what VLANSs,
- set up a virtual layer 3 management interface (give it an IP number and tell where its default gateway is).

For the router we need to:

- group required interfaces using LACP into a LAG,
- create virtual interfaces for each VLAN
- add DHCP and DNS servers

Let's start with the access switches.

### Access switches 1 and 2

Since they are almost identical I will treat them jointly. The only difference is the IP address for the management VLAN so be careful when copying.

1. First we create a LAG aka bonding interface comprised of ether1 and ether2.

```
/interface bonding add mode=802.3ad name=LAG_1-2 slaves=ether1,ether2
```

2. Then we create a bridge interface and add member interfaces. ether3 through 5 are going to be access/untagged ports so we need to specify the VLAN tag they will be assigning to the ingress packets.

```
/interface bridge add name=bridge vlan-filtering=no

/interface bridge port
add bridge=bridge interface=ether3 pvid=10
add bridge=bridge interface=ether4 pvid=20
add bridge=bridge interface=ether5 pvid=30
add bridge=bridge interface=LAG_1-2
```

3. In addition to VLANs 10,20 and 30 we need a management VLAN 77. For that purpose we create a virtual layer 3 interface directly on the bridge interface itself (not on the LAG or any other port).

```
/interface vlan add interface=bridge name=MANAGEMENT vlan-id=77
```

4. We want **ether3,4,5** to accept **only untagged** ingress traffic (these are going to be pure access ports) and **LAG** interface to transmit **only tagged** traffic. For security reasons we also want **bridge** interface itself to accept **only tagged** traffic, so the CPU can be reached only on our management VLAN. So let's set up frame admittance policy (the first command affects all ports but next two override it for the LAG and the bridge):

```
/interface bridge port set [ find ] frame-types=admit-only-untagged-and-priority-tagged
/interface bridge port set [find where interface=LAG_1-2] frame-types=admit-only-vlan-tagged
/interface bridge set [find name=bridge] frame-types=admit-only-vlan-tagged
```

5. To complete VLAN configuration we need to create filtering rules in the VLAN table affecting egress traffic. Here we specify that the LAG interface should be tagged/trunk for ALL VLANs, bridge interface (think of it as connection to switch's CPU) should be tagged/trunk for VLAN77 only (that way with admittance policy accept-only-vlan-tagged bridge's CPU is only reachable with tagged frames). Notice that we are not defining ether3,4 or 5 here as untagged interfaces. This is because in RouerOS v7 these entries will be added automatically - do check however this indeed happened.

```
/interface bridge vlan
add bridge=bridge tagged=LAG_1-2 vlan-ids=10
add bridge=bridge tagged=LAG_1-2 vlan-ids=20
add bridge=bridge tagged=LAG_1-2 vlan-ids=30
add bridge=bridge tagged=LAG_1-2,bridge vlan-ids=77
```

6. To access the device from e.g. the router through a SSH session, we need to assign an IP address to our virtual MANAGEMENT interface and provide the default gateway. **WARNING** this address must be different on ASW1 and ASW2 - make changes accordingly.

```
# Ip is 192.168.77.101 for ASW1 and 192.168.77.102 for ASW2
/ip address add address=192.168.77.101/24 interface=MANAGEMENT

/ip route add gateway=192.168.77.1
```

7. We may also like to specify the DNS.

```
/ip dns set servers=192.168.77.1
```

8. Since the switch ignores VLAN tags until vlan-filtering is enabled on the bridge interface, we turn it on in this last step.

```
/interface bridge set bridge vlan-filtering=yes
```

### Distribution switch

After going through the access switches configuration this one is straightforward. The only differences are that there are 3 LAG interfaces and there are no access ports because no hosts connect to this switch directly. Other than that all steps are the same so I will just show entire configuration in one listing:

```
/interface/bonding/add mode=802.3ad name=LAG_1-2 slaves=ether1,ether2
/interface/bonding/add mode=802.3ad name=LAG_3-4 slaves=ether3,ether4
/interface/bonding/add mode=802.3ad name=LAG_5-6-7 slaves=ether5,ether6,ether7


/interface bridge add name=bridge vlan-filtering=no

/interface bridge port
add bridge=bridge interface=LAG_1-2
add bridge=bridge interface=LAG_3-4
add bridge=bridge interface=LAG_5-6-7

/interface vlan add interface=bridge name=MANAGEMENT vlan-id=77

/interface bridge port set [ find ] frame-types=admit-only-vlan-tagged
/interface bridge set [find name=bridge] frame-types=admit-only-vlan-tagged


/interface bridge vlan
add bridge=bridge tagged=LAG_1-2,LAG_3-4,LAG_5-6-7 vlan-ids=10,20,30
add bridge=bridge tagged=LAG_1-2,LAG_3-4,LAG_5-6-7,bridge vlan-ids=77

/ip address add address=192.168.77.103/24 interface=MANAGEMENT
/ip route add gateway=192.168.77.1
/ip dns set servers=192.168.77.1

/interface bridge set bridge vlan-filtering=yes
```

### Router

On the router the essential LACP configuration is even simpler.

1. First step is exactly the same as for the switches. We create a LAG that will handles the connection to the distribution switch:

```
/interface/bonding/add mode=802.3ad name=LAG_2-3-4 slaves=ether2,ether3,ether4
```

2. Since the router is only doing layer 3 operations we don't need any bridge. What we need instead is to create 4 virtual layer 3 interfaces for each of our VLANs. We will add them directly onto the newly created LAG interface like so:

```
/interface vlan
add interface=LAG_2-3-4 name=VLAN10 vlan-id=10
add interface=LAG_2-3-4 name=VLAN20 vlan-id=20
add interface=LAG_2-3-4 name=VLAN30 vlan-id=30
add interface=LAG_2-3-4 name=MANAGEMENT vlan-id=77
```

3. Next we assign IP addresses to those interfaces:

```
/ip address
add address=172.16.10.1/24 interface=VLAN10
add address=172.16.20.1/24 interface=VLAN20
add address=172.16.30.1/24 interface=VLAN30
add address=192.168.77.1/24 interface=MANAGEMENT
```

4. If at this point we decide to configure static IP addresses on hosts connected the access switches that would be all we needed to do. But to fulfil the requirement for this training lab we want to create a DHCP server on the router. Configurations steps are simple. First we create address pools for each VLAN (except management where configuration is static and no DHCP is required):

```
/ip pool
add name=VLAN10_POOL ranges=172.16.10.100-172.16.10.200
add name=VLAN20_POOL ranges=172.16.20.100-172.16.20.200
add name=VLAN30_POOL ranges=172.16.30.100-172.16.30.200
```

5. Then we define what network configuration will be sent to the hosts:

```
/ip dhcp-server network
add address=172.16.10.0/24 dns-server=172.16.10.1 gateway=172.16.10.1
add address=172.16.20.0/24 dns-server=172.16.20.1 gateway=172.16.20.1
add address=172.16.30.0/24 dns-server=172.16.30.1 gateway=172.16.30.1
```

6. Finally we set up servers each bound to the dedicated VLAN interface created previously:

```
/ip dhcp-server
add address-pool=VLAN10_POOL disabled=no interface=VLAN10 name=VLAN10_DHCP
add address-pool=VLAN20_POOL disabled=no interface=VLAN20 name=VLAN20_DHCP
add address-pool=VLAN30_POOL disabled=no interface=VLAN30 name=VLAN30_DHCP
```

7. We may also configure a DNS:

```
/ip dns
set allow-remote-requests=yes servers=8.8.8.8
```

8. As the last step we create a NAT rule for the port that is leading to the outside network and in this lab I chose to additionally set a dhcp-client for this interface so it obtains its configuration automatically (I know that in my lab there is a DHCP server that will provide one).

```
/ip/dhcp-client/add interface=ether1

/ip firewall nat add action=masquerade chain=srcnat out-interface=ether1
```

So with the configuration completed let's see if it actually works.

## Configuration check-up and monitoring

To check if LAG aka bonding interfaces were created we can print the list of all interfaces (here on ASW1):

```
[admin@ASW1] > /interface/print

Flags: R - RUNNING; S - SLAVE
Columns: NAME, TYPE, ACTUAL-MTU, L2MTU, MAC-ADDRESS
 #    NAME        TYPE      ACTUAL-MTU  L2MTU  MAC-ADDRESS
 0 RS ether1      ether           1500         0C:F0:41:3B:00:00
 1 RS ether2      ether           1500         0C:F0:41:3B:00:01
 2 RS ether3      ether           1500         0C:F0:41:3B:00:02
 3 RS ether4      ether           1500         0C:F0:41:3B:00:03
 4 RS ether5      ether           1500         0C:F0:41:3B:00:04
 5    ether6      ether           1500         0C:F0:41:3B:00:05
 6    ether7      ether           1500         0C:F0:41:3B:00:06
 7    ether8      ether           1500         0C:F0:41:3B:00:07
 8 RS LAG_1-2     bond            1500   1500  0C:F0:41:3B:00:00
 9 R  MANAGEMENT  vlan            1496   1496  0C:F0:41:3B:00:02
10 R  bridge      bridge          1500   1500  0C:F0:41:3B:00:02
11 R  lo          loopback       65536         00:00:00:00:00:00
```

We can also check that this new interface is part of the bridge interface:

```
[admin@ASW1] > /interface/bridge/port/print

Columns: INTERFACE, BRIDGE, HW, HORIZON, TRUSTED, FAST-LEAVE, BPDU-GUARD,
         EDGE, POINT-TO-POINT, PVID
# INTERFACE  BRIDGE  HW   HORIZON  TRUSTED  FA  BP  EDGE  POIN  PVID
0 ether3     bridge  yes  none     no       no  no  auto  auto    10
1 ether4     bridge  yes  none     no       no  no  auto  auto    20
2 ether5     bridge  yes  none     no       no  no  auto  auto    30
3 LAG_1-2    bridge  yes  none     no       no  no  auto  auto     1
```

So far so good. Let's see what are the settings of our LAG interfaces (here just one).

```
[admin@ASW1] > /interface/bonding/print

Flags: X - disabled; R - running
 0  R name="LAG_1-2" mtu=1500 mac-address=0C:F0:41:3B:00:00 arp=enabled
      arp-timeout=auto slaves=ether1,ether2 mode=802.3ad primary=none
      link-monitoring=mii arp-interval=100ms arp-ip-targets=""
      mii-interval=100ms down-delay=0ms up-delay=0ms lacp-rate=30secs
      transmit-hash-policy=layer-2 min-links=0 lacp-mode=active
      lacp-system-priority=65535
```

(command `/interface/bonding/print detail` will show the same output)

Among essential options discussed previously are:

- aggregated interfaces are ether1 and ether2
- `mode` - was set explicitly and is in fact 802.3ad aka LACP
- `link-monioring` - defaults to `mii`
- `lacp-rate` - neighbors update their information every 30 seconds
- `transmit-hash-policy` - is by default set to `layer-2` meaning only MAC addresses are used to pick physical interface in the aggregate transmitted packet will actually take.
- `lacp-mode` - is by default set to `active` meaning this switch will actively seek to form LACP connection with the neighbor
  For the meaning of other options see official documentation.

MikroTik also offers live monitoring feature for:

- The LACP interface as a whole
- and for each of the aggregated interfaces.

First the whole LAG:

```
[admin@ASW1] > /interface/bonding/monitor LAG_1-2
                    mode: 802.3ad
            active-ports: ether1
                          ether2
          inactive-ports:
          lacp-system-id: 0C:F0:41:3B:00:00
    lacp-system-priority: 65535
  lacp-partner-system-id: 0C:05:78:BC:00:00
```

We can see here:

- ports that actively participate in the aggregate
- this system id - which is the MAC address assigned to the LAG interface and usually this will be the MAC address of the first slave interface in that LAG (which in this case can be verified like this: `/interface/print where name=ether1`)
- neighbor's id - this will be the MAC address of the LAG interface on the other side of the connection

If we now disconnect one of the "cables" in the connection (let's say ether2) the output will immediately reflect that:

```
                    mode: 802.3ad
            active-ports: ether1
          inactive-ports: ether2
          lacp-system-id: 0C:F0:41:3B:00:00
    lacp-system-priority: 65535
  lacp-partner-system-id: 0C:05:78:BC:00:00
```

The second monitoring feature is of the slave interfaces themselves:

```
[admin@ASW1] > /interface/bonding/monitor-slaves  LAG_1-2

Flags: A - active; P - partner
 AP port=ether1 key=15 flags="A-GSCD--" partner-sys-id=0C:05:78:BC:00:00
     partner-sys-priority=65535 partner-key=15 partner-flags="A-GSCD--"

  P port=ether2 key=0 flags="A-GSCD--" partner-sys-id=0C:05:78:BC:00:00
     partner-sys-priority=65535 partner-key=15 partner-flags="A-GSCD--"
```

As we can see ether2 is not active (only letter "P").

Let's reconnect it:

```
Flags: A - active; P - partner
 AP port=ether1 key=15 flags="A-GSCD--" partner-sys-id=0C:05:78:BC:00:00
     partner-sys-priority=65535 partner-key=15 partner-flags="A-GSCD--"

 AP port=ether2 key=15 flags="A-GSCD--" partner-sys-id=0C:05:78:BC:00:00
     partner-sys-priority=65535 partner-key=15 partner-flags="A-GSCD--"
```

One more point of interest are the virtual layer 3 VLAN interfaces. On ASW1 we have created it on the bridge interface

```
[admin@ASW1] > /interface/vlan/print

Flags: R - RUNNING
Columns: NAME, MTU, ARP, VLAN-ID, INTERFACE
#   NAME         MTU  ARP      VLAN-ID  INTERFACE
0 R MANAGEMENT  1496  enabled       77  bridge
```

But on the **router** they should be on the LAG interface leading to DSW:

```
[admin@R] > /interface/vlan/print

Flags: R - RUNNING
Columns: NAME, MTU, ARP, VLAN-ID, INTERFACE
#   NAME         MTU  ARP      VLAN-ID  INTERFACE
0 R MANAGEMENT  1496  enabled       77  LAG_2-3-4
1 R VLAN10      1496  enabled       10  LAG_2-3-4
2 R VLAN20      1496  enabled       20  LAG_2-3-4
3 R VLAN30      1496  enabled       30  LAG_2-3-4
```

And as we can see this is the case.

Finally we can turn on our client PCs and test whether they can ping each other, so e.g.:

```
NAME        : PC1-VLAN10[1]
IP/MASK     : 172.16.10.200/24
GATEWAY     : 172.16.10.1
DNS         : 172.16.10.1
DHCP SERVER : 172.16.10.1
DHCP LEASE  : 1321, 1800/900/1575
MAC         : 00:50:79:66:68:00
LPORT       : 20094
RHOST:PORT  : 127.0.0.1:20095
MTU         : 1500

> ping 172.16.10.199
84 bytes from 172.16.10.199 icmp_seq=1 ttl=64 time=3.457 ms
^C

> ping 172.16.20.199
84 bytes from 172.16.20.199 icmp_seq=1 ttl=63 time=6.932 ms
^C
```

So it works in the same VLAN and also between VLANs - as it should - and we have thus successfully created and tested 3 aggregated interfaces using LACP.

## Complete listings

Just to make testing easier here are the complete listings for all devices:

```
/system/identity/set name=ASW1

/interface bonding add mode=802.3ad name=LAG_1-2 slaves=ether1,ether2

/interface bridge add name=bridge vlan-filtering=no

/interface bridge port
add bridge=bridge interface=ether3 pvid=10
add bridge=bridge interface=ether4 pvid=20
add bridge=bridge interface=ether5 pvid=30
add bridge=bridge interface=LAG_1-2

/interface vlan add interface=bridge name=MANAGEMENT vlan-id=77

/interface bridge port set [ find ] frame-types=admit-only-untagged-and-priority-tagged
/interface bridge port set [find where interface=LAG_1-2] frame-types=admit-only-vlan-tagged
/interface bridge set [find name=bridge] frame-types=admit-only-vlan-tagged

/interface bridge vlan
add bridge=bridge tagged=LAG_1-2 vlan-ids=10
add bridge=bridge tagged=LAG_1-2 vlan-ids=20
add bridge=bridge tagged=LAG_1-2 vlan-ids=30
add bridge=bridge tagged=LAG_1-2,bridge vlan-ids=77

# Ip is 192.168.77.101 for ASW1 and 192.168.77.102 for ASW2
/ip address add address=192.168.77.101/24 interface=MANAGEMENT

/ip route add gateway=192.168.77.1

/ip dns set servers=192.168.77.1

/interface bridge set bridge vlan-filtering=yes
```

```
/system/identity/set name=ASW2

/interface bonding add mode=802.3ad name=LAG_1-2 slaves=ether1,ether2

/interface bridge add name=bridge vlan-filtering=no

/interface bridge port
add bridge=bridge interface=ether3 pvid=10
add bridge=bridge interface=ether4 pvid=20
add bridge=bridge interface=ether5 pvid=30
add bridge=bridge interface=LAG_1-2

/interface vlan add interface=bridge name=MANAGEMENT vlan-id=77

/interface bridge port set [ find ] frame-types=admit-only-untagged-and-priority-tagged
/interface bridge port set [find where interface=LAG_1-2] frame-types=admit-only-vlan-tagged
/interface bridge set [find name=bridge] frame-types=admit-only-vlan-tagged

/interface bridge vlan
add bridge=bridge tagged=LAG_1-2 vlan-ids=10
add bridge=bridge tagged=LAG_1-2 vlan-ids=20
add bridge=bridge tagged=LAG_1-2 vlan-ids=30
add bridge=bridge tagged=LAG_1-2,bridge vlan-ids=77

# Ip is 192.168.77.101 for ASW1 and 192.168.77.102 for ASW2
/ip address add address=192.168.77.102/24 interface=MANAGEMENT

/ip route add gateway=192.168.77.1

/ip dns set servers=192.168.77.1

/interface bridge set bridge vlan-filtering=yes
```

```
/system/identity/set name=DSW

/interface/bonding/add mode=802.3ad name=LAG_1-2 slaves=ether1,ether2
/interface/bonding/add mode=802.3ad name=LAG_3-4 slaves=ether3,ether4
/interface/bonding/add mode=802.3ad name=LAG_5-6-7 slaves=ether5,ether6,ether7

/interface bridge add name=bridge vlan-filtering=no

/interface bridge port
add bridge=bridge interface=LAG_1-2
add bridge=bridge interface=LAG_3-4
add bridge=bridge interface=LAG_5-6-7

/interface vlan add interface=bridge name=MANAGEMENT vlan-id=77

/interface bridge port set [ find ] frame-types=admit-only-vlan-tagged
/interface bridge set [find name=bridge] frame-types=admit-only-vlan-tagged

/interface bridge vlan
add bridge=bridge tagged=LAG_1-2,LAG_3-4,LAG_5-6-7 vlan-ids=10,20,30
add bridge=bridge tagged=LAG_1-2,LAG_3-4,LAG_5-6-7,bridge vlan-ids=77

/ip address add address=192.168.77.103/24 interface=MANAGEMENT
/ip route add gateway=192.168.77.1
/ip dns set servers=192.168.77.1

/interface bridge set bridge vlan-filtering=yes
```

```
/system/identity/set name=R

/interface/bonding/add mode=802.3ad name=LAG_2-3-4 slaves=ether2,ether3,ether4

/interface vlan
add interface=LAG_2-3-4 name=VLAN10 vlan-id=10
add interface=LAG_2-3-4 name=VLAN20 vlan-id=20
add interface=LAG_2-3-4 name=VLAN30 vlan-id=30
add interface=LAG_2-3-4 name=MANAGEMENT vlan-id=77

/ip address
add address=172.16.10.1/24 interface=VLAN10
add address=172.16.20.1/24 interface=VLAN20
add address=172.16.30.1/24 interface=VLAN30
add address=192.168.77.1/24 interface=MANAGEMENT

/ip pool
add name=VLAN10_POOL ranges=172.16.10.100-172.16.10.200
add name=VLAN20_POOL ranges=172.16.20.100-172.16.20.200
add name=VLAN30_POOL ranges=172.16.30.100-172.16.30.200

/ip dhcp-server network
add address=172.16.10.0/24 dns-server=172.16.10.1 gateway=172.16.10.1
add address=172.16.20.0/24 dns-server=172.16.20.1 gateway=172.16.20.1
add address=172.16.30.0/24 dns-server=172.16.30.1 gateway=172.16.30.1

/ip dhcp-server
add address-pool=VLAN10_POOL disabled=no interface=VLAN10 name=VLAN10_DHCP
add address-pool=VLAN20_POOL disabled=no interface=VLAN20 name=VLAN20_DHCP
add address-pool=VLAN30_POOL disabled=no interface=VLAN30 name=VLAN30_DHCP

/ip dns
set allow-remote-requests=yes servers=8.8.8.8

/ip dhcp-client
add interface=ether1

/ip firewall nat add action=masquerade chain=srcnat out-interface=ether1
```
