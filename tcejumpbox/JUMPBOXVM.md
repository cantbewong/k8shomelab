# How to Install a TCE bootstrap/”jumpbox” host VM on vSphere
We will start with an official published Ubuntu LTS server OVA and customize and configure it during deployment to vSphere. The deployed Ubuntu VM will be fully patched with pre-installed docker-ce runtime, kubectl, and tanzu CLI - ready to use for Tanzu Community Edition deployments to vSphere. After deploying a Kubernetes cluster, this VM can also be used to manage the cluster.

## Pre-requisites
1. Recommended: Visual Studio Code editor with YAML(Red Hat) and vscode-base64 extensions installed
2. An ssh key pair of your choice - recent distros are deprecating support for RSA keys so ED25519 based key pair recommended.
3. A vSphere resource pool (or cluster) and folder, a network port group with routing to the Internet and a running DHCP server offering a subset of available IPs. (Static IPs could be made to work but this is not documented here.) more detail [here](https://tanzucommunityedition.io/docs/v0.12/vsphere/)
4. Create a new account called **tce-user** in your LDAP system which is linked to vCenter. Create a new role called TCE in vCenter with permissions as described [here](https://tanzucommunityedition.io/docs/v0.12/ref-vsphere/) - and associate the new user with the new role, and then associate with vCenter objects as described in documentation.
5. You will need a VMware Customer Connect account to download OVAs, register [here](https://customerconnect.vmware.com/account-registration)


Get OVAs and place them in a TCE content library

Instructions assume using Windows and the next two downloads will be stored in C:\OVAs\tce
Download an OVA to be used for Kubernetes cluster node VMs from (note, version specific)
[https://customerconnect.vmware.com/downloads/get-download?downloadGroup=TCE-0120](https://customerconnect.vmware.com/downloads/get-download?downloadGroup=TCE-0120)

Download a Ubuntu server 20 cloud-init OVA to be used to deploy a TCE “jumpbox” VM
Ubuntu cloud images are here [https://cloud-images.ubuntu.com/focal/current/](https://cloud-images.ubuntu.com/focal/current/)
You want the OVA for VMware/Virtualbox
[https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.ova](https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.ova)

start a powershell ISE windows as administrator
```powershell 
Install-Module -Name VMware.PowerCLI

Connect-VIServer -Server myvcenterfqdn -User myusername@domain -Password mypassword
# show all the available datastores
Get-Datastore
#create a new content library
New-ContentLibrary -Datastore myDatastore -Name ContentLibTCE
# put the Kubernetes cluster node OVA in the content Library
New-ContentLibraryItem -ContentLibrary ContentLibTCE -Name photon-3-kube-v1.22.8+vmware.1-tkg.1-d69148b2a4aa7ef6d5380cc365cac8cd.ova -Files C:\OVAs\tce\photon-3-kube-v1.22.8+vmware.1-tkg.1-d69148b2a4aa7ef6d5380cc365cac8cd.ova
# put the Ubuntu Server 20 cloud-init OVA in the content library
New-ContentLibraryItem -ContentLibrary ContentLibTCE -Name focal-server-cloudimg-amd64.ova -Files C:\OVAs\tce\focal-server-cloudimg-amd64.ova
```

# Create a cloud-init yaml
Using Visual Studio create a new file named **tcejumpbox-init.yml** with this content (change ssh public key to your own, and change KUBECTLVER, TCEVER and HELMVER if you like):

See example file in this repo.
https://github.com/cantbewong/k8shomelab/blob/95fc2b291e47670cbb9c0dbd4a639ff27d870ac4/tcejumpbox/tcejumpbox-init.yml


Save the file then do “Save As” and save again as **tcejumpbox-init.base64**. Hit ctl A to select all text, then ctl-E ctl-E to convert to base64 (assumes you installed the base64 plugin to VisualStudioCode). Save again in encoded form.

## Create a TCE Jumpbox VM from the Ubuntu cloud-init OVA

Go to the TCE content library and right click on the OVA to create a new VM called tcejumpbox placing it in the resource pool associated with TCE.

In Customize template stage

1. Set Instance ID to tcejumpbox (same as VM name)
2. Set hostname to tcejumpbox (same as VM name)
3. Leave URL to seed instance data untouched and empty
4. Set ssh public keys to the same public key you used in the cloud-init file
5. Set Encoded user-data to the Base64 encoded version of the cloud-init file that we converted in VisualStudioCode - just cut and paste from the editor
6. set password to changeme
7. press NEXT

## Configure the TCE jumpbox VM and power it on

After the jumpbox VM is created, edit VM settings to 8GB of memory and 64GB of disk. Optionally you can enable cut and paste to console window as described [here](https://kb.vmware.com/s/article/57122). If you think you may be doing this frequently use the name tcejumpboxtemplate instead and convert this to a template without ever powering it on - then deploy/customize jumpboxes from the template.

Power on the VM and open a remote console window to watch. You can also watch cpu, and network consumption using the Monitor/Overview tab. You should see it start and consume a burst of resource as it initializes the VM, after about 5 minutes (depends on Internet download speed) it will complete and power itself off.

Power the VM back on and logon as default user ubuntu, password changeme. It will force you to enter a new password. You should also be able to ssh in using your ssh key once you determine a network IP.

The tanzu CLI is designed for use by a non-root user. You can login as the second user (tceadmin) using ssh with the key associated with the public one you submitted in user-data when the VM was started the first time.

If things went as expected, you can invoke the Tanzu CLI to deploy a TCE management cluster 

```shell
tanzu management-cluster create --ui --bind <VM ip here>:8080 --browser none
```

If things did not go well, after login see logs inside VM at 

```shell
 sudo more /var/log/cloud-init-output.log
 sudo more /var/log/cloud-init.log
 ```
