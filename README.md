Test arp suppression configurations
========================
This Github repository contains the configuration files necessary for setting
up Multi-Chassis Link Aggregation using Cumulus Linux and FRRouting on the [Reference Topology](http://github.com/cumulusnetworks/cldemo-vagrant).

The flatfiles in this repository will set up a BGP unnumbered routing fabric
between the leafs and spines, and will configure MLAG between the two
top-of-rack switches and the two servers in that rack. Ansible is used to
quickly deploy configuration from the OOB server to devices in the network.

Quickstart: Run the demo
------------------------
Before running this demo, install [VirtualBox](https://www.virtualbox.org/wiki/Download_Old_Builds) and [Vagrant](https://releases.hashicorp.com/vagrant/). The currently supported versions of VirtualBox and Vagrant can be found on the [cldemo-vagrant](https://github.com/cumulusnetworks/cldemo-vagrant).

    git clone https://github.com/cumulusnetworks/cldemo-vagrant
    cd cldemo-vagrant
    vagrant up oob-mgmt-server oob-mgmt-switch leaf01 leaf02 leaf03 spine01 spine02 
    vagrant ssh oob-mgmt-server
    git clone https://github.com/mgrzybowski/cldemo-arp-suppression-test
    cd arp-test
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


    leaf01 and leaf02 are each configured with subinterfaces in vlan100 and vlan200.
    leaf01:
     - 10.0.100.10/24 vlan100
     - 10.0.200.10/24 vlan200
    leaf02:
     - 10.0.100.20/24 vlan100
     - 10.0.200.20/24 vlan200

