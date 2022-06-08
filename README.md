# Deployment Guide - Azure Resources

## 1. New subscription checklist

### 1.1 Enable Microsoft Defender for Cloud

>Navigate to Microsoft Defender for Cloud and enable it for new subscription.

### 1.2 Enable encryption at host

>**Note**: this will be automatically enabled during jump-server VM deployment.
>
>```bash
>az provider register --namespace Microsoft.Compute
>az feature register --namespace Microsoft.Compute --name EncryptionAtHost
>```

## 2. Create Azure Resources

### 2.1 Internal windows machine
>
> 1. Create internal windows machine (with WSL) that will be used to access jump-server
> 2. Install **VS Code** with **Remote - SSH** extension on your windows machine

### 2.2 Prerequisites

>>**Notes:**
>>
>>- You will need your windows machine created in previous step.
>>
>>- When using WSL don't extract package to folder mounted from windows. Ideally use your home (~) folder.
>>
>>- **We recommend to use VS Code to connect to WSL or your linux machine. You will have available all VS Code tools that way.**
>>
>1. Copy **dw-infrastructure.tgz** to your home (~) folder
>2. Unzip the dw-infrastructure.tgz package
>
>    ```bash
>    cd ~
>    mkdir dw-infrastructure
>    tar -xzvf dw-infrastructure.tgz -C dw-infrastructure
>    cd dw-infrastructure/
>    ```
>
>3. Install the prerequisites (Azure CLI, Azure CLI AKS extension, Python)
>
>    ```bash
>    ./deploy-infrastructure-prerequisites.sh
>    direnv allow
>    ```

### 2.3 Azure Container Registry (Only when deploying first region)

> **Note:** you can skip this step, ACR already exists.
>
>1. Sign in with Azure CLI and set subscription. You can find the Tenant ID in the Azure Active Directory blade in the Azure Portal.
>
>    ```bash
>    az login --tenant {azure_tenant_id} --allow-no-subscriptions --use-device-code
>    az account set -s {subscription_name}
>    ```
>
>1. Create the DW resource group. Use location values from this command: az account list-locations --output=table.
>
>    ```bash
>    az group create -l {location} -n dw
>    ```
>
>1. Create the Azure Container Registry. Parameter 'name' must conform to the following pattern: '^[a-zA-Z0-9]*$'.
>
>    ```bash
>    az acr create --name {name} --resource-group dw --sku basic
>    ```
>
>1. After the Azure Container Registry is created, push images to the registry (kube/images/push_images_to_production_repository.py).

### 2.4 DW region Azure resources (Per region)

