---
layout: page
title: Operator Guide
---

This guide describes how to install and operate an XOS-based cloud. It
draws heavily from an existing operational cloud (OpenCloud), but with
the intent of documenting how others might replicate OpenCloud on their
own infrastructure.

##Installing XOS

##Installing OpenStack

This section describes how to bring up OpenCloud's version of an
OpenStack cloud on a cluster.

###Cluster Architecture

Figure 1 shows the goal of the installation process.  At the top is a
controller node running 10 VMs attached to a private management
network; each VM hosts a service needed by OpenStack.  Below are
compute nodes running the nova-compute and neutron-plugin-openvswitch
agents.  The compute nodes connect to a publicly routable network.

![Figure 1. OpenCloud cluster architecture.]({{ site.url }}/figures/controller.jpg)

Figure 1. OpenCloud Cluster Architecture.

One VM on the controller node (denoted "Juju/router") serves as a
router for traffic flowing between the compute nodes and the OpenStack
controller services.  The Juju/router VM has network interfaces on
both the public and the private management network, and is on the same
IP subnet as the compute nodes.  Forwarding rules in the VMs and on
the nodes enable packets to be exchanged between networks.

###Configuring Physical Servers and Network

The controller and compute nodes should meet the following *minimum*
hardware requirements:

* 12 CPU cores, x86_64 architecture
* 48GB RAM
* 3TB disk
* 2x 1Gbps NICs

The nodes should be installed with Ubuntu 14.04 LTS.  Both NICs should
be wired to a public network; NIC1 should have a public IP address and
NIC2 should be left unconfigured.  The compute nodes should not be
behind a firewall.  If the controller node is behind a firewall, the
following TCP ports should be opened for XOS: 22, 3128, 5000, 8080,
8777, 9292, 9696, 35357.

###Using the Install Cloud

The controller node architecture shown in Figure 1 runs each OpenStack
service in its own VM, with all VMs connected by a private management
(virtual) network. To easily bring up this virtual infrastructure we
leverage another OpenStack cloud that runs its controller services in
Amazon EC2. We call the *Install Cloud*, and we use it to bootstrap
individual OpenStack clusters, where the compute nodes in the Install
Cloud are also the head nodes in individual OpenStack clusters.

The following instructions are for OpenCloud admins that have access
to the Install Cloud.

####Networking Setup

To allow the new node to communicate with the Keystone and Neutron
services running in EC2, it is necessary to add a couple security
group rules for the new node.  Log into the EC2 console at Amazon AWS
and add the following rules for the new node's IP address.  Look at
existing rules in each security group for examples.

* In security group *juju-amazon-6*: Add GRE tunnel rule
* In security group, *juju-amazon-3*: Add TCP/35357 rule 

On the node, add the following entries to */etc/hosts*:

```
54.165.170.148  ip-172-31-34-147.ec2.internal juju.amazon
54.165.40.107   ip-172-31-35-79.ec2.internal  keystone.amazon
54.165.71.183   ip-172-31-3-137.ec2.internal  rabbitmq.amazon
54.209.16.51    ip-172-31-4-171.ec2.internal  nova_cc.amazon
54.164.16.21    ip-172-31-26-103.ec2.internal mysql.amazon
54.88.254.182   ip-172-31-16-210.ec2.internal glance.amazon
54.209.88.253   ip-172-31-41-186.ec2.internal neutron.amazon
```

####Add the Server to the Install Cloud as a Compute Node 

Log into the Juju VM (*ubuntu@54.88.138.52*).  Before proceeding, make
sure that you can login to the server from this VM; then run the
following commands:

```
$ juju add-machine ssh:ubuntu@<server IP address>
$ juju add-unit nova-compute --to <juju id of server>
```

After the nova-compute service is running on the node, it is necessary
to install some OpenCloud-specific configuration.  Add information for
the new head node to *~/opencloud-install/ansible/amazon.hosts* and
run:

```
$ cd ~/opencloud-install/ansible
$ ansible-playbook -i inventory/amazon.hosts compute.yml
```

This will likely cause the node to reboot. 

####Create the Management Network and VMs

Log into the Juju VM (*ubuntu@54.88.138.52*).  To configure VMs for
hosting the new cluster's OpenStack controller services, run the
following: 

```
$ cd ~/opencloud-install/ansible
$ ansible-playbook -i inventory/amazon.hosts site-setup.yml
```

After the VMs have been created on the new head node, add information
for the Juju VM to *~/opencloud-install/ansible/inventory/juju.hosts*
and *~/.ssh/config*.  Then run:

