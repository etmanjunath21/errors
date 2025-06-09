# AVM VM Deployment - Complete ZIP Package

## Directory Structure for ZIP File

```
avm-vm-deployment.zip
├── examples/
│   ├── 01-prerequisites/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars.example
│   └── 02-vm-deployment/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── terraform.tfvars.example
├── README.md
└── deployment-guide.md
```

---

## File: examples/01-prerequisites/main.tf

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy    = true
      recover_soft_deleted_key_vaults = true
    }
  }
}

# Data source for current client configuration
data "azurerm_client_config" "current" {}

# Data source for existing resource group
data "azurerm_resource_group" "main" {
  name = var.resource_group_name
}

# Random password for VM admin
resource "random_password" "vm_admin_password" {
  length  = 16
  special = true
}

# Deploy Key Vault using Microsoft AVM module
module "key_vault" {
  source = "Azure/avm-res-keyvault-vault/azurerm"
  version = "~> 0.5"

  name                = var.key_vault_name
  location            = data.azurerm_resource_group.main.location
  resource_group_name = data.azurerm_resource_group.main.name
  tenant_id           = data.azurerm_client_config.current.tenant_id

  sku_name                        = var.key_vault_sku
  enabled_for_deployment          = true
  enabled_for_disk_encryption     = true
  enabled_for_template_deployment = true
  purge_protection_enabled        = true
  soft_delete_retention_days      = 7

  network_acls = {
    default_action = "Allow"
    bypass         = "AzureServices"
  }

  role_assignments = {
    current_user = {
      role_definition_id_or_name = "Key Vault Administrator"
      principal_id               = data.azurerm_client_config.current.object_id
    }
  }

  secrets = {
    vm_admin_password = {
      name  = "vm-admin-password"
      value = random_password.vm_admin_password.result
    }
  }

  tags = var.tags
}

# Deploy a single shared public IP using Microsoft AVM module
module "shared_public_ip" {
  source = "Azure/avm-res-network-publicipaddress/azurerm"
  version = "~> 0.1"

  name                = var.shared_public_ip_name
  location            = data.azurerm_resource_group.main.location
  resource_group_name = data.azurerm_resource_group.main.name
  
  allocation_method = "Static"
  sku              = "Standard"
  zones            = ["1", "2", "3"]  # Zone redundant for high availability

  tags = merge(var.tags, {
    Purpose = "Shared-Public-IP"
    Usage   = "Multiple-VMs"
  })
}
  source = "Azure/avm-res-network-virtualnetwork/azurerm"
  version = "~> 0.1"

  name                = var.vnet_name
  location            = data.azurerm_resource_group.main.location
  resource_group_name = data.azurerm_resource_group.main.name
  address_space       = var.vnet_address_space

  subnets = {
    vm_subnet = {
      name             = var.subnet_name
      address_prefixes = var.subnet_address_prefixes
    }
  }

  tags = var.tags
}

# Deploy Virtual Network using Microsoft AVM module
module "virtual_network" {
  source = "Azure/avm-res-network-networksecuritygroup/azurerm"
  version = "~> 0.2"

  name                = "${var.subnet_name}-nsg"
  location            = data.azurerm_resource_group.main.location
  resource_group_name = data.azurerm_resource_group.main.name

  security_rules = {
    allow_rdp = {
      access                     = "Allow"
      direction                  = "Inbound"
      name                       = "Allow-RDP-Restricted"
      priority                   = 1001
      protocol                   = "Tcp"
      source_address_prefix      = var.allowed_ip_range
      source_port_range          = "*"
      destination_address_prefix = "VirtualNetwork"
      destination_port_range     = "3389"
      description               = "Allow RDP from specified IP range only"
    }
    allow_winrm = {
      access                     = "Allow"
      direction                  = "Inbound"
      name                       = "Allow-WinRM-Restricted"
      priority                   = 1002
      protocol                   = "Tcp"
      source_address_prefix      = var.allowed_ip_range
      source_port_range          = "*"
      destination_address_prefix = "VirtualNetwork"
      destination_port_range     = "5985"
      description               = "Allow WinRM from specified IP range only"
    }
    deny_all_inbound = {
      access                     = "Deny"
      direction                  = "Inbound"
      name                       = "Deny-All-Other-Inbound"
      priority                   = 4000
      protocol                   = "*"
      source_address_prefix      = "*"
      source_port_range          = "*"
      destination_address_prefix = "*"
      destination_port_range     = "*"
      description               = "Deny all other inbound traffic"
    }
  }

