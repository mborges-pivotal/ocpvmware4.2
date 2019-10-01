ocpvmware
=========

For deploying OCP 4.1 a minimum recommendation is to provision 1 ESXi
server and 1 Centos/Redhat VSI (Virtual Server Infrastructure) on the
same VLAN in IBM Cloud for Government.

The Centos/Redhat VSI is needed to run a series of ansible scripts to
provision a vCenter and to deploy OCP. It is only required for a few
hours and can be de-provisioned after the OCP deployment is completed.

> **NOTE** Before you begin, understanding your IP address is very
> important. The IP addresses in the following table were obtained from
> IC4G. They are listed here for illustration purpose only. Besides
> setting up your ESXi, you also need to order a minimum of 16 portable
> IP address which will be used to assign to the VMs.
>
> Each VM node takes up one IP address. The recommendation minimum of 16
> portable IP addresses is determined by: 1 helper node + 1 boot node +
> 3 control-plane nodes + 3 worker nodes = 8 nodes IC4G reserves 4 IP
> addresses out of every portable IP subnet. Therefore 8 + 4 = 12. The
> extra four IP addresses are for having a cushion. This installation
> provisioned the vCenter on the same portable IP subnet, thus a total
> of 9 IP addresses are used.

Architecture Diagram
--------------------

coming soon!!!!!!!!!!!!!!!!!

Hardware requirements
---------------------

  Node Name         vCPU   Mem   HDD   Role
  ----------------- ------ ----- ----- ------------------------------
  Helper Node       4      16    150   DNS/Proxy/DHCP/OCP Installer
  Bootstrap-0       4      16    150   Bootstrap OCP
  Control-plane-0   4      16    150   Master OCP
  Control-plane-1   4      16    150   Master OCP
  Control-plane-2   4      16    150   Master OCP
  compute-0         4      16    150   Compute OCP
  compute-1         4      16    150   Compute OCP
  compute-2         4      16    150   Compute OCP

Prep the VIS System
-------------------

-   Provision a VSI with 2 vCPUs, 10 GB of RAM, and 100 GB of storage

-   Install Required Packages.

-   Download vCenter ISO image from VMware. (VMware website requires an
    account to download the ISO image)

#### Install Packages

After the VSI is provisioned, logon to the VSI as root. To run ansible
scripts, ansible rpm and python library need to be installed:

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

> ***HINT*** for Redhat server pip install command will fail. Need to
> execute the following commands
>
> sudo subscription-manager repos --enable rhel-server-rhscl-7-rpms
>
> sudo yum install python27-python-pip
>
> sudo scl enable python27 bash
>
> sudo which pip
>
> sudo pip -V

#### Download vCenter Server Appliance

