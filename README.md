## ovirt-deploy
The end goal of this set of templates is to speed up Red Hat Virtualization (RHV) deployment through the use of oVirt Ansible roles and additional tools that can be required to complete the installation process. 

Currently the playbooks may have more than needed hard-coded settings which will be eventually changed to variables. 

In release this release (0.98, just because I don't want to call it 1.0) RHV is expecting to find a Red Hat Gluster Storage (RHGS) cluster, which can also be deployed through available playbooks.

These playbooks will be eventually updated to support oVirt and glusterfs-server from CentOS storage SIG.

## How to use it
After cloning the repository to a Linux system which has Ansible 2.6 (untested with other releases), you may:

1. Optional: Deploy libvirt based VMs where to deploy RHV and RHGS, for demo or learning purposes
2. Deploy RHGS
3. Deploy RHV
4. Optional: Deploy a second environment to be used for disaster recovery

### Initial configuration
The following variables from *config-vars.yml* must be configured according to your system.

| Variable | Description  |
|--|--|
| repository | should be "yumrepo" for a local yum repository or "customer_portal" if system should be registered in Red Hat Customer Portal |
| baseurl | base path to yum repositories - required when repository is set to "yumrepo" |
| activation_key | activation key configured in Red Hat Customer Portal including subscriptions for all the nodes  - required when repository is set to "customer_portal" |
| rhsm_org_id | ID of the user organization or Red Hat Customer Portal - required when repository is set to "customer_portal" |

**TODO**: add support to use customer portal login and password instead of activation keys.

Create an ansible inventory file named *hosts* at the top directory of the cloned repository with content similar to the following, changing IP addresses it required.

>  [gluster-prd]
> 192.168.8.11
> 192.168.8.12
> 192.168.8.13
> 
> [ovirt-engine-prd]
> 192.168.8.21
> 
> [ovirt-hosts-prd]
> 192.168.8.22
> 
> [ovirt-prd:children] 
> ovirt-engine-prd 
> ovirt-hosts-prd
> 
> [prd:children] 
> ovirt-prd 
> gluster-prd
> 
> [gluster-dr]
> 192.168.70.11
> 192.168.70.12
> 192.168.70.13
> 
> [ovirt-engine-dr]
> 192.168.70.21
> 
> [ovirt-hosts-dr]
> 192.168.70.22
> 
> [ovirt-dr:children] 
> ovirt-engine-dr 
> ovirt-hosts-dr
> 
> [dr:children] 
> ovirt-dr 
> gluster-dr

### Optional: Deploy libvirt VMs
If you'd like to use this only to learn how RHV and RHGS may work together, instead of deploying the software on physical servers, you may deploy everything on virtual machines and use nested virtualization features available in operating systems like Fedora 28 or Red Hat Enterprise Linux 7. 

#### Configuration
I would recommend running this in a hypervisor with at least 24 GB RAM. And there isn't much you can do with the default RAM and CPU configuration for the guests, besides creating one or two nested VMs on RHV. If you'd like to run real applications on the nested VMs, you should increase the resoures for the libvirt guests. These definitions are on the file *config-vars.yml*, in the *guests* variable, like in the example below:

    guests:
      gluster01:
        mem: 1024
        cpus: 1
        os_type: rhel7
        file_type: qcow2
        ip_addr: 192.168.8.11
        data_disk_size: 25
        role: gluster

| Parameter | Description  |
|--|--|
| mem | amount of RAM |
| cpus | number of vCPUs |
| os_type | used to set libvirt guest configuration |
| file_type | currently only qcow2 supported |
| ip_addr | IP address to be configured in guest |
| data_disk_size | size of additional disk, created for gluster nodes |
| role | can be gluster, rhvm or rhvh |

> **NOTE**: changing data_disk_size currently will have no effect on the size of the gluster volumes that will be created. It'll only change
> the size of the disk. To modify LV and gluster volume size it'll be
> required to edit roles/gluster/templates/gdeploy.conf.j2.

Other variables in *config-vars.yml* that may need modification:

| Variable | Description  |
|--|--|
| reg_user | regular user to be created and have sudo configured on the image |
| reg_user_pass | password for reg_user |
| reg_user_sshkey | path to public ssh key to be added to image |
| vm_location | directory where libvirt stores guests disks in your system - usually /var/lib/libvirt/images |
| disk_image | path where to find RHEL cloud image in your system - note that this file **will be modified**|
| bridge_name | name of the bridge that will be created by libvirt - **should not exist in the system** |
| bridge_ip_addr | first IP address to be configured in the bridge, used as guests gateway - NAT enabled |
| bridge_netmask | netmask to be configured for bridge_ip_addr |

**NOTE**: RHEL cloud image is being used for all roles (gluster, rhvm, rhvh). 

#### Usage
After finishing configuration, execute deploy-libvirt-vms.yml as follows:

    $ ansible-playbook deploy-libvirt-vms.yml --extra-vars "prepare_image=true"

Providing --extra-vars will be only needed in the first execution of the playbook, when the image will be modified to disable cloud-init, add the admin user and inject the public ssh key. 

### Deploy RHGS

Run the playbook to deploy RHGS.

    $ ansible-playbook configure-gluster.yml

It will enable required repositories, install packages and configure gluster volumes to be used by RHV.

### Deploy RHV
Run the playbook to deploy RHV.

    $ ansible-playbook configure-rhv.yml

This playbook will:

- enable required repositories
- install packages on ovirt-engine and hypervisors
- configure ovirt-engine
- configure hypervisors
- configure a Data Center and a Cluster in RHV
- configure Storage Domains

RHV is now ready to be used.
