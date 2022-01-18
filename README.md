![OpenShift Disaster Recovery](images/title.png?raw=true "Title")

# Cold start OpenShift Disaster Recovery

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

*A new OpenShift cluster is created in the secondary datacenter. The connectivity change between the primary and secondary Datacenter is a manual operation, but it can be fully automated.*

---


Speaking of OpenShift, we will take advantage of the powerful "[Installer Provisioned Infrastructure](https://docs.openshift.com/container-platform/4.9/installing/installing_vsphere/installing-vsphere-installer-provisioned.html)" (IPI) method that will allow us, once everything is configured, to create our cluster with a single command. We will use ArgoCD to complete all the configurations when the cluster is operational.

Here is the summary of the customer's requests:


1. Automatic Installation of an OCP cluster using the IPI method on a VMware cluster
2. Automatic installation of Operators from different catalog sources
Configuration of the operators’ custom resources
3. Synchronization of LDAP users and groups
4. Automatic Installation of an arbitrary number of applications using the GitOps approach
5. SLA (service level agreement) <= 2 hours
6. “Easy to run” procedure

# Strategy


In the “[Disaster Recovery Strategies for Applications Running on OpenShift](https://www.openshift.com/blog/disaster-recovery-strategies-for-applications-running-on-openshift)” blog post written by **Raffaele Spazzoli**, disaster recovery strategies are grouped into two main categories :

1. **Active/Passive:** with this approach, under normal circumstances, all the traffic goes to one datacenter. The second datacenter is in a standby mode in case of a disaster. If a disaster occurs, it is assumed that there will be downtime in order to perform tasks needed to recover the service in the other datacenter.
 
2. **Active/Active**: with this approach, load is spread across all available datacenters. If a datacenter is lost due to a disaster, there can be an expectation that there is no impact to the services.


It is not possible for us to use the Active/Active or Active/Passive approach due to the before mentioned “same IP address” constraint.

In our strategy (also known as “cold start procedure”), after having isolated the primary datacenter at the network level (to avoid the IP addressing problems) the OCP cluster will be created and configured from **scratch** in the secondary datacenter.

OpenShift will be installed in [disconnected mode](https://docs.openshift.com/container-platform/4.9/installing/installing-mirroring-installation-images.html) (More on this later)

The cold start procedure for our OCP cluster is all about creating e configuring services and OCP resources. There is some pre-work to do, here is a summarized list:

1. Create a bastion host for mirroring the OCP installation image to a private image registry already present in the infrastructure.
2. Find the operator catalogs that include our operators, "prune it,” (remove all the unnecessary images), and mirror only the required operators images.
3. Configure a Git repository for our ArgoCD applications
4. Prepare all the YAML files that describe the resources we need like:
   1.  Identity providers
   2. OperatorGroup
   3. Operator Subscriptions
   4. Main ArgoCD application
   5. MachineSets for Infra nodes


These steps are **preparatory** and must be performed on the primary datacenter.
Once all the virtual machines (bastion, private image registry, etc ..) have been prepared, they will be **cloned** to the secondary datacenter in a **power-off** state (again for avoiding the IP address problem) using the underlying VMware infrastructure.


The following graph summarizes the logical flow overview of tasks that culminates with the IPI installation process. This also demonstrates how the OpenShift cluster installation can be enriched with preliminary and post-install customizations that can be fully automated.



![Graph](images/4.png?raw=true "Title")

*High level procedure schema*

---