  tags = var.tags
}

# Deploy Network Security Group using Microsoft AVM module
module "network_security_group" {
  subnet_id                 = module.virtual_network.subnets["vm_subnet"].resource_id
  network_security_group_id = module.network_security_group.resource_id
}
```

---

## File: examples/01-prerequisites/variables.tf

```hcl
# Associate NSG with subnet
resource "azurerm_subnet_network_security_group_association" "vm_subnet_nsg" {
  subnet_id                 = module.virtual_network.subnets["vm_subnet"].resource_id
  network_security_group_id = module.network_security_group.resource_id
}
  description = "Name of the existing resource group"
  type        = string
}

variable "key_vault_name" {
  description = "Name of the Key Vault"
  type        = string
}

variable "key_vault_sku" {
  description = "SKU name for Key Vault"
  type        = string
  default     = "standard"
}

variable "vnet_name" {
  description = "Name of the Virtual Network"
  type        = string
}

variable "vnet_address_space" {
  description = "Address space for the Virtual Network (limited for testing)"
  type        = list(string)
  default     = ["10.10.0.0/24"]  # Limited to 256 IPs for testing
}

variable "subnet_name" {
  description = "Name of the subnet"
  type        = string
}

variable "subnet_address_prefixes" {
  description = "Address prefixes for the subnet (limited for testing)"
  type        = list(string)
  default     = ["10.10.0.0/28"]  # Limited to 16 IPs for testing
}

variable "allowed_ip_range" {
  description = "IP range allowed for RDP/WinRM access (CHANGE THIS FOR SECURITY)"
  type        = string
  default     = "0.0.0.0/0"  # CHANGE TO YOUR PUBLIC IP FOR SECURITY
  validation {
    condition = can(cidrhost(var.allowed_ip_range, 0))
    error_message = "The allowed_ip_range must be a valid CIDR block (e.g., '203.0.113.0/32' for single IP)."
  }
}

variable "enable_public_ip" {
  description = "Enable public IP for the VM (set to false for private access only)"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default = {
    Environment = "lab"
    Project     = "avm-vm-deployment"
  }
}
```

---

## File: examples/01-prerequisites/outputs.tf

```hcl
output "resource_group_name" {
  description = "Name of the resource group"
  value       = data.azurerm_resource_group.main.name
}

output "resource_group_location" {
  description = "Location of the resource group"
  value       = data.azurerm_resource_group.main.location
}

output "key_vault_id" {
  description = "ID of the Key Vault"
  value       = module.key_vault.resource_id
}

output "key_vault_name" {
  description = "Name of the Key Vault"
  value       = module.key_vault.resource.name
}

output "key_vault_location" {
  description = "Location of the Key Vault"
  value       = module.key_vault.resource.location
}

output "vnet_id" {
  description = "ID of the Virtual Network"
  value       = module.virtual_network.resource_id
}

output "vnet_name" {
  description = "Name of the Virtual Network"
  value       = module.virtual_network.resource.name
}

output "subnet_id" {
  description = "ID of the VM subnet"
  value       = module.virtual_network.subnets["vm_subnet"].resource_id
}

output "subnet_name" {
  description = "Name of the VM subnet"
  value       = module.virtual_network.subnets["vm_subnet"].name
}

output "vm_admin_password_secret_name" {
  description = "Name of the secret containing VM admin password"
  value       = "vm-admin-password"
}
```

---

## File: examples/01-prerequisites/terraform.tfvars.example

```hcl
# Copy this file to terraform.tfvars and update with your values

