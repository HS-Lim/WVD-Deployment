# Windows Virtual Desktop Deployment Tutorial

## Introduction
Windows Virtual Desktop (WVD) is a virtualized desktop service provided in Azure Cloud. This tutorial will cover the configuration and creation of WVDs in several parts:
1. **Initial Consent, Permissions, and Registration Requirements Setup**
2. **Virtual Network Setup and Deployment**
3. **Domain Controller Setup - Azure AD Domain Services or Azure VM DC**
4. **Host Pool, App Group, and User Setup and Deployment**
    * ***Optional:*** **ARM Template Host Pool Deployment**

## 1. Initial Consent, Permissions, and Registration Requirements Setup
In order to complete WVD setup, there are a couple of prerequisites to fulfill:
1. You should have access to an account with access to resource providers, virtual machine creation, and group assignment at the AD level. Generally speaking, this will be an account with Global Administrator role in your AD and Owner role in the Azure subscription you're setting up WVD.
2. If you want to utilize pre-existing on-prem Active Directory (AD), you require domain admin access to that AD.
3. Have access to an account that is part of the **AAD DC Administrators** group, so that it can facilitate domain joins for VMs in our host pools.
4. Make sure that **Microsoft.DesktopVirtualization** and **Microsoft.AAD** is registered under **Resource providers** in your subscription.

**Additional Tips**: 
* It's recommended that virtually all resources and services in this tutorial be deployed in the same region and resource group.

## 2. Virtual Network Setup and Deployment
A Virtual Network is required to utilize a DC or Azure AD Domain Services, as well as for the creation of VMs in our host pools (for domain joins). This section will thus go over the creation of a Virtual Network.

### Deploying a Virtual Network
To deploy a Virtual Network:
1. Within the Azure Portal, use the search bar to go to the Virtual Networks overview page.
2. Select **Add**.
3. Fill out the requisite fields under the **Basics** tab:
    * **Subscription**: Enter the subscription you'd like the resource to be in. *Ex: SAA-AWU-POC-1*
    * **Resource group**: Enter the resource group you'd like the resource to be in. *Ex: RG-SPOC-AWU-01*
    * **Name**:  Enter the name of the Virtual Network. *Ex: VNET-SPOC-AWU-01*
    * **Region**: Enter the region you'd like the resource to be in. *Ex: West US*
4. Fill out the requisite fields under the **IP Addresses** tab:
    * **IPv4 address space**: Enter the address space you'd like. *Ex: 10.0.0.0/16*
    * **Subnet name**: Select **add** to enter a subnet. Create both a default subnet and an Azure AD Domain Services subnet if you plan to use that method later. *Ex: Name: default - Range: 10.0.0.0/24, Name: AADDS-Subnet - Range: 10.0.1.0/24*
5. Select **Review + create**. Once validation is finished, select **Create**. Wait for the deployment to finish.
6. Select **Go to resource**. Go to the **DNS servers** blade and select the **Custom** radio dial. Enter **8.8.8.8** and save. If using AADDS, addresses will be added here in the future to connect the VNET to AADDS. If using a DC, the address to that DC will have to be added here later.

## 3. Domain Controller Setup - Azure AD Domain Services or Azure VM DC
Windows Virtual Desktop requires an Active Directory (either on-prem or on an Azure VM) or Azure AD Domain Services to function correctly. We will be going over both methods below:

### Azure AD Domain Services (AADDS) Method
To deploy Azure AD Domain Services:
1. Within the Azure Portal, use the search bar to go to the Azure AD Domain Services overview page.
2. Select **Add**.
3. Fill out the requisite fields under the **Basics** tab:
    * **Subscription**: Enter the subscription you'd like the service to be in. *Ex: SAA-AWU-POC-1*
    * **Resource Group**: Enter the resource group you'd like the service to be in. *Ex: RG-SPOC-AWU-01*
    * **DNS domain name**: Enter the DNS domain name. Most likely, this will just be the built-in domain name of the directory with a .onmicrosoft.com suffix. Custom domain names can be used if added to the **Custom domain names** section in Azure AD. *Ex: spuspoc.onmicrosoft.com*
    * **SKU**: Select the SKU you'd like for the service. Following the **Help me choose a SKU** link will provide pricing and performance comparisons. *Ex: Standard*
    * **Forest type**: Select **User**.
