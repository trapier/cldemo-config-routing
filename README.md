Demo Routing Configurations
===========================
This demo and these configurations are written to be used with the [cldemo-vagrant](https://github.com/cumulusnetworks/cldemo-vagrant) reference topology.

The configuration files in this repository will be placed on the appropriate devices in order to set up the desired Routing Protocol between the leafs and spines, and will configure a Layer 2 bridge on each leaf top-of-rack switch for the servers in that rack. 

This Github repository contains the configuration files necessary for setting up Layer 3 routing on a [CLOS topology](http://www.networkworld.com/article/2226122/cisco-subnet/clos-networks--what-s-old-is-new-again.html) using Cumulus Linux and Quagga. 

A helper script named `push-config.py` will quickly deploy the flat configuration files to the devices in the network, but you could just as easily copy and paste them by hand or incorporate them into an automation tool like Ansible instead.

Topology
--------
This demo runs on a spine-leaf topology with two single-attached hosts. The helper script `push-config.py` requires an out-of-band management network that provides access to eth0 on all of the in-band devices.

         +------------+       +------------+
         | spine01    |       | spine02    |
         |            |       |            |
         +------------+       +------------+
         swp1 |    swp2 \   / swp1    | swp2
              |           X           |
        swp51 |   swp52 /   \ swp51   | swp52
         +------------+       +------------+
         | leaf01     |       | leaf02     |
         |            |       |            |
         +------------+       +------------+
         swp1 |                       | swp2
              |                       |
         eth1 |                       | eth2
         +------------+       +------------+
         | server01   |       | server02   |
         |            |       |            |
         +------------+       +------------+



Deploying An Example Configuration
------------------------

Eight different configuration examples are included:

 * OSPF Numbered
 * OSPF Unnumbered
 * BGP Numbered
 * BGP Unnumbered
 * OSPF Numbered with IPv6
 * OSPF Unnumbered with IPv6
 * BGP Numbered with IPv6
 * BGP Unnumbered with IPv6

### 1). Install Prerequisites

Go to the [prequisites list](https://github.com/CumulusNetworks/cldemo-vagrant#prerequisites) associated with the reference topology to download an install the required software.

### 2). Clone or Download the Reference Topology

To Clone the software (git must be installed already on your machine):

    git clone https://github.com/cumulusnetworks/cldemo-vagrant
    
If git is not installed on your machine and you'd rather [download the code directly](https://github.com/CumulusNetworks/cldemo-vagrant/archive/master.zip) follow the link below and unzip the file once it has been downloaded.

### 3). Start the VMs
After obtaining the software, move in to the directory containing the software.

    cd cldemo-vagrant
 
Use the "vagrant up" command as shown below to start the different VMs that will be used in this demo.

    vagrant up oob-mgmt-server oob-mgmt-switch leaf01 leaf02 spine01 spine02 server01 server02
    
### 4). Login to the Management Server 
Use SSH to login to the out-of-band management server.
    
    vagrant ssh oob-mgmt-server
    
Login to the "cumulus" user
   
    sudo su - cumulus
    
### 5). Download the Routing Configurations

    git clone https://github.com/cumulusnetworks/cldemo-config-routing
    cd cldemo-config-routing
    sudo ln -s  /home/cumulus/cldemo-config-routing /var/www/cldemo-config-routing
    
### 6). Push Configuration Files to Devices
After setting up the repo, you can now use `push-config.py` This script will log in to each device, download the configuration files, and reboot the device. [Click to learn more about the operation of the script](https://github.com/CumulusNetworks/cldemo-config-routing#using-the-helper-script)

Note the keyword "bgp-unnumbered" in the command below; this can be replaced with whatever example configuration you would like to deploy (such as "ospf-unnumbered" or "ospf-numbered" ).

    python pushconfig.py bgp-unnumbered leaf01,leaf02,spine01,spine02,server01,server02
    
### 7). Experiment
Login to server01 and ping server02.

    ssh server01
    ping 172.16.2.101


Using the Helper Script
-----------------------
The `push-config.py` helper script deploys the configuration to the in-band network by downloading the files from the out-of-band management server. This requires a web server to be installed on the out-of-band server and passwordless login and sudo to be enabled on the in-band devices, both of which are done for you if you used [cldemo-vagrant](http://github.com/cumulusnetworks/cldemo-vagrant) to provision your topology. The demo repository needs to be linked in the management server's `/var/www/` directory:

    vagrant ssh oob-mgmt-server
    sudo su - cumulus
    git clone https://github.com/cumulusnetworks/cldemo-config-routing
    cd cldemo-config-routing
    sudo ln -s  /home/cumulus/cldemo-config-routing /var/www/cldemo-config-routing

After setting up the repo, you can now use `push-config.py`! This script will log in to each device, download the files, and reboot the device.

    python pushconfig.py <demo_name> leaf01,leaf02,spine01,spine02,server01,server02


Verifying Routing
-----------------
Running the demo is easiest with two terminal windows open. One window will log into server01 and ping server02's IP address. The second window will be used to deploy new configuration on the switches.

*In terminal 1*

    vagrant ssh oob-mgmt-server
    sudo su - cumulushttps://github.com/CumulusNetworks/cldemo-config-routing#using-the-helper-script
    cd cldemo-config-routing
    python pushconfig.py bgp-unnumbered leaf01,leaf02,spine01,spine02,server01,server02

*In terminal 2*

    vagrant ssh oob-mgmt-server
    sudo su - cumulus
    ssh server01
    ping 172.16.2.101

*In terminal 1*

    python pushconfig.py ospf-unnumbered leaf01,leaf02,spine01,spine02
    # wait and watch connectivity drop and then come back
    python pushconfig.py bgp-numbered leaf01,leaf02,spine01,spine02
    # again
    python pushconfig.py bgp-unnumbered-ipv6 leaf01,leaf02,spine01,spine02,server01,server02
    # this will reboot server01, so you'll need to log back in in terminal 2


Verify High Availability
------------------------
Using a routing protocol such as BGP or OSPF means that as long as one spine is still running, the network will automatically learn a new route and keep the fabric connected. This means that you can do rolling upgrades one spine at a time without incurring any downtime.

*In terminal 1*

    vagrant ssh oob-mgmt-server
    sudo su - cumulus
    cd cldemo-config-routing
    python pushconfig.py bgp-unnumbered leaf01,leaf02,spine01,spine02,server01,server02

*In terminal 2*

    vagrant ssh oob-mgmt-server
    sudo su - cumulus
    ssh server01
    ping 172.16.2.101
    
*In terminal 3*

    vagrant destroy -f spine01
    # note that the pings may hiccup a bit, but will keep going
    vagrant destroy -f spine02
    # now pings will totally fail
    vagrant up spine01 spine02

*In terminal 1*

    python pushconfig.py bgp-unnumbered spine01,spine02
    # watch Terminal 2, and pings will return


Using Quagga Dry Runs for Syntax Checking
-----------------------------------------
You can use Quagga's dry run functionality to check the syntax of Quagga configuration without applying the changes.

    vtysh -f Quagga.conf --dryrun

The syntax of all of the Quagga.conf files in this repository can be verified using the following line in bash.

     for i in `find  | grep Quagga.conf`; do vtysh -f $i --dryrun; done ;