resource_group_name = "rg-avm-lab-001"
key_vault_name      = "kv-avm-lab-001"
vnet_name          = "vnet-avm-lab-001"
subnet_name        = "snet-vm-001"

# Optional: Customize these values for testing environment
vnet_address_space       = ["10.10.0.0/24"]    # Limited to 256 IPs for testing
subnet_address_prefixes  = ["10.10.0.0/28"]    # Limited to 16 IPs for testing
allowed_ip_range         = "YOUR_PUBLIC_IP/32"  # IMPORTANT: Replace with your actual public IP

# To find your public IP, run: curl -s https://ipinfo.io/ip
# Example: allowed_ip_range = "203.0.113.45/32"

tags = {
  Environment = "lab"
  Project     = "avm-vm-deployment"
  Owner       = "your-name"
}
```

---

## File: examples/02-vm-deployment/main.tf

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# Data sources for prerequisites
data "azurerm_resource_group" "main" {
  name = var.resource_group_name
}

data "azurerm_key_vault" "main" {
  name                = var.key_vault_name
  resource_group_name = var.resource_group_name
}

data "azurerm_subnet" "vm_subnet" {
  name                 = var.subnet_name
  virtual_network_name = var.vnet_name
  resource_group_name  = var.resource_group_name
}

# Data source for shared public IP created in prerequisites
data "azurerm_public_ip" "shared" {
  name                = var.shared_public_ip_name
  resource_group_name = var.resource_group_name
}
  name         = "vm-admin-password"
  key_vault_id = data.azurerm_key_vault.main.id
}

data "azurerm_key_vault_secret" "vm_admin_password" {
  name         = "vm-admin-password"
  key_vault_id = data.azurerm_key_vault.main.id
}
  source = "Azure/avm-res-compute-virtualmachine/azurerm"
  version = "~> 0.15"

  name                = var.vm_name
  location            = data.azurerm_resource_group.main.location
  resource_group_name = data.azurerm_resource_group.main.name

  admin_username                     = var.admin_username
  disable_password_authentication    = false
  admin_password                     = data.azurerm_key_vault_secret.vm_admin_password.value
  os_type                           = "Windows"
  size                              = var.vm_size
  zone                              = var.vm_zone

  source_image_reference = var.source_image_reference

  os_disk = {
    caching                = var.os_disk_caching
    storage_account_type   = var.os_disk_storage_account_type
    disk_size_gb          = var.os_disk_size_gb
  }

  # Network interface configuration
  network_interfaces = {
    network_interface_1 = {
      name = "${var.vm_name}-nic"
      ip_configurations = {
        ip_configuration_1 = {
          name                          = "${var.vm_name}-ip-config"
          private_ip_subnet_resource_id = data.azurerm_subnet.vm_subnet.id
          create_public_ip_address      = var.enable_public_ip
          public_ip_address_name        = var.enable_public_ip ? "${var.vm_name}-pip" : null
          public_ip_sku                 = var.enable_public_ip ? "Standard" : null
          public_ip_allocation_method   = var.enable_public_ip ? "Static" : null
          is_primary_ipconfiguration    = true
        }
      }
    }
  }

  # Enable system-assigned managed identity
  managed_identities = {
    system_assigned = true
  }

  # Enable disk encryption if Key Vault is available
  encryption_at_host_enabled = true

  tags = var.tags
}
```

---

## File: examples/02-vm-deployment/variables.tf

