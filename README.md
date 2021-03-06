Test arp suppression configurations
========================
This Github repository contains the configuration files necessary for setting 
up basic arp-suppression tests on evpn/vxlan cumulus setup [Reference Topology](http://github.com/cumulusnetworks/cldemo-vagrant).

Quickstart: Run the demo
------------------------
Before running this demo, install [VirtualBox](https://www.virtualbox.org/wiki/Download_Old_Builds) and [Vagrant](https://releases.hashicorp.com/vagrant/). The currently supported versions of VirtualBox and Vagrant can be found on the [cldemo-vagrant](https://github.com/cumulusnetworks/cldemo-vagrant).

    git clone https://github.com/cumulusnetworks/cldemo-vagrant
    cd cldemo-vagrant
    vagrant up oob-mgmt-server oob-mgmt-switch leaf01 leaf02 leaf03 spine01 spine02 
    vagrant ssh oob-mgmt-server
    git clone https://github.com/mgrzybowski/cldemo-arp-suppression-test
    cd cldemo-arp-suppression-test
    ansible-playbook deploy.yml
    ssh leaf01
    ping -I RED 10.0.100.20



Topology
------------------------

         leaf03
        /      \
    spine01  spine02
       |        |
     leaf01   leaf02


    leaf01 and leaf02 are each configured with subinterfaces in vlan100 
    and vlan200 in "RED" VRF and vlan300 in "BLUE" VRF.

    leaf01 (44:38:39:00:00:53):
     - 10.0.100.10/24 vlan100 ( VRF RED )
     - 10.0.100.10/24 vlan300 ( VRF BLUE , the same MAC/IP in diffrent L3 tenant )
     - 10.0.200.10/24 vlan200 ( VRF RED ) 
    leaf02 (44:38:39:00:00:5d):
     - 10.0.100.20/24 vlan100 ( VRF RED )
     - 10.0.100.20/24 vlan300 ( VRF BLUE , the same MAC/IP in diffrent L3 tenant )
     - 10.0.200.20/24 vlan200 ( VRF RED ) 


Pings that should work in this lab 
----------------------------------

VNI 10100 / VLAN 100

      cumulus@leaf01:~$ ping -I RED 10.0.100.20
      ping: Warning: source address might be selected on device other than RED.
      PING 10.0.100.20 (10.0.100.20) from 10.0.100.10 RED: 56(84) bytes of data.
      64 bytes from 10.0.100.20: icmp_seq=1 ttl=64 time=2.11 ms
      64 bytes from 10.0.100.20: icmp_seq=2 ttl=64 time=3.91 ms

VNI 10200 / VLAN 200

      cumulus@leaf01:~$ ping -I RED 10.0.200.20
      ping: Warning: source address might be selected on device other than RED.
      PING 10.0.200.20 (10.0.200.20) from 10.0.200.10 RED: 56(84) bytes of data.
      64 bytes from 10.0.200.20: icmp_seq=1 ttl=64 time=2.31 ms
      64 bytes from 10.0.200.20: icmp_seq=2 ttl=64 time=4.01 ms


VNI 10300 / VLAN 300

     cumulus@leaf01:~$ ping -I BLUE 10.0.100.20
     ping: Warning: source address might be selected on device other than BLUE.
     PING 10.0.100.20 (10.0.100.20) from 10.0.100.10 BLUE: 56(84) bytes of data.
     64 bytes from 10.0.100.20: icmp_seq=1 ttl=64 time=2.35 ms
     64 bytes from 10.0.100.20: icmp_seq=2 ttl=64 time=2.18 ms




EVPN ARP Suppression in Cumulus explained
------------------------------------------

Corrent Cumulus Linux documentation is lacking full explenetion of how evpn arp suppression 
is implemented. Most of bellow is copy-paste from Cumulus Support ticket #8505 and was kindly provided by Trey Aspelund.


In general for ARP suppression to work Type 2 routes need to be exhanged. To originate Type 2 routes,
the local switch will look at its local fdb (bridge fdb show) and neigh (ip neigh show) tables:

    cumulus@spine01:~$ bridge fdb show
    44:38:39:00:00:53 dev swp1 vlan 1 master bridge 
    44:38:39:00:00:53 dev swp1 vlan 200 master bridge 
    44:38:39:00:00:54 dev swp1 master bridge permanent
    44:38:39:00:00:53 dev swp1 vlan 100 master bridge 
    44:38:39:00:00:53 dev swp1 vlan 300 master bridge 
    44:38:39:00:00:5d dev vni100 vlan 100 offload master bridge 
    aa:50:77:a4:1b:28 dev vni100 master bridge permanent
    00:00:00:00:00:00 dev vni100 dst 2.2.2.2 self permanent
    44:38:39:00:00:5d dev vni100 dst 2.2.2.2 self offload 
    44:38:39:00:00:5d dev vni300 vlan 300 offload master bridge 
    f2:2a:aa:6d:35:c2 dev vni300 master bridge permanent
    00:00:00:00:00:00 dev vni300 dst 2.2.2.2 self permanent
    44:38:39:00:00:5d dev vni300 dst 2.2.2.2 self offload 
    44:38:39:00:00:5d dev vni200 vlan 200 offload master bridge 
    5e:9f:52:e8:8a:f0 dev vni200 master bridge permanent
    00:00:00:00:00:00 dev vni200 dst 2.2.2.2 self permanent
    44:38:39:00:00:5d dev vni200 dst 2.2.2.2 self offload 
    44:38:39:00:00:54 dev bridge vlan 100 master bridge permanent
    44:38:39:00:00:54 dev bridge vlan 300 master bridge permanent
    44:38:39:00:00:54 dev bridge vlan 200 master bridge permanent

    cumulus@spine01:~$ ip neighbor  show vrf RED
    10.0.100.20 dev vlan100 lladdr 44:38:39:00:00:5d offload NOARP
    10.0.200.20 dev vlan200 lladdr 44:38:39:00:00:5d offload NOARP
    10.0.100.10 dev vlan100 lladdr 44:38:39:00:00:53 REACHABLE
    10.0.200.10 dev vlan200 lladdr 44:38:39:00:00:53 REACHABLE
    fe80::4638:39ff:fe00:5d dev vlan100 lladdr 44:38:39:00:00:5d router offload NOARP
    fe80::4638:39ff:fe00:53 dev vlan200 lladdr 44:38:39:00:00:53 router REACHABLE
    fe80::4638:39ff:fe00:5d dev vlan200 lladdr 44:38:39:00:00:5d router offload NOARP
    fe80::4638:39ff:fe00:53 dev vlan100 lladdr 44:38:39:00:00:53 router REACHABLE

    cumulus@spine01:~$ ip neighbor  show vrf BLUE
    10.0.100.20 dev vlan300 lladdr 44:38:39:00:00:5d offload NOARP
    10.0.100.10 dev vlan300 lladdr 44:38:39:00:00:53 REACHABLE
    fe80::4638:39ff:fe00:53 dev vlan300 lladdr 44:38:39:00:00:53 router REACHABLE
    fe80::4638:39ff:fe00:5d dev vlan300 lladdr 44:38:39:00:00:5d router offload NOARP


Local fdb table is build by the bridge , no special steps are needed. 
But for ip neighbor table to appear when You do not have IP on  VLAN/SVI inteface 
You need to  enable "bridge-arp-nd-suppress on" on any of the  VXLAN/VNI interface. 
Ifupdaown2 do so some magic then, and sets some flags on the interface

    info: bridge: vni100: set bridge-arp-nd-suppress on
    debug: (cache 0)
    info: vni100: netlink: ip link set dev vni100: bridge slave attributes
    debug: vni100: ifla_info_slave_data {152: True}

Then daemon called neighmgrd comes to play:

      985 ?        Ssl    0:00 /usr/bin/python /usr/bin/neighmgrd

whose sole purpose in life is to keep neighbor entries REACHABLE.
Enablinhg "bridge-arp-nd-suppress on" on interface will make neighmgrd 
to  generate ARP Probes and Neighbor Solicitations in order to keep the neighbors reachable for the kernel.

     21:57:46.621715 44:38:39:00:00:54 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.0.100.10 tell 0.0.0.0, length 28
     21:57:46.621754 44:38:39:00:00:53 > 44:38:39:00:00:54, ethertype ARP (0x0806), length 42: Reply 10.0.100.10 is-at 44:38:39:00:00:53, length 28

It seems that neighmgrd will base its refresh rate on the base_reachable_time_ms sysctl knob.
I haven't played around with gc_stale timers, but given the ARPs are generated by the kernel, 
I don't see why you wouldn't be able to modify the behavior using it.




For the VNI's it is configured to advertise for, it will use the entries there and create EVPN NLRIs 
that it will advertise to its neighbors that are enabled for the L2VPN EVPN AFI/SAFI. 
You can see these Type 2 routes with net show bgp l2vpn evpn route or sudo vtysh -> show bgp l2vpn evpn {route}:

    spine01# show bgp l2 ev 
    Route Distinguisher: ip 1.1.1.1:2
   
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:53]
                        1.1.1.1                            32768 i
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:53]:[32]:[10.0.100.10]
                        1.1.1.1                            32768 i
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:53]:[128]:[fe80::4638:39ff:fe00:53]
                        1.1.1.1                            32768 i
    *> [3]:[0]:[32]:[1.1.1.1]
                        1.1.1.1                            32768 i
    Route Distinguisher: ip 1.1.1.1:3
   
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:53]
                        1.1.1.1                            32768 i
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:53]:[32]:[10.0.100.10]
                        1.1.1.1                            32768 i
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:53]:[128]:[fe80::4638:39ff:fe00:53]
                        1.1.1.1                            32768 i
    *> [3]:[0]:[32]:[1.1.1.1]
                        1.1.1.1                            32768 i
    Route Distinguisher: ip 1.1.1.1:4
   
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:53]
                        1.1.1.1                            32768 i
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:53]:[32]:[10.0.200.10]
                        1.1.1.1                            32768 i
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:53]:[128]:[fe80::4638:39ff:fe00:53]
                        1.1.1.1                            32768 i
    *> [3]:[0]:[32]:[1.1.1.1]
                        1.1.1.1                            32768 i
    Route Distinguisher: ip 2.2.2.2:2
   
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:5d]
                        2.2.2.2                                0 65020 65012 i
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:5d]:[32]:[10.0.100.20]
                        2.2.2.2                                0 65020 65012 i
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:5d]:[128]:[fe80::4638:39ff:fe00:5d]
                        2.2.2.2                                0 65020 65012 i
    *> [3]:[0]:[32]:[2.2.2.2]
                        2.2.2.2                                0 65020 65012 i
    Route Distinguisher: ip 2.2.2.2:3
   
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:5d]
                        2.2.2.2                                0 65020 65012 i
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:5d]:[32]:[10.0.100.20]
                        2.2.2.2                                0 65020 65012 i
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:5d]:[128]:[fe80::4638:39ff:fe00:5d]
                        2.2.2.2                                0 65020 65012 i
    *> [3]:[0]:[32]:[2.2.2.2]
                        2.2.2.2                                0 65020 65012 i
    Route Distinguisher: ip 2.2.2.2:4
   
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:5d]
                        2.2.2.2                                0 65020 65012 i
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:5d]:[32]:[10.0.200.20]
                        2.2.2.2                                0 65020 65012 i
    *> [2]:[0]:[0]:[48]:[44:38:39:00:00:5d]:[128]:[fe80::4638:39ff:fe00:5d]
                        2.2.2.2                                0 65020 65012 i
    *> [3]:[0]:[32]:[2.2.2.2]
                        2.2.2.2                                0 65020 65012 i
   
    Displayed 24 out of 24 total prefixes


