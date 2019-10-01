# ocpvmware


For deploying OCP 4.1  a minimum recommendation is to provision 1 ESXi server and 1 Centos/Redhat VSI on the same VLAN in IBM Cloud for Government. 

The Centos/Redhat VSI is only required for a few hours and can de-provisioned after the install is complete. 


> **NOTE**  Openshift 4.1 has a complicated installation.  Use the following URL to access the official RedHat documentation on installing Openshift 4.1 on VMware: [URL](https://docs.openshift.com/container-platform/4.1/installing/installing_vsphere/installing-vsphere.html) 

> The information in this document is written in a condensed format. For a more verbose explanation please refer to [URL](https://docs.openshift.com/container-platform/4.1/installing/installing_vsphere/installing-vsphere.html)
 
> The automation and manual steps can all be pointed back to the above URL. Before you begin, understanding your IP address is very important.  The IP addresses in the following table were obtained from IC4G.  They are listed here for illustration purpose only. Besides setting up your ESXi and vCenter server, you also need to order a minimum of 16 portable IP address which will be used to assign to the VMs. 

> Each VM node takes up one IP address.  The recommendation minimum of 16 portable IP addresses is determined by:
> 1 helper node + 1 boot node + 3 control-plane nodes + 3 worker nodes = 8 nodes
> IC4G reserves 4 IP addresses out of every portable IP subnet.  Therefore 8 + 4 = 12.
> The extra four IP addresses are for having a cushion.  This installation provisioned the vCenter on the same portable IP subnet, thus a total of 9 IP addresses are used.

## 	Architecture Diagram 
coming soon!!!!!!!!!!!!!!!!!

## Hardware requirements

| Node Name       | vCPU   | Mem  | HDD | Role
| ------          | ------ |----  | --- | ------ |
| Helper Node | 4  | 16 | 150 | DNS/Proxy/DHCP/OCP Installer|
| Bootstrap-0 | 4  | 16 | 150 | Bootstrap OCP |
| Control-plane-0  |  4 | 16 | 150 | Master OCP |
| Control-plane-1  |  4 | 16 | 150 | Master OCP |
| Control-plane-2  |  4 | 16 | 150 | Master OCP |
| compute-0 | 4 | 16 | 150 | Compute OCP |
| compute-1 | 4 | 16 | 150 | Compute OCP |
| compute-2 | 4 | 16 | 150 | Compute OCP |

## Prep the VIS System 

- Install Required Packages.

- Download vCenter ISO image from VMware. (VMware website requires an account to download the ISO image)

#### Install Packages 
Before we can use ansible scripts, we have to prep the host with installing ansible rpm and python library. 

```
sudo yum update
sudo yum install ansible
sudo yum install genisoimage
sudo yum install git
sudo yum install python-pip gcc make openssl-devel python-devel
sudo pip install --upgrade ansible
sudo pip install PyVmomi
sudo yum install p7zip*
sudo pip install jmespath
curl -L https://github.com/vmware/govmomi/releases/download/v0.20.0/govc_linux_amd64.gz | gunzip > /usr/local/bin/govc
chmod +x /usr/local/bin/govc
```

> ***HINT*** for Redhat server pip install command wil fail

>  subscription-manager repos --enable rhel-server-rhscl-7-rpms

>  yum install python27-python-pip

>$ scl enable python27 bash

>$ which pip

>$ pip -V

#### Download vCenter Server Appliance 
Download the ISO images from [URL](https://my.vmware.com/web/vmware/details?downloadGroup=VC67U2&productId=742&rPId=33237)

move it to /opt/repo
Update the vcsa_ova variable in vars.yaml file with the downloaded ISO image name.

#### Create a Portgroup VMware
Name = vmportgroup

You can follow [URL](https://docs.vmware.com/en/VMware-vSphere/6.5/com.vmware.vsphere.html.hostclient.doc/GUID-67415625-FB59-4AE0-9E16-4FB39AEBC50B.html) VMware reference document to create a portgroup. 

#### Update NTP server ESXi
Connect to your ESXi host via vSphere host client  https://<ESXI_IP>/ui

Then on the left, just bellow the host, select the Manage > System > Time and Date

Update the NTP Server with correct NTP Server address for example 129.6.15.28,129.6.15.28

Then click the Edit Settings button to bring up the configuration window. Set the NTP service startup policy as “Start and stop with host”. Like this everytime the host will reboot, the NTP service will be started automatically.

#### Download the Git repository
```
cd /opt
git clone https://github.com/fctoibm/ocpvmware.git
cd /opt/ocpvmware
```
> *** HINT *** For Redhat you might have to update the ansible path if playbook can not load python modules
> In ansible.cfg
> Under the [defaults]
>interpreter_python = /opt/rh/python27/root/usr/bin/python

Edit the [vars.yaml](./vars.yaml) file with the IP addresss that will be assigned to the masters/workers/boostrap. The IP addresses need to be right since they will be used to create your OpenShift servers.

Edit the [hosts](./hosts) file kvmguest section to match helper node information. This should be similar to vars.yaml file 


## Running the playbooks

End-User can deploy the entire stack from vCenter to OCP4.1  using playbook 1, 2, and 3. In case you already have Vcenter deployed you can skip the playbook 1. 



### Run the playbook One

Run the playbook to setup your vCenter 

```
ansible-playbook -e @vars.yaml  play1.yaml

``` 
>  **HINT** After complete deployment wait for 15-30 mins to let the vCenter deploy. Verify vCenter by visting https://<vcenter_ip> URL and default username Administrator@vshere.local  before executing playbook 2.
>   You can also watch the vCenter install progress by opening a browser and enter following URL https://<vCenter_IP>:5480

> You can check the status by visting https://<vcenter_ip>:5480


### Run the playbook Two

Run the playbook 2 to deploy helper node OS and OCP4.1 VM's using terraform. 

```
ansible-playbook -e @vars.yaml  play2.yaml

``` 

> **HINT** You will have press enter a key during Playbook 2 execution, this is done so end-user can verify helper VM deployed successfully. 


### Run the playbook Three

Run the playbook 3  updates the helper node  which acts as LB/DSN/DHCP/PEX. This playbook will also restart the OCP VM's

```
ansible-playbook -e @vars.yaml  play3.yaml

``` 

### Playbook fail for some reason 
If the ansible scripts fail you can execute the following script to clean the environment but do it your own risk.

```
ansible-playbook -e @vars.yaml  clean_ocp_vms.yaml
```

> **HINT** this will delete all the  OCP related VM's and you execute Play2 and Play3 playbook

```
ansible-playbook -e @vars.yaml  clean_everything.yaml
```

> **HINT** this will delete all the VM's and you execute Play1, Play2 and Play3 playbook

## Wait for install

The boostrap VM actually does the install for you; you can track it with the following command by ssh into helper node guest KVM.

```
cd /opt/ocp4
openshift-install wait-for bootstrap-complete --log-level debug
```

Once you see this message below...

```
DEBUG OpenShift Installer v4.1.0-201905212232-dirty 
DEBUG Built from commit 71d8978039726046929729ad15302973e3da18ce 
INFO Waiting up to 30m0s for the Kubernetes API at https://api.ocp4.example.com:6443... 
INFO API v1.13.4+838b4fa up                       
INFO Waiting up to 30m0s for bootstrapping to complete... 
DEBUG Bootstrap status: complete                   
INFO It is now safe to remove the bootstrap resources
```

...you can continue....at this point you can delete the bootstrap server.

## Finish Install

First, ssh into helper node guest KVM

```
cd /opt/ocp4
export KUBECONFIG=/opt/ocp4/auth/kubeconfig
```

Set up storage for you registry (to use PVs follow [URL](https://docs.openshift.com/container-platform/4.1/installing/installing_bare_metal/installing-bare-metal.html#registry-configuring-storage-baremetal_installing-bare-metal) )

```
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```

If you need to expose the registry, run this command

```
oc patch configs.imageregistry.operator.openshift.io/cluster --type merge -p '{"spec":{"defaultRoute":true}}'
```

finish up the install process
```
openshift-install wait-for install-complete 
```
Following message should be shown 
```
INFO Waiting up to 30m0s for the cluster at https://api.test.os.fisc.lab:6443 to initialize... 
INFO Waiting up to 10m0s for the openshift-console route to be created... 
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/opt/ocp4/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.test.os.fisc.lab 
INFO Login to the console with user: kubeadmin, password: ###-????-@@@@-**** 
```
## Access OCP URL
> Add following lines to your /etc/hosts files on from where you plan to access the Opensshift URL 

> <Helper_HOST_IP> console-openshift-console.apps.<base_domain_prefix>.<base_domain>  oauth-openshift.apps.<base_domain_prefix>.<base_domain>