```hcl
variable "resource_group_name" {
  description = "Name of the existing resource group"
  type        = string
}

variable "key_vault_name" {
  description = "Name of the Key Vault"
  type        = string
}

variable "vnet_name" {
  description = "Name of the Virtual Network"
  type        = string
}

variable "subnet_name" {
  description = "Name of the subnet"
  type        = string
}

variable "vm_name" {
  description = "Name of the virtual machine"
  type        = string
}

variable "vm_size" {
  description = "Size of the virtual machine"
  type        = string
  default     = "Standard_B2s"
}

variable "admin_username" {
  description = "Admin username for the VM"
  type        = string
  default     = "azureuser"
}

variable "vm_zone" {
  description = "Availability zone for the VM"
  type        = string
  default     = "1"
}

variable "os_disk_caching" {
  description = "OS disk caching type"
  type        = string
  default     = "ReadWrite"
}

variable "os_disk_storage_account_type" {
  description = "OS disk storage account type"
  type        = string
  default     = "Premium_LRS"
}

variable "os_disk_size_gb" {
  description = "OS disk size in GB"
  type        = number
  default     = 128
}

variable "source_image_reference" {
  description = "Source image reference"
  type = object({
    publisher = string
    offer     = string
    sku       = string
    version   = string
  })
  default = {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2022-Datacenter"
    version   = "latest"
  }
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default = {
    Environment = "lab"
    Project     = "avm-vm-deployment"
  }
}
```

---

## File: examples/02-vm-deployment/outputs.tf

```hcl
output "vm_id" {
  description = "ID of the virtual machine"
  value       = module.windows_vm.resource_id
}

output "vm_name" {
  description = "Name of the virtual machine"
  value       = module.windows_vm.resource.name
}

output "vm_private_ip" {
  description = "Private IP address of the virtual machine"
  value       = module.windows_vm.virtual_machine_azurerm.network_interface_ids
}

output "vm_public_ip" {
  description = "Public IP address of the virtual machine"
  value       = try(module.windows_vm.public_ip_addresses[0], null)
}
```

---

## File: examples/02-vm-deployment/terraform.tfvars.example

```hcl
# Copy this file to terraform.tfvars and update with your values

resource_group_name = "rg-avm-lab-001"
key_vault_name      = "kv-avm-lab-001"
vnet_name          = "vnet-avm-lab-001"
subnet_name        = "snet-vm-001"

vm_name        = "vm-avm-lab-001"
admin_username = "azureuser"

# Security Configuration for Testing
enable_public_ip = true                         # Set to false for private-only access
# allowed_ip_range = "YOUR_PUBLIC_IP/32"        # UNCOMMENT and set your IP in prerequisites
# allowed_ip_range = "YOUR_PUBLIC_IP/32"        # UNCOMMENT and set your IP in prerequisites

# Optional: Customize these values
# vm_size                        = "Standard_B2s"
# vm_zone                        = "1"
# os_disk_caching               = "ReadWrite"
# os_disk_storage_account_type  = "Premium_LRS"
# os_disk_size_gb               = 128

# For Windows Server 2019
# source_image_reference = {
#   publisher = "MicrosoftWindowsServer"
#   offer     = "WindowsServer"
#   sku       = "2019-Datacenter"
#   version   = "latest"
# }

tags = {
  Environment = "lab"
  Project     = "avm-vm-deployment"
  Owner       = "your-name"
  TestLimited = "true"
}
```

---

## File: README.md

