# packerimage

How to use Packer to create Linux virtual machine images in Azure
Create Azure resource group
During the build process, Packer creates temporary Azure resources as it builds the source VM. To capture that source VM for use as an image, you must define a resource group. The output from the Packer build process is stored in this resource group.

Create a resource group with az group create. The following example creates a resource group named myResourceGroup in the eastus location:

Azure CLI

Copy
az group create -n myResourceGroup -l eastus
Create Azure credentials
Packer authenticates with Azure using a service principal. An Azure service principal is a security identity that you can use with apps, services, and automation tools like Packer. You control and define the permissions as to what operations the service principal can perform in Azure.

Create a service principal with az ad sp create-for-rbac and output the credentials that Packer needs:

Azure CLI

Copy
az ad sp create-for-rbac --query "{ client_id: appId, client_secret: password, tenant_id: tenant }"
An example of the output from the preceding commands is as follows:

Output

Copy
{
    "client_id": "f5b6a5cf-fbdf-4a9f-b3b8-3c2cd00225a4",
    "client_secret": "0e760437-bf34-4aad-9f8d-870be799c55d",
    "tenant_id": "72f988bf-86f1-41af-91ab-2d7cd011db47"
}
To authenticate to Azure, you also need to obtain your Azure subscription ID with az account show:

Azure CLI

Copy
az account show --query "{ subscription_id: id }"
You use the output from these two commands in the next step.

Define Packer template
To build images, you create a template as a JSON file. In the template, you define builders and provisioners that carry out the actual build process. Packer has a provisioner for Azure that allows you to define Azure resources, such as the service principal credentials created in the preceding step.

Create a file named ubuntu.json and paste the following content. Enter your own values for the following:

DEFINE PACKER TEMPLATE
Parameter	Where to obtain
client_id	First line of output from az ad sp create command - appId
client_secret	Second line of output from az ad sp create command - password
tenant_id	Third line of output from az ad sp create command - tenant
subscription_id	Output from az account show command
managed_image_resource_group_name	Name of resource group you created in the first step
managed_image_name	Name for the managed disk image that is created
JSON

Copy
{
  "builders": [{
    "type": "azure-arm",

    "client_id": "f5b6a5cf-fbdf-4a9f-b3b8-3c2cd00225a4",
    "client_secret": "0e760437-bf34-4aad-9f8d-870be799c55d",
    "tenant_id": "72f988bf-86f1-41af-91ab-2d7cd011db47",
    "subscription_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxx",

    "managed_image_resource_group_name": "myResourceGroup",
    "managed_image_name": "myPackerImage",

    "os_type": "Linux",
    "image_publisher": "Canonical",
    "image_offer": "UbuntuServer",
    "image_sku": "16.04-LTS",

    "azure_tags": {
        "dept": "Engineering",
        "task": "Image deployment"
    },

    "location": "East US",
    "vm_size": "Standard_DS2_v2"
  }],
  "provisioners": [{
    "execute_command": "chmod +x {{ .Path }}; {{ .Vars }} sudo -E sh '{{ .Path }}'",
    "inline": [
      "apt-get update",
      "apt-get upgrade -y",
      "apt-get -y install nginx",

      "/usr/sbin/waagent -force -deprovision+user && export HISTSIZE=0 && sync"
    ],
    "inline_shebang": "/bin/sh -x",
    "type": "shell"
  }]
}
This template builds an Ubuntu 16.04 LTS image, installs NGINX, then deprovisions the VM.

 Note

If you expand on this template to provision user credentials, adjust the provisioner command that deprovisions the Azure agent to read -deprovision rather than deprovision+user. The +user flag removes all user accounts from the source VM.

Build Packer image
If you don't already have Packer installed on your local machine, follow the Packer installation instructions.

Build the image by specifying your Packer template file as follows:

Bash

Copy
./packer build ubuntu.json
An example of the output from the preceding commands is as follows:

Output

Copy
azure-arm output will be in this color.

==> azure-arm: Running builder ...
    azure-arm: Creating Azure Resource Manager (ARM) client ...