As you can see, the MACs and ARP/ND entries are being injected into the L2VPN EVPN BGP table as per the contents of the fdb and neigh tables.

In FRR, zebra acts as the RIB, the linux kernel acts as the FIB, and bgpd is the service operating BGP for any AFI/SAFIs that exist. 
Regarding the rules for how the routes get into frr and zebra, that follows typical BGP operations (eBGP/iBGP loop prevention, AS-Path, etc.).
Once an NLRI is accepted into the local BGP table, the best-path algorithm is run, and the best paths are imported into zebra. 
Zebra will decide which routes get sent to the linux kernel. 
Typically Zebra doesn't need to do much more than just pass the best path along to the kernel, 
unless there are multiple paths from different protocols, in which case Administrative Distance comes into play.


Regarding the "EVPN ARP cache", that is just a visual wrapper for the NLRI that exist in the BGP table for the EVPN AFI/SAFI. 
I believe it looks at the BGP table for each VNI, and selects any MAC/IP Type 2 Routes for ease of visibility:

    spine01# show evpn arp-cache vni all 
    
    VNI 10200 #ARP (IPv4 and IPv6, local and remote) 5
    
    IP                      Type   State    MAC               Remote VTEP          
    fe80::4638:39ff:fe00:5d remote active   44:38:39:00:00:5d 2.2.2.2              
    10.0.200.20             remote active   44:38:39:00:00:5d 2.2.2.2              
    fe80::4638:39ff:fe00:54 local  active   44:38:39:00:00:54
    fe80::4638:39ff:fe00:53 local  active   44:38:39:00:00:53
    10.0.200.10             local  active   44:38:39:00:00:53
    
    VNI 10100 #ARP (IPv4 and IPv6, local and remote) 5
    
    IP                      Type   State    MAC               Remote VTEP          
    fe80::4638:39ff:fe00:5d remote active   44:38:39:00:00:5d 2.2.2.2              
    10.0.100.10             local  active   44:38:39:00:00:53
    10.0.100.20             remote active   44:38:39:00:00:5d 2.2.2.2              
    fe80::4638:39ff:fe00:54 local  active   44:38:39:00:00:54
    fe80::4638:39ff:fe00:53 local  active   44:38:39:00:00:53
    
    VNI 10300 #ARP (IPv4 and IPv6, local and remote) 5
    
    IP                      Type   State    MAC               Remote VTEP          
    fe80::4638:39ff:fe00:5d remote active   44:38:39:00:00:5d 2.2.2.2              
    10.0.100.10             local  active   44:38:39:00:00:53
    10.0.100.20             remote active   44:38:39:00:00:5d 2.2.2.2              
    fe80::4638:39ff:fe00:54 local  active   44:38:39:00:00:54
    fe80::4638:39ff:fe00:53 local  active   44:38:39:00:00:53

