```hcl
locals {
  rg_name             = "rg-${var.short}-${var.loc}-${var.env}-02"
  vnet_name           = "vnet-${var.short}-${var.loc}-${var.env}-02"
  bastion_name        = "bst-${var.short}-${var.loc}-${var.env}-02"
  bastion_subnet_name = "AzureBastionSubnet"
  vm_subnet_name      = "VMSubnet"
  subnets = {
    (local.bastion_subnet_name) = {
      mask_size = 26
      netnum    = 0
    }
    (local.vm_subnet_name) = {
      mask_size = 26
      netnum    = 1
    }
  }
  nsg_name       = "nsg-${var.short}-${var.loc}-${var.env}-02"
  scale_set_name = "vmss-${var.short}-${var.loc}-${var.env}-02"
  admin_username = "Local${title(var.short)}${title(var.env)}Admin"
}

module "rg" {
  source = "libre-devops/rg/azurerm"

  rg_name  = local.rg_name
  location = local.location
  tags     = local.tags
}

module "shared_vars" {
  source = "libre-devops/shared-vars/azurerm"
}

locals {
  lookup_cidr = {
    for landing_zone, envs in module.shared_vars.cidrs : landing_zone => {
      for env, cidr in envs : env => cidr
    }
  }
}

module "subnet_calculator" {
  source = "libre-devops/subnet-calculator/null"

  base_cidr = local.lookup_cidr[var.short][var.env][0]
  subnets   = local.subnets
}


module "network" {
  source = "libre-devops/network/azurerm"

  rg_name  = module.rg.rg_name
  location = module.rg.rg_location
  tags     = module.rg.rg_tags

  vnet_name          = local.vnet_name
  vnet_location      = module.rg.rg_location
  vnet_address_space = [module.subnet_calculator.base_cidr]

  subnets = {
    for i, name in module.subnet_calculator.subnet_names :
    name => {
      address_prefixes = toset([module.subnet_calculator.subnet_ranges[i]])
    }
  }
}

module "nsg" {
  source = "libre-devops/nsg/azurerm"

  rg_name  = module.rg.rg_name
  location = module.rg.rg_location
  tags     = module.rg.rg_tags

  nsg_name              = local.nsg_name
  associate_with_subnet = true
  subnet_id             = module.network.subnets_ids[local.vm_subnet_name]
  custom_nsg_rules = {
    "AllowVnetInbound" = {
      priority                   = 100
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = "*"
      source_address_prefix      = "VirtualNetwork"
      destination_address_prefix = "VirtualNetwork"
    }
  }
}


module "bastion" {
  source = "libre-devops/bastion/azurerm"

  rg_name  = module.rg.rg_name
  location = module.rg.rg_location
  tags     = module.rg.rg_tags

  bastion_host_name        = local.bastion_name
  bastion_sku              = "Basic"
  virtual_network_id       = module.network.vnet_id
  create_bastion_nsg       = true
  create_bastion_nsg_rules = true
  create_bastion_subnet    = false
  external_subnet_id       = module.network.subnets_ids[local.bastion_subnet_name]
}


module "linux_vm_scale_set" {
  source = "../../"

  rg_name  = module.rg.rg_name
  location = module.rg.rg_location
  tags     = module.rg.rg_tags

  scale_sets = [
    {

      name = local.scale_set_name

      computer_name_prefix            = "vmss1"
      admin_username                  = local.admin_username
      instances                       = 1
      sku                             = "Standard_D4ds_v5"
      use_simple_image                = true
      vm_os_simple                    = "Ubuntu24.04Gen2"
      use_custom_image                = false
      disable_password_authentication = true
      overprovision                   = false    # Azure DevOps will set overprovision to false
      upgrade_mode                    = "Manual" # Azure DevOps will set to Manual anyway
      single_placement_group          = false    # Must be disabled for Azure DevOps or will fail
      enable_automatic_updates        = true
      create_asg                      = true

      admin_ssh_key = [
        {
          username   = local.admin_username
          public_key = data.azurerm_ssh_public_key.mgmt_ssh_key.public_key
        }
      ]

      identity_type = "SystemAssigned"
      network_interface = [
        {
          name                          = "nic-${local.scale_set_name}"
          primary                       = true
          enable_accelerated_networking = false
          ip_configuration = [
            {
              name                           = "ipconfig-${local.scale_set_name}"
              primary                        = true
              subnet_id                      = module.network.subnets_ids[local.vm_subnet_name]
              application_security_group_ids = []
            }
          ]
        }
      ]
      os_disk = {
        caching              = "ReadOnly"
        storage_account_type = "Premium_LRS"
        disk_size_gb         = 256
      }

      boot_diagnostics = {
        storage_account_uri = null
      }

      extension = []
    }
  ]
}




```
## Requirements

