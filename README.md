Test arp suppression configurations
========================
This Github repository contains the configuration files necessary for setting 
up basi arp-suppression tests on evpn/vxlan cumulus setup [Reference Topology](http://github.com/cumulusnetworks/cldemo-vagrant).

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
    ping 10.0.100.20



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
Ifupdaown2 do so some magic then, and Cumulus patched kernels starts to learn neighbors:

    info: bridge: vni100: set bridge-arp-nd-suppress on
    debug: (cache 0)
    info: vni100: netlink: ip link set dev vni100: bridge slave attributes
    debug: vni100: ifla_info_slave_data {152: True}

Additionaly  to that CL runs a daemon called neighmgrd

      985 ?        Ssl    0:00 /usr/bin/python /usr/bin/neighmgrd

whose sole purpose in life is to keep neighbor entries REACHABLE.
Enablinhg ""bridge-arp-nd-suppress on" on interface will make neighmgrd 
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

