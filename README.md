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