If you have the Type 2 Routes you're expecting in the EVPN BGP table, and they have successfully added to the kernel's neighbor table with the "offload" tag, then you know the neighbor entry is working correctly.

In the case of ARP/ND Suppression, the local switch will perform Proxy-ARP/Proxy-NA on behalf of the remote host. You will see the tag "NOARP" in the neighbor table for each neighbor the local switch will be doing this for:

    root@spine01:/home/cumulus# ip neigh show | grep NOARP
    10.0.100.20 dev vlan300 lladdr 44:38:39:00:00:5d offload NOARP
    10.0.100.20 dev vlan100 lladdr 44:38:39:00:00:5d offload NOARP
    10.0.200.20 dev vlan200 lladdr 44:38:39:00:00:5d offload NOARP
    fe80::4638:39ff:fe00:5d dev vlan100 lladdr 44:38:39:00:00:5d router offload NOARP
    fe80::4638:39ff:fe00:5d dev vlan300 lladdr 44:38:39:00:00:5d router offload NOARP
    fe80::4638:39ff:fe00:5d dev vlan200 lladdr 44:38:39:00:00:5d router offload NOARP


EVPN - Disable IPv6 NLRIs
------------------------------------------
In most cases IPv6 NLRIs are not needed. Additionaly Ipv6 link local address is additional vector of switch attack in LAN. Trey Aspelund from cumulus suport had severa ideas howto disable ipv6:

The options that I considered were the following:

- Control Plane filtering
  > Likely route-maps, if frr has the right match parameters and it works in the L2VPN EVPN AFI/SAFI.
  > I don't think this has ever been tested, so results may vary.
  > Even if IPv6 MAC/IP routes were prevented from propagating via EVPN, I think v6 traffic would still be able to traverse VXLAN via the MAC routes.

- Kernel settings (sysctl)
  > This is kind of a messy solution, as sysctl can prove interesting to manipulate correctly
  > I did some testing with this in VX and got interesting results. I set net.ipv6.conf.[interface].disable_ipv6 to '1' and net.ipv6.conf.[interface].accept_ra to '0'. I ended up with no IPv6 addresses on the SVI's, yet neighbor entries still show up via the vlan interfaces. This resulted in BGP creating Type 2 MAC/IPv6 routes anyway.

- Inbound "deny" ACL matching IPv6 traffic
  > This should prevent any v6 traffic from hitting the SVI's to begin with, so no neighbor entries should be learned
  > If there are no neighbor entries, then there should be no MAC/IPv6 routes in EVPN.
  > This should also prevent data plane v6 traffic from traversing VXLAN as well.


In the end the simples solution seems to be disabling ipv6 on vlan interface. For example:

    auto vlan100
    iface vlan100
        ip6-forward off
        ip-forward off
        vlan-id 100
        vlan-raw-device bridge
        vrf RED
        ## disable ipv6 on interface and ipv6 EVPN  NLRIs
        up echo 1 > /proc/sys/net/ipv6/conf/$IFACE/disable_ipv6



Linux box bridge connected to cumulus - multiple arp-reply to ARP "Probe",
------------------------------------------