> **Note**:  Virtual machine size for larger clusters    - E2as v5
><https://azure.microsoft.com/en-us/pricing/details/virtual-machines/linux/>
>
>
>1. Check if you are logged into azure cli
>
>    ```bash
>    # outputs your current context
>    az account show
>
>    # if previous info is incorrect or you are not logged in
>    az logout
>    az login --tenant {azure_tenant_id} --allow-no-subscriptions --use-device-code
>    az account set -s {subscription_name}
>    ```
>
>1. Pick an AZ region
>
>     - Pick region from list (value will be from column 'Name' -> output of next command)
>     - For regions with availability zones support see <https://docs.microsoft.com/en-us/azure/availability-zones/az-overview>
>
>       ```bash
>       az account list-locations --output=table
>       ```
>
>     - For list of available vm sizes run command:
>
>       ```bash
>       az vm list-sizes --location {region} --output=table
>       ```
>
>     - To see vm size availability in zones run:
>
>       ```bash
>       az vm list-skus --all --location {region} --resource-type virtualMachines --size {vm-size} --output table
>       ```
>
>1. Enter the correct AZ region to settings.json
>
>    ```bash
>    code settings.json # if using VS Code, or any other preferred editor
>
>    {
>     "region": "", # azure  region
>     "aks_node_vm_size": "Standard_D2as_v5", # (Dasv5-series), in large cluster we would want `E2as v5` (Easv5-series)
>     "kubernetes_version": "1.21.9",
>     "aks_pools_subnet": "aks_pools_subnet",
>     "aks_pools_subnet_cidr": "10.1.0.0/16",
>     "lb_public_ip_dns_name": "r2lb" # optional, can be left empty, availability will be validated
>    }
>    ```
>
>1. ***Optional***: Edit the default-settings.json configuration file.
>
>    ```bash
>    code default-settings.json # if using VS Code, or any other preferred editor
>    {
>        "lb_public_ip": "lb_public_ip",
>        "aks": "aks",
>        "resource_group_prefix": "dw",
>        "region_resource_group": "{resource_group_prefix}-{region}",
>        "jump_server": "jump-server",
>        "jump_server_vm_size": "Standard_B1ms",
>        "nsg": "nsg",
>        "grafana_waf_policy": "grafana",
>        "aks_node_count": 3,
>        "vnet": "vnet",
>        "vnet_cidr": "10.0.0.0/8",
>        "vm_subnet": "vm-subnet",
>        "vm_subnet_cidr": "10.2.0.0/16",
>        "jump_server_ip": "10.2.0.10",
>        "b2c_resource_group": "{resource_group_prefix}-{region}-b2c",
>        "aks_resource_group": "{resource_group_prefix}-{region}-aks-1",
>        "aks_resource_group_generated": "{resource_group_prefix}-{region}-aks-1-generated",
>        "storage_account": "r2{region}",
>        "availability_zone": null, # null if not using zones otherwise use 1 for primary deployment, use 2 for disaster recovery deployment
>        "nodepool_name": "system"
>    }
>    ```
>
>
>1. Create ssh key for jump-server
>    - Create folder `~/.ssh/jump-server`
>      - Copy the `id_rsa.pub` key to the `~/.ssh/jump-server` folder. This key will be installed on jump-server VM.
>      - Copy the `id_rsa` key to the `~/.ssh/jump-server` folder. This key will be used to connect to jump-server VM.
>    - Run the deploy.py script.
>
>      ```bash
>      cd ~/dw-infrastructure/
>      ./deploy.py
>      ```
>
>1. Create DNS entry for jump-server pointing to jump-server public IP (can be find out on jump-server overview in azure portal)
>
>    ```bash
>    {region}-jump.2ring.cloud
>    ```
>
> 1. Enable the Azure just-in-time access.
> 1. In the Azure portal, open the Microsoft Defender for Cloud blade and click the Upgrade button.
> 1. Open the jump-server blade, choose Configuration and then click the Enable Just-in-time button.
> 1. In the jump-server blade, click the Connect button, choose SSH, from Source IP select My IP and request the just-in-time access (default
>3 hours).
>
>    ![just_in_time_access.png](./img/just_in_time_access.png)
>

### 2.5 Jump-server