4. Fill out the requisite fields under the **Networking** tab:
    * **Virtual Network**: Select the Virtual Network created as part of **Virtual Network Setup and Deployment** above. *Ex: VNET-SPOC-AWU-01*
    * **Subnet**: Select the Azure AD Domain Service subnet from your Virtual Network. *Ex: Name: AADDS-Subnet - Range: 10.0.1.0/24*
5. Fill out the requisite fields under the **Administration** tab:
    * Everything can be left by its defaults here. Uncheck groups you'd not like notified. Additionally, add email addresses as necessary.
6. Fill out the requisite fields under the **Synchronization** tab:
    * **Synchronization type**: Select **All** if you'd like everyone in Azure AD to be sync'd to AADDS. Select **Scoped** if you'd rather sync AD groups instead.
    * If you chose **Scoped**, fill out:
        * **Synchronization scope**: Select **Select groups**, then **Add groups** to add user groups to be synchronized.
7. Select **Review + create**. Once validation is finished, select **Create**, and in the following prompt, **OK**. Wait for the deployment to finish - about 30-40 minutes.
8. Once deployment is finished, select **Go to resource**. Wait for the managed domain to be provisioned - this can take 30-40 minutes on top of the deployment.
9. Once provisioning is finished, select **Configure**. Azure will automatically update the Virtual Network.
10. Note that, to use AADDS, users must reset their passwords to force a password hash synchronization. Otherwise, AD users will not be able to connect to AADDS and therefore the WVD. Refer to documentation [here](https://aka.ms/aadds-pwsynccloud) and [here](https://aka.ms/aadds-pwsync) for password hash synchronizing cloud-only and on-prem users respectively.

**Note**: AADDS synchronizes one-way from Azure AD to AADDS, meaning Azure AD is leading. In other words, one must add users through Azure AD (and be assigned to a group if you chose scoped synchronization) to create users for WVD.

### Domain Controller Method
If you already have a DC (either on-prem or on Azure), skip to step [fill later]. Otherwise, we must create a VM and set it up as a DC to create an AD to sync with Azure. Note that this method is much more involved - a VM must be provisioned, assigned to the DC role, and setup with Azure AD Connect Sync. Additionally, a VPN must be created to secure the connection to the DC, which involves more steps like certificate management.

To setup a DC on an Azure VM:
1. Within the Azure Portal home page, select **Virtual Machines** from the left side of the screen, then **Add**.
2. Fill out the requisite fields under the **Basics** tab:
    * **Subscription**: Enter the subscription you'd like to service to be in. *Ex: SAA-AWU-POC-1*
    * **Resource group**: Enter the resource group you'd like the service to be in. *Ex: RG-SPOC-AWU-01*
    * **Virtual machine name**: Enter a descriptive name for the VM to be our DC. *Ex: WVD-DC-01*
    * **Region**: Select the region you'd like the VM to be in. *Ex: West US*
    * **Availability options**: Select **No inifrastructure redundancy required**.
    * **Image**: Select **Windows Server 2019 Datacenter**.
    * **Size**: Select the size you'd like to VM to be. *Ex: D2s_v3*
    * **Username**: Enter the username of the administrator account for this VM. *Ex: wvdadmin*
    * **Password**: Enter the password for the administrator account for this VM.
    * **Confirm password**: Enter the password again.
    * **Public inbound ports**: Select **None**.
    * **Already have a Windows Server license?**: Select **Yes** or **No** depending on whether you have a Windows Server license already. If **Yes**, confirm by checking the box that appears.
3. Fill out the requisite fields under the **Disks** tab:
    * **OS disk type**: Select a disk type. *Ex: Standard SSD*
    * **Encryption type**: Select **(Default)**.
    * Under the Data disks section, select **Create and attach a new disk** - we'll be making a disk to store AD/DC logs and other information. We can't use the automatically generated OS disk as it has caching, which might result in corruption on unexpected shutdowns. Fill out the following fields:
        * **Name**: Enter a name for the disk. *Ex: WVD-DC-01-Disk*
        * **Source type**: Select **None (empty disk)**
        * **Size**: Select a size for the disk. *Ex: 512 GiB Standard SSD*
        * **Encryption type**: Select **(Default)**.
    * Select **OK** to return to the VM configuration screen.
    * Make sure that **Host caching** is set to **None** for our newly created disk.
4. Fill out the requisite fields under the **Networking** tab:
    * **Virtual network**: Select the Virtual Network that you previously created. *Ex: VNET-SPOC-AWU-01*
    * **Subnet**: Select **default (10.0.0.0/24)** or whatever equivalent you set as the subnet for your Virtual Network.
    * **Public IP**: Select **None**.
    * Leave everything else on its defaults.
5. Skip **Management**, **Advanced**, and **Tags** unless you wish to use them, then select **Review + create**. Once validation is finished, select **Create**.
6. Once deployment is finished, select **Go to resource**.
7. Select the **Networking** blade on the left, then **Network interface** in blue text that was generated with the VM.
8. Select the **IP configurations** blade on the left, then **ipconfig1**.
9. Under **Private IP address settings**, change the **Assignment** to **Static**. Note the IP address of the VM. Select **Save**.
10. Wait for the changes to save, then select your **Virtual network/subnet** in blue text.
11. We will now enter the DC VM as our DNS server for the Virtual Network. Select the **DNS servers** blade on the left.
12. Select the **Custom** radio dial, then enter the IP address you noted of the VM we just created. Also enter **8.8.8.8** (Google's DNS) so that our DC has internet access. Select **Save**. *Ex: 10.0.0.4*
13. Once the changes are saved, select the **Subnets** blade on the left.
14. Select **+ Gateway subnet** and enter **10.0.1.0/24** as the **Address range**. Leave everything else on its defaults. Select **OK**.

Next, we set up our VPN by utilizing Azure's Virtual Network Gateway (VNG):
15. Using the search bar, go to the Virtual Network Gateways overview page.
16. Select **Add**.
17. Fill out the requisite fields under the **Basics** tab - leave unmentioned fields by their defaults:
    * **Subscription**: Enter the subscription you'd like the VNG to be in.
    * **Resource group**: This will be filled in automatically once you select your VNET.
    * **Name**: Enter a name for your VNG.
    * **Region**: Enter the region you'd like your VNG to be in.
    * **SKU**: Select **Basic** unless another SKU is necessary.
    * **Virtual network**: Select the Virtual Network you previously created.
    * **Subnet**: Select the **Gateway Subnet** we previously created - likely 10.0.1.0/24.
    * **Public IP address name**: Enter a name for the Public IP address.
    * Select **Review + create**. Once validation is finished, select **Create**. Wait for the deployment to finish. This process will take around 30 minutes.
18. Once deployment is finished, select **Go to resource** then the **Point-to-site configuration** blade.
19. Select **Configure now**.
20. For the **Address Pool**, enter any private internet range (i.e. 172.16.0.0/24) that's not in use with the VNET (like 10.0.0.0 - 10.0.255.255).
21. We must now use PowerShell to generate Root and Client certificates for connecting to the VPN and then to our DC.

        


## 4. Azure Portal - Host Pool, App Group, and User Setup and Deployment
With the Azure Spring 2020 update, host pool, application group, and user management can be done through the Azure Portal. This section will cover creating host pools, workspaces, application groups, and also assigning users. 

### Deploying a Host Pool
Host pools are a collection of identically sized virtual machines that will be the layer under Windows Virtual Desktops. Later, app groups will be assigned to host pools to manage application offerings to end users.

To create a host pool (using the Azure Portal):
1. Within the Azure Portal, use the search bar to go to the Windows Virtual Desktop overview page.
2. Select **Create a host pool**.
3. Fill out the requisite fields under the **basics** tab:
   * **Subscription**: Enter the subscription you'd like the host pool to be in. *Ex: SAA-AWU-POC-1*
   * **Resource Group**: Enter the resource group you'd like the host pool to be in. *Ex: RG-SPOC-AWU-01*
   * **Host Pool Name**: Enter the name you'd like to assign to the host pool. Ex: *HPL-SAA-POC-AWU-1*
   * **Location**: Enter the location you'd like the metadata of the host pool to be stored in. *Ex: West US*
   * **Host pool type**: Select either **Personal** or **Pooled**. Personal host pool type will only allow one user to each VM, i.e a 1-to-1 ratio. Pooled, which will likely be the choice in most scenarios, will allow multiple users to each VM.
   * If you selected **Pooled**, fill out:
       * **Max session limit**: Enter the maximum amount of users allowed per VM.
       * **Load balancing algorithm**: Select either **Breadth-first** or **Depth-first**. Breadth-first logic will attempt to distribute users in a way that spreads them out as much as possible between VMs in a host pool, which Depth-first logic will attempt to distribute users in a way so that available VMs already under use will be "filled up" with connections before the next VM is used for additional connections.
    * If you selected **Personal**, fill out:
        * **Assignment type**: Select either **Automatic** or **Direct**. Automatic assignment allows the WVD service to pick the VM for the user on connection, while direct assignment requires that admins assign a specific VM for a user.
4. Fill out the requisite fields under the **Virtual Machines** tab:
    * **Add virtual machines**: Select **No** if you already have virtual machines you'd like to use for the host pool or would like to create them later. Select **Yes** if you'd like to create virtual machines as part of the host pool creation process now.
    * If you chose **Yes**, fill out:
        * **Resource group**: Enter the resource group you'd like the service to be in. *Ex: RG-SPOC-AWU-01*
        * **Virtual machine location**: Enter the location you'd like the virtual machines in the host pool to be in. *Ex: West US*
        * **Virtual machine size**: Enter the size of the virtual machines. *Ex: Standard B2s*
        * **Number of VMs**: Enter the number of VMs you'd like to provision for this host pool.
        * **Name prefix**: Enter a descriptive name to be the prefix of the provisioned VMs. *Ex: VM-WVD*
        * **Image type**:
        * If you chose **Gallery**, fill out:
            * **Image**: Select the OS for the VMs. *Ex: Windows 10 Enterprise multi-session, Version 1909*
        * If you chose **Storage blob**, fill out:
            * **Image URI**: Enter the storage blob URI to the image/snapshot/etc. you'd like to utilize for the VMs.
        * **Virtual network**: Enter the Virtual Network created before. *Ex: VNET-SPOC-AWU-01*
        * **Subnet**: Select the default subnet. *Ex: Range: 10.0.0.0/24*
        * **Public IP**: Select **No**.
        * **Network security group**: Select **Basic**.
        * **Public inbound ports**: Select **No**.
        * **Specify domain or unit**: Select **No**.
        * **AD domain join UPN**: Enter the account with domain join permissions here - if you used AADDS, the account must be an Azure AD account that's been synchronized and is also part of the **AAD DC Administrators** group. Remember that the password must have been reset for the account to have been synchronized. *Ex: john.doe@spuspoc.onmicrosoft.com*
        * **Password**: Enter the password.
        * **Confirm password**: Enter the password again.
5. Fill out the reequisite fields under the **Workspace** tab:
    * **Register desktop app group**: Select **Yes**.
    * **To this workspace**: Select **Create new** to create a new workspace, then enter a workspace name. A workspace is just a grouping to house application groups. *Ex: Default-Workspace*
6. Select **Review + create**. Once validation is finished, select **Create**. Wait for the deployment to finish.

*Optional*: The hostpool can instead be deployed using the ARM template [here](https://www.yammer.com/saasplaza.com/uploaded_files/524419563520) (from the SaaSplaza yammer.)

After downloading the above file and extracting its contents, follow these steps to deploy the template:
1. From the Azure Portal home page, select **Create a resource**.
2. Using the search bar, enter **template**. Select **Template deployment**.
3. Select **Create**, then **Build your own template in the editor**.
4. Select **Load file** then upload template.json downloaded from before. Select **Save**.
5. Fill out the parameters - note that the parameters.json file from the downloaded zip can be used as a guide here - parameters which are required have a * next to them - notable parameters:
* **Administrator Account Username**: Enter the admin account which has domain join privilege. If you used AADDS, this is the account that is assigned to the AAAD DC Administrators group.
* **Existing VNET Name**: Make sure you use the VNET we created previously here.
6. Once the parameters are filled out, check the terms and conditions acceptance box and select **Purchase**.

### Setting up Application Groups and Users
Note that the host pool creation process above by default makes a Desktop Application Group (for desktop sessions). You can add a Remote Application Group to allow users access to streamed software to the host pool using the Azure Portal. First, however, we will go over how to add users to your application groups:
1. At the Windows Virtual Desktop overview page, select the **Users** blade.
2. Use the search bar to look up a user or user group you'd like to add, and select them.
3. Switch to the **Assignments** tab, then select **Add**.
4. Check the boxes of the application groups you'd like the user to have access to, then select **Add**.

To create a Remote Application Group:
1. At the Windows Virtual Desktop overview page, select the **Application groups** blade.
2. Select **Add**, then fill out the requisite fields under **Basics**:
    * **Subscription**: Enter the subscription you'd like the application group to be in. *Ex: SAA-AWU-POC-1*
    * **Resource group**: Enter the resource group you'd like the application group to be in. *Ex: RG-SPOC-AWU-01*
    * **Host pool**: Select the host pool you'd like the application group to be in (the one we just created.) *Ex: HPL-SAA-POC-AWU-1*
    * **Application group type**: Select **RemoteApp**. Notice that the **Desktop** option is greyed out as there already exists a Desktop Application Group on this hostpool.
    * **Application group name**: Enter a name for your remote application group. *Ex: Remote Applications*
3. Switch to the **Assignments** tab:
    * Select **Add Azure AD users or user groups** to give access to this application group to the users or groups of your choice. Use the search bar to look up users, select them, then confirm by clicking **Select**.
4. Switch to the **Applications** tab:
    * Select **Add applications** to give users access to streamed software like Microsoft Paint. Give it a display name and description if you wish.
    * Select **Save** to confirm your choice.
5. Switch to the **Workspace** tab:
    * **Register application group**: Select the **Yes** radio dial unless you want a new Workspace to be created.
    * **Register application group**: Select a workspace to put the application group in. This preference seems to be mislabeled - it should likely be saying Application group workspace.
6. Select **Review + create**. Once validation is finished, select **Create**. Wait for the deployment to finish.

### Accessing Windows Virtual Desktop
Now that you have set up WVD and added some users to an application group, they can access the WVD using this [link](https://aka.ms/wvdarmweb).


## F.A.Q.
***What are the differences between WVD and RDS?***

When using RDS, a choice had be made between multi-session or single-session usage. Multi-session allowed users to share a VM and therefore save on costs, but made saving personal data was complicated or slow because of required synchronization. Single-session tied VMs to users at a 1-to-1 ratio, allowing personal data, but also potentially incurring greater cost. WVDs, however, allow multi-session usage with personal data through FSLogix, thus saving on costs and allowing easier access to personal storage.

***How many users can be on one VM at once?***

Theoretically, any amount of users can be logged into one VM on WVD. However, in practice, users will have to suffer more and more performance degredation as more people join a particular VM. One can scale the host pool to prevent this scenario by resizing the VMs in the host pool or by increasing the number of instances of VMs are in the host pool. It's recommended that at least 1 Core and 2 GB of RAM is provisioned per user.

***Does a VPN not have to be configured when using AADDS?***

## Tentative To-Do List
* Add pictures/animated GIFs
* Expand ARM template parameters section
* Expand remote app section to include post-creation application management
* Add a section for configuring friendly names for Workstations and app groups.
* Add a section for making user groups in Azure AD
* Expand hostpool setup for OU for on-prem solution
* Expand FAQ
* Add references and additional resources
* Section for future configuration
    * Scaling host pool
* Dynamic scaling
* Automated downtime