Download the ISO images from
[URL](https://my.vmware.com/web/vmware/details?downloadGroup=VC67U2&productId=742&rPId=33237)

Move it to /opt/repo

Later in the step where vars.yaml is to be modified, update the
vcenter\_iso\_name variable in vars.yaml with the downloaded ISO image
name.

#### Create a Portgroup VMware

Name = vmportgroup

Refer to the
[URL](https://docs.vmware.com/en/VMware-vSphere/6.5/com.vmware.vsphere.html.hostclient.doc/GUID-67415625-FB59-4AE0-9E16-4FB39AEBC50B.html)
VMware reference document for instruction on creating a portgroup.

#### Update NTP server on the ESXi 

Connect to the ESXi host via vSphere host client https://\<ESXI\_IP\>/ui

Then on the left, just bellow the host, select the Manage \> System \>
Time and Date

Update the NTP Server with correct NTP Server address for example :
129.6.15.28,129.6.15.28

Then click the Edit Settings button to bring up the configuration
window. Set the NTP service startup policy as "Start and stop with
host". That selection means that every time the host reboot, the NTP
service will be started automatically.

#### Download the Git repository

    cd /opt
    git clone https://github.com/fctoibm/ocpvmware.git
    cd /opt/ocpvmware

> \*\*\* HINT \*\*\* For Redhat you might have to update the ansible
> path if playbook can not load python modules In ansible.cfg Under the
> \[defaults\] interpreter\_python =
> /opt/rh/python27/root/usr/bin/python

Edit the [vars.yaml](./vars.yaml) file with the IP addresss that will be
assigned to the masters/workers/boostrap. The IP addresses need to be
correct since they will be used to create the OpenShift servers.

Edit the [hosts](./hosts) file kvmguest section to match helper node
information. This should be similar to the helpernode\_IP in the
vars.yaml file.

Running the playbooks
---------------------

Running playbook 1, 2 and 3 will deploy the entire stack from vCenter to
OCP v4.1. If vCenter already exist, skip playbook one and continue with
playbook two.

### Run the playbook One

Run the following playbook to setup vCenter:

    ansible-playbook -e @vars.yaml  play1.yaml

> **HINT** After play1.yaml is completed, wait 15-30 mins to let the
> vCenter completes its deployment. Before proceeding to playbook 2,
> verify that vCenter has completed deployment by visiting
> https://\<vcenter\_ip\> and logon using the credential entered in the
> vars.yaml file. The vCenter installation progress can be monitored by
> opening a browser and entering the following URL
> https://\<vCenter\_IP\>:5480

### Run the playbook Two

Run the playbook 2 to deploy helper node OS and the OCP4.1 VM's using
terraform.

    ansible-playbook -e @vars.yaml  play2.yaml

> **HINT** End user will be prompted to press a key during Playbook 2
> execution before the script continues. That is done so end-user can
> verify that the helper node VM deployed successfully.

### Run the playbook Three

Playbook 3 will update the helper node to act as Load Balancer/ Dynamic
Host Configuration Protocol / Preboot Execution Environment / Domain
Name System (LB/DHCP/PEX/DNS). This playbook will also restart the OCP
VM's

    ansible-playbook -e @vars.yaml  play3.yaml

### Playbook fails for some reason

If the ansible scripts fail, execute the following script to clean the
environment.

    ansible-playbook -e @vars.yaml  clean_ocp_vms.yaml

> **HINT** The above command will delete all the OCP related VM's.
> Execute Play2 and Play3 playbooks to resume OCP installation.

    ansible-playbook -e @vars.yaml  clean_everything.yaml

> **HINT** The above command will delete the vCenter and all the OCP
> related VMs. Execute Play1, Play2 and Play3 playbook to resume OCP
> installation.

Wait for bootstrap VM to complete installation
----------------------------------------------

The bootstrap VM actually does the install. To track the progress, ssh
into helper node and execute the following commands:

    cd /opt/ocp4
    openshift-install wait-for bootstrap-complete --log-level debug

Once you see this message below...

    DEBUG OpenShift Installer v4.1.0-201905212232-dirty 
    DEBUG Built from commit 71d8978039726046929729ad15302973e3da18ce 
    INFO Waiting up to 30m0s for the Kubernetes API at https://api.ocp4.example.com:6443... 
    INFO API v1.13.4+838b4fa up                       
    INFO Waiting up to 30m0s for bootstrapping to complete... 
    DEBUG Bootstrap status: complete                   
    INFO It is now safe to remove the bootstrap resources

...you can continue....at this point you can delete the bootstrap
server.

Finish OCP Installation
-----------------------

ssh into the helper node . Execute the following commands:

    cd /opt/ocp4
    export KUBECONFIG=/opt/ocp4/auth/kubeconfig

Set up storage for you registry (to use PVs follow URL )

    oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'

If the registry need to be exposed, execute the following command

    oc patch configs.imageregistry.operator.openshift.io/cluster --type merge -p '{"spec":{"defaultRoute":true}}'

Finish up the install process. Execute the following command:

    openshift-install wait-for install-complete 

When messages similar to the following message are displayed, that means
the installation is completed.

    INFO Waiting up to 30m0s for the cluster at https://api.test.os.fisc.lab:6443 to initialize... 
    INFO Waiting up to 10m0s for the openshift-console route to be created... 
    INFO Install complete!                            
    INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/opt/ocp4/auth/kubeconfig' 
    INFO Access the OpenShift web-console here: https://console-openshift-console.apps.test.os.fisc.lab 
    INFO Login to the console with user: kubeadmin, password: ###-????-@@@@-**** 

Access OCP URL
--------------

> Add following lines to your /etc/hosts files on from where you plan to
> access the Opensshift URL
>
> \<HELPER\_HOST\_IP\>
> console-openshift-console.apps.\<base\_domain\_prefix\>.\<base\_domain\>
> oauth-openshift.apps.\<base\_domain\_prefix\>.\<base\_domain\>