> **Note:** You can use VS Code and configure your connection using ssh extension.
>
>1. Connect to the jump-server
>
>    ```bash
>    ssh -i ~/.ssh/jump-server/id_rsa deploy@{jump_server_dns_name}
>    ```
>
>2. Generate new SSH key pairs that will be used as a deploy keys
>
>    ```bash
>    mkdir ~/.ssh/dw-aks
>    mkdir ~/.ssh/dw-aks-infrastructure
>
>    ssh-keygen -f ~/.ssh/dw-aks/id_rsa -N ""
>    ssh-keygen -f ~/.ssh/dw-aks-infrastructure/id_rsa -N ""
>
>    cat ~/.ssh/dw-aks/id_rsa.pub
>    cat ~/.ssh/dw-aks-infrastructure/id_rsa.pub
>    ```
>
>3. Upload dw-aks-infrastructure key
>      - Open this URL: <https://github.com/2Ring/dw-aks-infrastructure>
>      - Settings -> Security -> Deploy keys
>      - Create new Deploy key
>        - Title: subscription name (e.g. dw-us-east)
>        - Key: Use content of this file: ~/.ssh/dw-aks-infrastructure/id_rsa.pub
>        - Allow write access: true
>4. Upload dw-aks key
>     - Open this URL: <https://github.com/2Ring/dw-aks>
>     - Settings -> Security -> Deploy keys
>     - Create new Deploy key
>       - Title: subscription name (e.g. dw-us-east)
>       - Key: Use content of this file: ~/.ssh/dw-aks/id_rsa.pub
>       - Allow write access: true
>
>5. Clone the dw-aks and dw-aks-infrastructure repositories
>
>     ```bash
>     sudo mkdir /srv/spring22
>     sudo mkdir /srv/infrastructure
>
>     sudo chmod -R g+rw+s /srv/spring22
>     sudo chmod -R g+rw+s /srv/infrastructure
>
>     sudo chown deploy:deploy /srv/spring22
>     sudo chown deploy:deploy /srv/infrastructure
>
>     git config --global --add advice.detachedHead false
>     git config --global user.name "Your Name"
>     git config --global user.email you@example.com
>
>     GIT_SSH_COMMAND='ssh -i ~/.ssh/dw-aks/id_rsa -o IdentitiesOnly=yes' git clone -q -b spring22 git@github.com:2Ring/dw-aks.git /srv/spring22
>
>     GIT_SSH_COMMAND='ssh -i ~/.ssh/dw-aks-infrastructure/id_rsa -o IdentitiesOnly=yes' git clone -q -b spring22 git@github.com:2Ring/dw-aks-infrastructure.git /srv/infrastructure
>     ```
>
>
>6. Install the prerequisites. **During installation you will be prompted to log in to your azure subscription**.
>
>    ```bash
>    cd /srv/infrastructure/jump-server-scripts
>    ./jump-server-prerequisites.sh {azure_tenant_id} {subscription_id} aks {aks_resource_group}
>    ```
>
>
>7. **Important:** Now you need to logout to refresh your bash console to apply changes made to .bashrc
>
>    ```bash
>    exit
>    ssh -i ~/.ssh/jump-server/id_rsa deploy@{jump_server_dns_name}
>    ```
>
>
>8. Create new branch, push local branches to origin
>
>    ```bash
>    cd /srv/infrastructure
>    git checkout -b {subscription name}
>    git init --shared=group
>
>    git push -u origin {subscription name}
>    direnv allow .
>
>    cd /srv/spring22
>    git checkout -b {subscription name}
>    git init --shared=group
>
>    git push -u origin {subscription name}
>    direnv allow .
>    ```
>
>
>9. Create ubuntu and kubernetes users
>
>    **Notes:**
>    - Using the --kubernetes-user flag, the script will create a ubuntu user as well as a kubernetes user. Without the flag, script will create only ubuntu user.
>    - Username convention is *bielikm, sivakp, moravekm*
>
>    ```bash
>    cd /srv/infrastructure/jump-server-scripts
>    ./create_user.py --username {name} --kubernetes-user
>
>    # script will output password which needs to be saved to safe place
>
>    # this newly created user is automatically part of sudo and deploy groups
>    # script will also create ~/.ssh/authorized_keys for this user with correct permissions
>
>    # allow ssh connect
>    sudo nano /home/{username}/.ssh/authorized_keys
>    # paste user id_rsa.pub here and save
>    ```
>
>10. **Now you can switch to your newly created user and continue installation with this user.**
>
>     **Note:** If you switched to new user you need to login to `Azure CLI`, set subscription again and add an exception for the installation directory.
>
>     ```bash
>     az login --tenant {azure_tenant_id} --allow-no-subscriptions --use-device-code
>     az account set -s {subscription_name}
>
>     git config --global --add safe.directory /srv/spring22
>     git config --global --add safe.directory /srv/infrastructure
>
>     git config --global user.email "you@example.com"
>     git config --global user.name "Your Name"
>
>     direnv allow /srv/spring22
>     direnv allow /srv/infrastructure
>     ```
>

### 2.6 Authenticate AKS with an Azure container registry
>
>1. Navigate to the Azure portal and open Azure container registry resource.
>2. Open the Access control (IAM) blade, click add and select Add role assignment.
>
>    ![acr_aim.png](./img/acr_iam.png)
>3. In Role tab choose AcrPull.
>
>    ![add_role_assignment_role_tab.png](./img/add_role_assignment_role_tab.png)
>4. In Members tab choose Managed Identity and Select members.
>5. In Select managed identities choose subscription where the aks is created, User-assigned managed identity and select aks-agentpool.
>
>    ![add_role_assignment_members_tab.png](./img/add_role_assignment_members_tab.png)
>
>6. Configure ACR integration for existing AKS clusters \
>  Attach ACR
>
>    ```bash
>    az aks update -n aks -g {resource_group_name} --attach-acr tworing.azurecr.io
>    ```
