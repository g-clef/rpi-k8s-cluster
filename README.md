# rpi-k8s-cluster

This is a set of ansible playbooks to setup a k8s cluster on a cluster of 20 raspberry pi's.
The rasberrypis are all netbooted, to avoid the need for an sd card. 
All storage for the cluster comes from a Synology NAS, with drives
mounted on the raspberry pi's. The TFTP server and DHCP server are run from a VM
on the NAS. That VM is also the ansible controller node. 