==> azure-arm: Creating resource group ...
==> azure-arm:  -> ResourceGroupName : ‘packer-Resource-Group-swtxmqm7ly’
==> azure-arm:  -> Location          : ‘East US’
==> azure-arm:  -> Tags              :
==> azure-arm:  ->> dept : Engineering
==> azure-arm:  ->> task : Image deployment
==> azure-arm: Validating deployment template ...
==> azure-arm:  -> ResourceGroupName : ‘packer-Resource-Group-swtxmqm7ly’
==> azure-arm:  -> DeploymentName    : ‘pkrdpswtxmqm7ly’
==> azure-arm: Deploying deployment template ...
==> azure-arm:  -> ResourceGroupName : ‘packer-Resource-Group-swtxmqm7ly’
==> azure-arm:  -> DeploymentName    : ‘pkrdpswtxmqm7ly’
==> azure-arm: Getting the VM’s IP address ...
==> azure-arm:  -> ResourceGroupName   : ‘packer-Resource-Group-swtxmqm7ly’
==> azure-arm:  -> PublicIPAddressName : ‘packerPublicIP’
==> azure-arm:  -> NicName             : ‘packerNic’
==> azure-arm:  -> Network Connection  : ‘PublicEndpoint’
==> azure-arm:  -> IP Address          : ‘40.76.218.147’
==> azure-arm: Waiting for SSH to become available...
==> azure-arm: Connected to SSH!
==> azure-arm: Provisioning with shell script: /var/folders/h1/ymh5bdx15wgdn5hvgj1wc0zh0000gn/T/packer-shell868574263
    azure-arm: WARNING! The waagent service will be stopped.
    azure-arm: WARNING! Cached DHCP leases will be deleted.
    azure-arm: WARNING! root password will be disabled. You will not be able to login as root.
    azure-arm: WARNING! /etc/resolvconf/resolv.conf.d/tail and /etc/resolvconf/resolv.conf.d/original will be deleted.
    azure-arm: WARNING! packer account and entire home directory will be deleted.
==> azure-arm: Querying the machine’s properties ...
==> azure-arm:  -> ResourceGroupName : ‘packer-Resource-Group-swtxmqm7ly’
==> azure-arm:  -> ComputeName       : ‘pkrvmswtxmqm7ly’
==> azure-arm:  -> Managed OS Disk   : ‘/subscriptions/guid/resourceGroups/packer-Resource-Group-swtxmqm7ly/providers/Microsoft.Compute/disks/osdisk’
==> azure-arm: Powering off machine ...
==> azure-arm:  -> ResourceGroupName : ‘packer-Resource-Group-swtxmqm7ly’
==> azure-arm:  -> ComputeName       : ‘pkrvmswtxmqm7ly’
==> azure-arm: Capturing image ...
==> azure-arm:  -> Compute ResourceGroupName : ‘packer-Resource-Group-swtxmqm7ly’
==> azure-arm:  -> Compute Name              : ‘pkrvmswtxmqm7ly’
==> azure-arm:  -> Compute Location          : ‘East US’
==> azure-arm:  -> Image ResourceGroupName   : ‘myResourceGroup’
==> azure-arm:  -> Image Name                : ‘myPackerImage’
==> azure-arm:  -> Image Location            : ‘eastus’
==> azure-arm: Deleting resource group ...
==> azure-arm:  -> ResourceGroupName : ‘packer-Resource-Group-swtxmqm7ly’
==> azure-arm: Deleting the temporary OS disk ...
==> azure-arm:  -> OS Disk : skipping, managed disk was used...
Build ‘azure-arm’ finished.

==> Builds finished. The artifacts of successful builds are:
--> azure-arm: Azure.ResourceManagement.VMImage:

ManagedImageResourceGroupName: myResourceGroup
ManagedImageName: myPackerImage
ManagedImageLocation: eastus
It takes a few minutes for Packer to build the VM, run the provisioners, and clean up the deployment.

Create VM from Azure Image
You can now create a VM from your Image with az vm create. Specify the Image you created with the --image parameter. The following example creates a VM named myVM from myPackerImage and generates SSH keys if they do not already exist:

Azure CLI

Copy
az vm create \
    --resource-group myResourceGroup \
    --name myVM \
    --image myPackerImage \
    --admin-username azureuser \
    --generate-ssh-keys
If you wish to create VMs in a different resource group or region than your Packer image, specify the image ID rather than image name. You can obtain the image ID with az image show.

It takes a few minutes to create the VM. Once the VM has been created, take note of the publicIpAddress displayed by the Azure CLI. This address is used to access the NGINX site via a web browser.

To allow web traffic to reach your VM, open port 80 from the Internet with az vm open-port:

Azure CLI

Copy
az vm open-port \
    --resource-group myResourceGroup \
    --name myVM \
    --port 80
Test VM and NGINX
Now you can open a web browser and enter http://publicIpAddress in the address bar. Provide your own public IP address from the VM create process. The default NGINX page is displayed as in the following example:

NGINX default site

Next steps
You can also use existing Packer provisioner scripts with Azure Image Builder.

Recommended content
Use cloud-init to add a user to a Linux VM on Azure - Azure Virtual Machines
How to use cloud-init to add a user to a Linux VM during creation with the Azure CLI

Use cloud-init to run a bash script in a Linux VM on Azure - Azure Virtual Machines
How to use cloud-init to run a bash script in a Linux VM during creation with the Azure CLI

Use Azure Image Builder with an image gallery for Linux VMs (preview) - Azure Virtual Machines
Create Linux VM images with Azure Image Builder and Shared Image Gallery.