```
$ cd ~/opencloud-install/ansible
$ ansible-playbook -i inventory/juju.hosts juju-preinstall.yml
```

(This step requires a bit of cleanup to be actually as simple as
described above.)

###Deploying OpenStack Controller Services

At this point, the VMs and VNs required by the new site should be in
place but they are not running any services.  Next we need to install
the OpenStack controller services in the VMs.

Log into the Juju VM on the new head node.  Create a basic
*/etc/ansible/hosts* file containing a *[service]*  group.  Add the
private IP addresses of all VMs to this group.  An example:

```
[service]
192.168.6.2
192.168.6.[4:11]
```

Next, create a "XXXX-preinstall.yml" playbook, where XXXX is replaced
by the name of the cluster.  Copy one of the existing preinstall
playbooks and change the variables as appropriate for the new cluster.
To install and configure Juju in the VM, run that playbook:

```
$ ansible-playbook foobar-preinstall.yml
$ juju generate-config
```

Now Juju can be used to deploy the controller
services.  First, edit *~/.juju/environments.yaml* using the following
as a template:

```
default: mpisws

environments:
    mpisws:
        type: manual
        bootstrap-host: juju
        bootstrap-user: ubuntu
        default-series: trusty
        apt-http-proxy: http://192.168.6.2:8000
```

Next, add the local VMs to the local Juju environment:

```
$ juju add-machine ssh:192.168.x.y
```

Once all VMs are added to Juju, edit
*~/opencloud-install/juju/manual-install-controller-services.py* so
that it contains the correct Juju IDs and then run it:

```
$ cd ~/opencloud-install/juju
$ ./manual-install-controller-services.py
```

Wait for all services to be installed and started, then run:

```
$ ./manual-install-controller-relations.py
```

To perform the final configuration steps for the site, create
*~/opencloud-install/ansible/<cluster>.yml* using an existing file as
a template.  Run that playbook using *juju-ansible-playbook*:

```
$ cd ~/opencloud-install/ansible
$ juju-ansible-playbook foobar.yml
```

Now the compute nodes can be added to the cluster.

###Deploying OpenStack Compute Nodes

This step assumes a pool of servers in the cluster with Ubuntu 14.04
LTS installed, that are accessible via SSH from the Juju VM.  It is
necessary to do some initial configuration before adding them to
Juju.  Edit */etc/ansible/hosts* so that it contains a *[compute]*
group containing the server names, for example:

```
[compute]
node[62:68].mpisws.vicci.org
```

Next run the preinstall playbook again:

```
$ ansible-playbook foobar-preinstall.yml
```

Now the servers can be added to Juju using *juju add-machine*.  Once
they are all added, edit
*~/opencloud-install/juju/manual-install-nova-compute.py* so that it
contains the Juju ID of one of the servers and run it:

```
$ cd ~/opencloud-install/juju
$ ./manual-install-nova-compute.py
```

Once the nova-compute service is running on that node, it can be added
to the other nodes by their Juju IDs as follows:

```
$ juju add-unit nova-compute --to <Juju ID>
```

Finally, re-run the main cluster playbook created earlier using
*juju-ansible-playbook*: 

```
$ juju-ansible-playbook foobar.yml
```

At this point you can start testing the new OpenStack cluster to make
sure that it's working.

###Configuring Remote OpenStack Clients 

Port forwarding on the Juju/router VM enables remote clients to
connect to the OpenStack services on the cluster.  An OpenStack client
connecting to the VM's public IP address has its request forwarded to
the private IP address of the appropriate VM.  A firewall in the
Juju/router VM ensures that only authorized clients are able to
connect.

When connecting to an OpenStack service, many OpenStack client
libraries fetch its endpoint information from Keystone. The OpenStack
controller services register their private IP addresses on the
management network with Keystone.  If a client is not connected to the
management network, then it may be necessary to translate this private
IP address to the public IP address used for port forwarding.  One way
to do this is with iptables.  For example, if the cluster's management
network is on the 192.168.100.0/24 subnet, and the public IP address
for port forwarding is 1.2.3.4, then one could add the following
iptables rule on the client machine:

```
$ iptables -t nat -A OUTPUT -p tcp -d 192.168.100.0/24 -j DNAT --to-destination 1.2.3.4
```

The OpenCloud install scripts enable SSL for the OpenStack endpoints,
using a certificate generated by Juju.  Get the certificate from
*/etc/ssl/certs/keystone_juju_ca_cert.pem* in the
nova-cloud-controller VM and append it to
*/etc/ssl/certs/ca-certificates.crt* on the client.

##Installing OpenVirteX

##Monitoring

##Operator View

