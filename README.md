# Automated VM Deployment Using Azure CLI + Bash/PowerShell Script

Automating VM deployments with Azure CLI and Bash/PowerShell brings consistency, speed, and reliability to cloud provisioning. Instead of manually creating VMs through the Azure Portal—an error‑prone and time‑consuming process—scripts allow you to deploy fully configured environments in seconds. This approach ensures every VM is built the same way, supports repeatable infrastructure for dev/test/production, and enables version control through GitHub. Automated scripts are also ideal for CI/CD pipelines, disaster recovery scenarios, and large-scale infrastructure rollouts, making your Azure environment more predictable, secure, and maintainable.

Each stage includes detailed documentation, example scripts, and diagrams to make it easy to understand or reproduce in your own Azure subscription.

---

## What This Project Covers

- **Virtual Machines** – Automated creation of Azure Virtual Machines using Azure CLI 
- **Networking** – Configuration of networking components (VNet, subnet, NSG rules, public/private IPs) 
- **Identity & Governance** – Tagging, naming standards, and resource‑group structure for clean Azure governance  
- **Automation** – PowerShell scripting  

---

## Project Goal

The goal of this project is to provide a simple, repeatable, and fully automated way to deploy Azure Virtual Machines using Azure CLI with Bash or PowerShell scripting. By standardizing VM creation through automation.
The size Standard_B1s is eligible under the Azure Free Account offer for compute (750 hrs/month), but Windows Server licensing may still incur charges. Linux on B1s is typically the truly free option. Double‑check your subscription benefits and estimated cost before deploying.

---

> gif goes here


## 📦 Installation

#Prerequisites

Azure CLI installed and logged in:
(PowerShell 7+ (for the script portion), and Contributor rights to your target subscription.)

```bash
az login
az account set --subscription "<your-subscription-id-or-name>"
```

Variables
```bash
# Core variables
RG="rg-uksouth-winvm-demo"
LOCATION="uksouth"
VNET_NAME="vnet-uksouth-demo"
VNET_CIDR="10.10.0.0/16"
SUBNET_NAME="snet-workloads"
SUBNET_CIDR="10.10.1.0/24"
NSG_NAME="nsg-workloads"
PIP_NAME="pip-winvm-demo"
NIC_NAME="nic-winvm-demo"
VM_NAME="winvm-demo"
VM_SIZE="Standard_B1s"
ADMIN_USER="azureuser"
```

### Prompt for a password (recommended)

```bash
read -s -p "Enter a strong password for $ADMIN_USER: " ADMIN_PASS; echo
```

### 1. Resource Group

```bash
az group create \
  --name $RG \
  --location $LOCATION
```

### 2. Virtual Network + Subnet

```bash
az network vnet create \
  --resource-group $RG \
  --name $VNET_NAME \
  --address-prefixes $VNET_CIDR \
  --subnet-name $SUBNET_NAME \
  --subnet-prefixes $SUBNET_CIDR
``
```

### 3. NSG and RDP rule (restrict to your IP)

```bash
MYIP=$(curl -s https://ifconfig.me || curl -s https://api.ipify.org)
az network nsg create \
  --resource-group $RG \
  --name $NSG_NAME

az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_NAME \
  --name "Allow-RDP-From-MyIP" \
  --priority 1000 \
  --access Allow \
  --direction Inbound \
  --protocol Tcp \
  --source-address-prefixes ${MYIP}/32 \
  --source-port-ranges "*" \
  --destination-address-prefixes "*" \
  --destination-port-ranges 3389
```

### 4. Associate NSG to Subnet

```bash
az network vnet subnet update \
  --resource-group $RG \
  --vnet-name $VNET_NAME \
  --name $SUBNET_NAME \
  --network-security-group $NSG_NAME
```

### 5. Public IP (Standard, Static)

```bash
az network public-ip create \
  --resource-group $RG \
  --name $PIP_NAME \
  --location $LOCATION \
  --sku Standard \
  --allocation-method Static
``
```

### 6. NIC

```bash
SUBNET_ID=$(az network vnet subnet show \
  --resource-group $RG \
  --vnet-name $VNET_NAME \
  --name $SUBNET_NAME \
  --query id -o tsv)

PIP_ID=$(az network public-ip show \
  --resource-group $RG \
  --name $PIP_NAME \
  --query id -o tsv)

az network nic create \
  --resource-group $RG \
  --name $NIC_NAME \
  --subnet $SUBNET_ID \
  --public-ip-address $PIP_ID
``
```

### 7. Windows VM (Windows Server 2022)

```bash
az vm create \
  --resource-group $RG \
  --name $VM_NAME \
  --location $LOCATION \
  --nics $NIC_NAME \
  --image "MicrosoftWindowsServer:WindowsServer:2022-datacenter:latest" \
  --size $VM_SIZE \
  --admin-username $ADMIN_USER \
  --admin-password "$ADMIN_PASS" \
  --authentication-type password \
  --os-disk-name "${VM_NAME}-osdisk" \
  --tags "env=demo" "owner=$USER"
```

### 8. Get Public IP and RDP

```bash
az vm list-ip-addresses -g $RG -n $VM_NAME -o table
# Connect with: mstsc /v:<public-ip>
```


