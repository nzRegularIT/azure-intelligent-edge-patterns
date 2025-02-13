# AzureStack SSTP VPN Server

***This template is intended for use in an Azure Stack environment.***

The purpose of this template is to offer a solution for providing clients with a point to site SSTP VPN into their Azure Stack Virtual Network. It is currently not possible to use a point-to-site connection to the Azure Stack Virtual Network Gateway.

# This is currently a work in progress.

## Note: only the scripts azuredeploy.json and azuredeploy.parameters.json are currently working to deploy a single Windows VM

## Ignore the remaining steps, documentation pending!

### Deployment Steps

1. Deploy Infrastructure to Instance A using azuredeploy.json
2. Deploy Infrastructure to Instance B using azuredeploy.json
3. Use outputs from step 2 to deploy azuredeploy.tunnel.gre.json or azuredeploy.tunnel.ike.json to Instance A
4. Use outputs from step 1 to deploy azuredeploy.tunnel.gre.json or azuredeploy.tunnel.ike.json to Instance B
5. Tunnel will connnect between endpoints

## Requirements

- ASDK or Azure Stack Integrated System with latest updates applied. 
- Required Azure Stack Marketplace items:
    -  Windows Server 2016 Datacenter or Windows Server 2019 Datacenter (latest build recommended)
	-  PowerShell DSC extension
    -  Custom Script Extension

## Template Inputs & Outputs

There are several JSON parameters files with default values to assist you in deploying this in your own environments.

### Inputs for azuredeploy.json

|**Parameters**|**default**|**description**|
|------------------|---------------|------------------------------|
|WindowsImageSKU         |2019-Datacenter   |Please select the base Windows VM image
|adminUsername           |rrasadmin         |The name of the Administrator of the new VMs"
|adminPassword           |                  |The password for the Administrator account of the new VMs
|VNetName                |VNet              |The name of VNet.  THis will be used to label the resources
|VNetAddressSpace        |10.10.0.0/16      |Address Space for VNet
|VNetInternalSubnetName  |Internal          |VNet Internal Subnet Name
|VNetTunnelSubnetRange   |10.10.254.0/24    |VNet Tunnel Subnet Range
|VNetTunnelGW            |10.10.254.4       |Static Address for VNet RRAS Server
|VNetInternalSubnetRange |10.10.1.0/24      |Address Range for VNet Internal Subnet
|VNetInternalGW          |10.10.1.4         |Static Address for VNet RRAS Server.  Used for User defined route in Route table
|_artifactsLocation      ||
|_artifactsLocationSasToken||

### Outputs from azuredeploy.json

|**Output**|
|-------------|
|LocalTunnelEndpoint
|LocalVNetAddressSpace
|LocalVNetGateway
|adminUsername
|VNet
|InternalRefVNet
|VNetInternalSubnetName
|InternalSubnetRefVNet

### Inputs for azuredeploy.tunnel.ike.json

|**Inputs**|**Outputs from azuredeploy.json**|
|-------------|-----------|
|RemoteTunnelEndpoint|LocalTunnelEndpoint|
|RemoteVNetAddressSpace|LocalVNetAddressSpace|
|RemoteVNetGateway|LocalVNetGateway|
|SharedSecret|
|_artifactsLocation|
|_artifactsLocationSasToken|

### Inputs for azuredeploy.tunnel.gre.json

the LocalVNetGateway is from the resourcegroup you are deploying to not from the remote resource group

|**Inputs**|**Outputs from azuredeploy.json**|
|-------------|-----------|
|RemoteTunnelEndpoint|LocalTunnelEndpoint|
|RemoteVNetAddressSpace|LocalVNetAddressSpace|
|RemoteVNetGateway|LocalVNetGateway|
|LocalTunnelGateway|
|_artifactsLocation|
|_artifactsLocationSasToken|

## Things to Consider

- A Network Security Group is applied to the template Tunnel Subnet.  It is recommended to secure the internal subnet in each VNet with an additional NSG.
- An RDP Deny rule is applied to the Tunnel NSG and will need to be set to allow if you intend to access the VMs via the Public IP address
- This solution does not take into account DNS resolution
- The resource group name is used for the VM and for the route table so that the Tunnel template can find the RRAS VM and RouteTable resources easily without user input.  The user can however label the VNet and Subnets to make this more relervant to its usage.
- The combination of Resource Group and vmName must be less than 15 characters.  Eg 'Resourcegr-RRAS'
- This template is using BYOL Windows License
- When deleting the resource group, currently on (1907) you have to manually detach the NSG's from the tunnel subnet to ensure the delete resource group completes
- This template is using a DS3v2 vm as default there other options but you many want to change the allowed values.  The RRAS service installs and run Windows internal SQL Server.  This can cause memory issues if your VM size is too small.  Validate performance before reducing the VM size.
- This is not a highly avaliable solution.  If you require a more HA style solution you can add a second VM, you would have to manually Change the route in the route table to the internal IP of the secondary interface.  You would also need to configure the mutliple Tunnels to cross connect.
- You may also want to consider using Accelerated Networking for greater throughput.

## Optional

- You can use your own Blob storage account and SAS token using the _artifactsLocation and _artifactsLocationSasToken parameters the ability to use your own storage blob with SAS token.
- This template provides default values for VNet naming and IP addressing.  You will need to change the address space for each side
- Be careful to keep these values within legal subnet and address ranges as deployment may fail.  
- The powershell DSC package is executed on each RRAS VM and installing routing and all required dependent services and features.  This DSC can be customized further if needed. These are the two DSC packages present https://github.com/PowerShell/ComputerManagementDsc/ https://github.com/mgreenegit/xRemoteAccess/.  xRemoteAccess is present but not used currently.
- The custom script extension runs the following script Add-Site2SiteIKE.ps1 and Add-Site2SiteGRE.ps1 and configures the VPNS2S tunnel between the two RRAS servers.  You can view the detailed output from the custom script extension to see the results of the VPN tunnel configuration

## Walkthrough

A detailed guide for the Mutliple VPN tunnel walkthrough can be found here
https://github.com/lucidqdreams/azure-intelligent-edge-patterns/blob/master/rras-vnet-vpntunnel/Source/Walkthrough-Multiple-VPN-Tunnels.docx?raw=true
