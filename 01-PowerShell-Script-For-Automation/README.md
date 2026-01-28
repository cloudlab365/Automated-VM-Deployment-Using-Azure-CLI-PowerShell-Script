# PowerShell Script for Automated VM Deployment

Automating the deployment of an Azure virtual network and virtual machine with PowerShell ensures a consistent, repeatable setup every time.
This approach reduces manual configuration errors, speeds up provisioning, and makes it easy to recreate environments for testing, learning, or production use.
PowerShell scripts can be version‑controlled, reused, and integrated into automation pipelines—making your infrastructure both more reliable and easier to manage as it grows.



> gif goes here


## 📦 Installation

#Run the below script to automate VM creation and Virtual Network using PowerShell

Save as deploy-azure-vm.ps1 and run from PowerShell 7+).

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

### 1. Uses Azure CLI (az) under the hood 

```bash
<#
.SYNOPSIS
Provision a VNet and a Windows VM (Standard_B1s) in UK South using Azure CLI (no Python).
#>

param(
    [string]$ResourceGroupName = "rg-uksouth-winvm-demo",
    [string]$Location = "uksouth",
    [string]$VnetName = "vnet-uksouth-demo",
    [string]$VnetCidr = "10.10.0.0/16",
    [string]$SubnetName = "snet-workloads",
    [string]$SubnetCidr = "10.10.1.0/24",
    [string]$NsgName = "nsg-workloads",
    [string]$PipName = "pip-winvm-demo",
    [string]$NicName = "nic-winvm-demo",
    [string]$VmName = "winvm-demo",
    [string]$VmSize = "Standard_B1s",
    [string]$AdminUser = "azureuser",
    [securestring]$AdminPasswordSecure
)

function Ensure-AzCli {
    if (-not (Get-Command az -ErrorAction SilentlyContinue)) {
        throw "Azure CLI (az) not found in PATH. Install from https://aka.ms/azure-cli"
    }
    try { $null = az account show --only-show-errors | Out-Null }
    catch {
        Write-Host "You are not logged in. Opening browser..." -ForegroundColor Yellow
        az login | Out-Null
    }
}

function New-RandomPassword {
    param([int]$Length = 24)
    $chars = ('a'..'z') + ('A'..'Z') + ('0'..'9') + @('!','@','#','$','%','^','&','*','(',')','-','_','=','+')
    -join (1..$Length | ForEach-Object { $chars | Get-Random })
}

# MAIN
Ensure-AzCli

if (-not $AdminPasswordSecure) {
    Write-Host "No admin password provided. Enter a strong password (min 12 chars w/ complexity):" -ForegroundColor Yellow
    $AdminPasswordSecure = Read-Host -AsSecureString "Admin password for $AdminUser"
}
$plain = [Runtime.InteropServices.Marshal]::PtrToStringAuto(
    [Runtime.InteropServices.Marshal]::SecureStringToBSTR($AdminPasswordSecure)
)

Write-Host "Using admin username: $AdminUser" -ForegroundColor Cyan

# 1) Resource Group
az group create `
  --name $ResourceGroupName `
  --location $Location `
  --only-show-errors | Out-Null

# 2) VNet + Subnet
az network vnet create `
  --resource-group $ResourceGroupName `
  --name $VnetName `
  --address-prefixes $VnetCidr `
  --subnet-name $SubnetName `
  --subnet-prefixes $SubnetCidr `
  --only-show-errors | Out-Null

# 3) NSG and RDP rule (restrict to current public IP)
try {
    $myIp = (Invoke-RestMethod -Uri "https://ifconfig.me").Trim()
    if (-not $myIp) { throw "Empty IP" }
} catch {
    Write-Host "Could not auto-detect public IP. Defaulting to 0.0.0.0/0 (NOT RECOMMENDED)" -ForegroundColor Yellow
    $myIp = "0.0.0.0"
}

az network nsg create `
  --resource-group $ResourceGroupName `
  --name $NsgName `
  --only-show-errors | Out-Null

$sourcePrefix = ($myIp -eq "0.0.0.0") ? "0.0.0.0/0" : "$myIp/32"

az network nsg rule create `
  --resource-group $ResourceGroupName `
  --nsg-name $NsgName `
  --name "Allow-RDP-From-MyIP" `
  --priority 1000 `
  --access Allow `
  --direction Inbound `
  --protocol Tcp `
  --source-address-prefixes $sourcePrefix `
  --source-port-ranges "*" `
  --destination-address-prefixes "*" `
  --destination-port-ranges 3389 `
  --only-show-errors | Out-Null

# 4) Associate NSG to Subnet
az network vnet subnet update `
  --resource-group $ResourceGroupName `
  --vnet-name $VnetName `
  --name $SubnetName `
  --network-security-group $NsgName `
  --only-show-errors | Out-Null

# 5) Public IP
az network public-ip create `
  --resource-group $ResourceGroupName `
  --name $PipName `
  --location $Location `
  --sku Standard `
  --allocation-method Static `
  --only-show-errors | Out-Null

# 6) NIC
$subnetId = az network vnet subnet show `
  --resource-group $ResourceGroupName `
  --vnet-name $VnetName `
  --name $SubnetName `
  --query id -o tsv

$pipId = az network public-ip show `
  --resource-group $ResourceGroupName `
  --name $PipName `
  --query id -o tsv

az network nic create `
  --resource-group $ResourceGroupName `
  --name $NicName `
  --subnet $subnetId `
  --public-ip-address $pipId `
  --only-show-errors | Out-Null

# 7) VM
az vm create `
  --resource-group $ResourceGroupName `
  --name $VmName `
  --location $Location `
  --nics $NicName `
  --image "MicrosoftWindowsServer:WindowsServer:2022-datacenter:latest" `
  --size $VmSize `
  --admin-username $AdminUser `
  --admin-password $plain `
  --authentication-type password `
  --os-disk-name "${VmName}-osdisk" `
  --tags "env=demo" "owner=$env:USERNAME" `
  --only-show-errors | Out-Null

# 8) Output connection info
$publicIp = az vm list-ip-addresses -g $ResourceGroupName -n $VmName --query "[0].virtualMachine.network.publicIpAddresses[0].ipAddress" -o tsv
Write-Host "`nDeployment complete." -ForegroundColor Green
Write-Host "VM Name       : $VmName"
Write-Host "Resource Group: $ResourceGroupName"
Write-Host "Region        : $Location"
Write-Host "Size          : $VmSize"
Write-Host "Public IP     : $publicIp"
Write-Host "Username      : $AdminUser"
Write-Host "Password      : (hidden)"
Write-Host "`nConnect with: mstsc /v:$publicIp" -ForegroundColor Cyan
``
```

### 2. Run it:

```bash
# You will be prompted securely for the admin password
.\deploy-azure-vm.ps1
```

### 3. Cleanup (avoid charges)

```bash
az group delete --name rg-uksouth-winvm-demo --yes --no-wait
```

## Quick Security Tips

- Prefer Azure Bastion in production to avoid exposing RDP on a public IP.
- Restrict RDP to your IP (already shown) or remove the rule entirely when done.
- Store credentials in Azure Key Vault and pull them at runtime if needed.