```markdown
# AVM VM Deployment using Microsoft Azure Verified Modules

This repository contains Terraform code for deploying Windows VMs using Microsoft's official Azure Verified Modules (AVM) directly from GitHub.

## Structure

- `examples/01-prerequisites/` - Creates Key Vault, VNet, and Subnet using AVM modules
- `examples/02-vm-deployment/` - Deploys the VM using Microsoft AVM module

## Microsoft AVM Modules Used

- **Key Vault**: `Azure/avm-res-keyvault-vault/azurerm`
- **Virtual Network**: `Azure/avm-res-network-virtualnetwork/azurerm`
- **Network Security Group**: `Azure/avm-res-network-networksecuritygroup/azurerm`
- **Virtual Machine**: `Azure/avm-res-compute-virtualmachine/azurerm`

## Prerequisites

1. Azure CLI installed and configured
2. Terraform >= 1.5
3. Existing Resource Group in Azure

## Usage

### Step 1: Deploy Prerequisites

```bash
cd examples/01-prerequisites
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your values
terraform init
terraform plan
terraform apply
```

### Step 2: Deploy VM

```bash
cd ../02-vm-deployment
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your values
terraform init
terraform plan
terraform apply
```

## Benefits of Using Microsoft AVM Modules

- **Microsoft Maintained**: Official modules maintained by Microsoft
- **Best Practices**: Built-in security and compliance best practices
- **Regular Updates**: Automatically updated with latest Azure features
- **Community Tested**: Thoroughly tested by the Azure community
- **Standardized**: Consistent patterns across all Azure resources

## Features

- Customer-managed encryption keys (handled by AVM modules)
- Network security groups with configurable rules
- Public IP assignment
- Key Vault integration for secrets
- RBAC and managed identity support
- Compliance with Azure best practices

## Customization

Update the `terraform.tfvars` files in each example directory to match your environment and requirements. All configuration is done through module variables - no need to modify the module code.

## Module Documentation

For detailed documentation on each module, refer to:
- [AVM Key Vault Module](https://github.com/Azure/terraform-azurerm-avm-res-keyvault-vault)
- [AVM Virtual Network Module](https://github.com/Azure/terraform-azurerm-avm-res-network-virtualnetwork)
- [AVM NSG Module](https://github.com/Azure/terraform-azurerm-avm-res-network-networksecuritygroup)
- [AVM Virtual Machine Module](https://github.com/Azure/terraform-azurerm-avm-res-compute-virtualmachine)
```

---

## File: deployment-guide.md

```markdown
# AVM VM Deployment Guide

## Quick Start

### 1. Prerequisites Setup
- Ensure you have an existing Azure Resource Group
- Update `examples/01-prerequisites/terraform.tfvars` with your values
- Run the prerequisites deployment

### 2. VM Deployment
- Update `examples/02-vm-deployment/terraform.tfvars` with your values
- Deploy the VM using the prerequisites infrastructure

### 3. Access Your VM
- Use the public IP output from the VM deployment
- Connect via RDP using the admin username and password from Key Vault

## Important Notes for Testing

- **Limited Address Space**: Uses smaller CIDR blocks to limit IP consumption
  - VNet: `10.10.0.0/24` (256 IPs)
  - Subnet: `10.10.0.0/28` (16 IPs)
- **Security**: Must specify your public IP for RDP access
- **Public IP**: Can be disabled for private-only testing
- **VM Sizes**: Choose cost-effective sizes like `Standard_B2s` for testing

## Security Configuration

### Finding Your Public IP
```bash
# Command to find your public IP
curl -s https://ipinfo.io/ip

# Or visit: https://whatismyipaddress.com/
```

### Setting IP Restrictions
Update `terraform.tfvars` in prerequisites:
```hcl
allowed_ip_range = "YOUR_PUBLIC_IP/32"  # Replace with actual IP
```

### Private-Only Deployment
For testing without public access, set in VM deployment:
```hcl
enable_public_ip = false
```

## Troubleshooting

### Common Issues
1. **Key Vault name already exists**: Use a unique name
2. **Insufficient permissions**: Ensure you have Contributor access to the resource group
3. **Network connectivity**: Verify NSG rules and public IP configuration

### Getting Help
- Review Terraform plan output before applying
- Check Azure portal for resource status
- Review Terraform state files for current configuration
```

---

## Instructions for Creating ZIP File

1. Create the directory structure as shown above
2. Copy each file content into the respective files
3. Create a ZIP file named `avm-vm-deployment.zip`
4. Extract and follow the deployment guide to use

## Usage Summary

1. **Extract ZIP** to your local machine
2. **Navigate** to `examples/01-prerequisites/`
3. **Copy** `terraform.tfvars.example` to `terraform.tfvars`
4. **Update** variables with your Azure environment details
5. **Run** `terraform init && terraform apply`
6. **Navigate** to `examples/02-vm-deployment/`
7. **Repeat** steps 3-5 for VM deployment
8. **Connect** to your VM using the output public IP