http://tools.ietf.org/html/rfc5227:

    The 'target IP address' field MUST be set to the address
    being probed. An ARP Request constructed this way, with an all-zero
    'sender IP address', is referred to as an 'ARP Probe'.


In case when linux bridge for ARP probe kernel could not find related interface (sorce ip is 0.0.0.0) and sends multiple arp replay, one for each  interface MAC that it is in bridge.
For example one :

    13:10:30.024064 00:25:90:b2:ec:a8 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.196.13.101 tell 0.0.0.0, length 46
    13:10:30.024112 0c:c4:7a:88:58:2e > 00:25:90:b2:ec:a8, ethertype ARP (0x0806), length 42: Reply 10.196.13.101 is-at 0c:c4:7a:88:58:2e, length 28
    13:10:30.024155 c6:bc:31:00:ea:bf > 00:25:90:b2:ec:a8, ethertype ARP (0x0806), length 42: Reply 10.196.13.101 is-at c6:bc:31:00:ea:bf, length 28
    13:10:30.024157 2e:b7:f7:89:77:a2 > 00:25:90:b2:ec:a8, ethertype ARP (0x0806), length 42: Reply 10.196.13.101 is-at 2e:b7:f7:89:77:a2, length 28
    13:10:30.024158 ea:d7:f3:39:58:0e > 00:25:90:b2:ec:a8, ethertype ARP (0x0806), length 42: Reply 10.196.13.101 is-at ea:d7:f3:39:58:0e, length 28

Multiple arp replays will cause cumulus box to learn last of them in neigh table and send NLRI containing that pair of ip-mac.
Possibe solution is to restrict arp_announce/arp_ignore on linux box:

    net.ipv4.conf.default.arp_announce = 2
    net.ipv4.conf.default.arp_ignore = 1


GARP Flood - is suppresed !
------------------------------------------
In cumulus tests in the LAB both the network and the attached hosts were updated upon an extended mobility scenario (mac or ip change upon "move") 
as long as a single GARP was sent by the host doing the moving. 
In all occasions, a BGP Update message was sent with the Extended MAC Mobility extended community incremented,
which resulted in a GARP sent by the remote VTEP.
In production enviroment single GARP packet is not enought:

- GARP stream need to be long ( 5 - 10 second ) to make sure that every device in vlan updated their ARP table ( for example ten packets every second ) . 
- VIP Could fall back to previous device , it could hours, or microseconds , so this IP-MAC pair for GARP not necessary is "new" . 
- In case of any off unexpected condition (like network segmentation) You should be able to send GARP stream to fix ARP table to correct values on all of the servers in vlan.


Acoording to https://tools.ietf.org/html/draft-ietf-bess-evpn-proxy-arp-nd-05 cumulus created Feature Request tracking as FR-1523 
to allow GARP to be flooded :

    4.5. Flooding (to Remote PEs) Reduction/Suppression

       The Proxy-ARP/ND function implicitly helps reducing the flooding of
       ARP Request and NS messages to remote PEs in an EVPN network.
       However, in certain use-cases, the flooding of ARP/NS/NA messages
       (and even the unknown unicast flooding) to remote PEs can be
       suppressed completely in an EVPN network.

       For instance, in an IXP network, since all the participant CEs are
       well known and will not move to a different PE, the IP->MAC entries
       may be all provisioned by a management system. Assuming the entries
       for the CEs are all provisioned on the local PE, a given Proxy-ARP/ND
       table will only contain static and EVPN-learned entries. In this
       case, the operator may choose to suppress the flooding of ARP/NS/NA
       to remote PEs completely.

       The flooding may also be suppressed completely in IXP networks with
       dynamic Proxy-ARP/ND entries assuming that all the CEs are directly
       connected to the PEs and they all advertise their presence with a
       GARP/unsolicited-NA when they connect to the network.

       In networks where fast mobility is expected (DC use-case), it is not
       recommended to suppress the flooding of unknown ARP-Requests/NS or
       GARPs/unsolicited-NAs. Unknown ARP-Requests/NS refer to those
       ARP-Request/NS messages for which the Proxy-ARP/ND lookups for the
       requested IPs do not succeed.

       In order to give the operator the choice to suppress/allow the
       flooding to remote PEs, a PE MAY support administrative options to
       individually suppress/allow the flooding of:

       o Unknown ARP-Request and NS messages.
       o GARP and unsolicited-NA messages.​

       The operator will use these options based on the expected behavior in
       the CEs.