No requirements.

## Providers

| Name | Version |
|------|---------|
| <a name="provider_azurerm"></a> [azurerm](#provider\_azurerm) | 4.26.0 |
| <a name="provider_random"></a> [random](#provider\_random) | 3.7.1 |

## Modules

| Name | Source | Version |
|------|--------|---------|
| <a name="module_bastion"></a> [bastion](#module\_bastion) | libre-devops/bastion/azurerm | n/a |
| <a name="module_linux_vm_scale_set"></a> [linux\_vm\_scale\_set](#module\_linux\_vm\_scale\_set) | ../../ | n/a |
| <a name="module_network"></a> [network](#module\_network) | libre-devops/network/azurerm | n/a |
| <a name="module_nsg"></a> [nsg](#module\_nsg) | libre-devops/nsg/azurerm | n/a |
| <a name="module_rg"></a> [rg](#module\_rg) | libre-devops/rg/azurerm | n/a |
| <a name="module_shared_vars"></a> [shared\_vars](#module\_shared\_vars) | libre-devops/shared-vars/azurerm | n/a |
| <a name="module_subnet_calculator"></a> [subnet\_calculator](#module\_subnet\_calculator) | libre-devops/subnet-calculator/null | n/a |

## Resources

| Name | Type |
|------|------|
| [random_string.entropy](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/string) | resource |
| [azurerm_client_config.current_creds](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/client_config) | data source |
| [azurerm_key_vault.mgmt_kv](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/key_vault) | data source |
| [azurerm_resource_group.mgmt_rg](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/resource_group) | data source |
| [azurerm_ssh_public_key.mgmt_ssh_key](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/ssh_public_key) | data source |
| [azurerm_user_assigned_identity.mgmt_user_assigned_id](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/user_assigned_identity) | data source |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_Regions"></a> [Regions](#input\_Regions) | Converts shorthand name to longhand name via lookup on map list | `map(string)` | <pre>{<br/>  "eus": "East US",<br/>  "euw": "West Europe",<br/>  "uks": "UK South",<br/>  "ukw": "UK West"<br/>}</pre> | no |
| <a name="input_env"></a> [env](#input\_env) | This is passed as an environment variable, it is for the shorthand environment tag for resource.  For example, production = prod | `string` | `"dev"` | no |
| <a name="input_loc"></a> [loc](#input\_loc) | The shorthand name of the Azure location, for example, for UK South, use uks.  For UK West, use ukw. Normally passed as TF\_VAR in pipeline | `string` | `"uks"` | no |
| <a name="input_name"></a> [name](#input\_name) | The name of this resource | `string` | `"tst"` | no |
| <a name="input_short"></a> [short](#input\_short) | This is passed as an environment variable, it is for a shorthand name for the environment, for example hello-world = hw | `string` | `"libd"` | no |
| <a name="input_static_tags"></a> [static\_tags](#input\_static\_tags) | The tags variable | `map(string)` | <pre>{<br/>  "Contact": "info@cyber.scot",<br/>  "CostCentre": "671888",<br/>  "ManagedBy": "Terraform"<br/>}</pre> | no |

## Outputs

No outputs.
