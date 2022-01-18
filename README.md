![Passive OpenShift Disaster Recovery](images/title.png?raw=true "Title")

# Passive OpenShift Disaster Recovery

We consultants often have to adapt to the customer's infrastructural situation and find the simplest and most maintainable way to face problems that are not always easy to solve. 

Fortunately, OpenShift 4 has greatly simplified the installation procedure while maintaining high levels of customization.

This blog will analyze a real customer's requests about a passive disaster recovery procedure and how they have been addressed: let’s get started!

## Scenario

The customer is a primary bank in Italy that follows BCE’s best practices for passive disaster recovery based on hypervisor high availability and storage low-level replication. There are two datacenters: one for production and one for the disaster recovery purpose.

For OpenShift disaster recovery this scenario is unsupported because it can lead to corruption of the OCP Etcd database when DR has been activated with standard procedures based on VM replication (e.g. VMware Site Recovery Manager). 

Futher, in case of DR, all primary services (VMs) and OCP clusters must go live in the secondary datacenter using same ipv4 address, gateway, DNS and routing table they had in the primary datacenter.

This is a significant constraint that needs to be addressed. 

For example, it is impossible to run the same AD server or an Image Repository or the same OpenShift cluster on both data centers simultaneously (the OCP API server IP address can not be changed). Therefore the only possible solution is to start all the virtual machines on the secondary data center only when the primary datacenter is completely isolated or unreachable (from a networking point of view).

The following images describe a high-level overview of the scenario:


![Normal operativity](images/1.jpg?raw=true "Title")

*Normal operativity - all the services running in the Primary Datacenter on VMware Cluster 1*

---


![Normal operativity](images/2.jpg?raw=true "Title")

*Disaster recovery in progress - all the VMS running on the secondary datacenter will use the  same ipv4 address, gateway, DNS and routing table of the primary datacenter*

---

![Normal operativity](images/3.jpg?raw=true "Title")

*A new OpenShift cluster is created in the secondary datacenter. The connectivity change between the Primary and Secondary Datacenter is a manual operation, but it can be fully automated.*

---


Speaking of OpenShift, we will take advantage of the powerful "[Installer Provisioned Infrastructure](https://docs.openshift.com/container-platform/4.9/installing/installing_vsphere/installing-vsphere-installer-provisioned.html)" (IPI) method that will allow us, once everything is configured, to create our cluster with a single command. We will use ArgoCD to complete all the configurations when the cluster is operational.









