// Data source to retrieve an existing Virtual Network
data "azurerm_virtual_network" "existing_vnet" {
  name                = var.vnet_name
  resource_group_name = var.vnet_resource_group
}

// Data source to retrieve an existing Subnet within the Virtual Network
data "azurerm_subnet" "existing_subnet" {
  name                 = var.subnet_name
  virtual_network_name = data.azurerm_virtual_network.existing_vnet.name
  resource_group_name  = data.azurerm_virtual_network.existing_vnet.resource_group_name
}

// Data source to retrieve existing Network Interfaces
data "azurerm_network_interface" "nic" {
  for_each            = toset(var.nic_names)
  name                = each.key
  resource_group_name = var.nic_resource_group
}

// Resource to create an Internal Load Balancer
resource "azurerm_lb" "internal_lb" {
  name                = var.lb_name
  location            = var.location
  resource_group_name = var.resource_group_name_tmp
  sku                 = "Standard"

  frontend_ip_configuration {
    name                          = "frontend-ip-config"
    subnet_id                     = data.azurerm_subnet.existing_subnet.id
    private_ip_address_allocation = "Dynamic"
    zones                         = var.location == "East US" || var.location == "South Central US" ? ["1", "2", "3"] : null
  }
}

// Single Backend pool for the Load Balancer
resource "azurerm_lb_backend_address_pool" "backend_pool" {
  name            = "backend-pool"
  loadbalancer_id = azurerm_lb.internal_lb.id
}

// HTTP Health Probe for the Load Balancer
resource "azurerm_lb_probe" "ilb_probe" {
  for_each             = toset(var.health_probe_ports)
  name                = "ilb-probe-${each.key}"
  loadbalancer_id     = azurerm_lb.internal_lb.id
  protocol            = "Tcp"
  port                = tonumber(each.key)
  interval_in_seconds = 5
  number_of_probes    = 2
}

// Correct Load Balancer Rule mapping frontend to backend ports
resource "azurerm_lb_rule" "ilb_rule" {
  for_each = {
    for index, port in var.lb_rule_ports :
    port => var.health_probe_ports[index]
  }

  name                           = "ilb-rule-${each.key}"
  loadbalancer_id                = azurerm_lb.internal_lb.id
  protocol                       = "Tcp"
  frontend_port                  = tonumber(each.key)
  backend_port                   = tonumber(each.value)
  frontend_ip_configuration_name = "frontend-ip-config"
  backend_address_pool_ids       = [azurerm_lb_backend_address_pool.backend_pool.id]
  probe_id                       = azurerm_lb_probe.ilb_probe[each.value].id
  enable_floating_ip             = false
  idle_timeout_in_minutes        = 4
  load_distribution              = "Default"
}

// Associate NICs with the shared Backend Pool with unique names
resource "azurerm_network_interface_backend_address_pool_association" "nic_backend" {
  for_each                = data.azurerm_network_interface.nic
  network_interface_id    = each.value.id
  ip_configuration_name   = each.value.ip_configuration[0].name
  backend_address_pool_id = azurerm_lb_backend_address_pool.backend_pool.id
}



// Output the Load Balancer ID
output "load_balancer_id" {
  description = "The ID of the Internal Load Balancer."
  value       = azurerm_lb.internal_lb.id
}

// Output the Frontend IP Configuration ID
output "frontend_ip_configuration_id" {
  description = "The ID of the Frontend IP Configuration."
  value       = azurerm_lb.internal_lb.frontend_ip_configuration[0].id
}

// Output the Backend Pool ID
output "backend_pool_id" {
  description = "The ID of the Backend Address Pool."
  value       = azurerm_lb_backend_address_pool.backend_pool.id
}

// Output the Load Balancer Probe IDs
output "probe_ids" {
  description = "The IDs of the Load Balancer Probes."
  value       = [for probe in azurerm_lb_probe.ilb_probe : probe.id]
}

// Output the Load Balancer Rule IDs
output "lb_rule_ids" {
  description = "The IDs of the Load Balancer Rules."
  value       = [for rule in azurerm_lb_rule.ilb_rule : rule.id]
}




provider "azurerm" {
  features {}
  subscription_id = "AAA"
}



// README.md

# Azure Internal Load Balancer Module

This Terraform module deploys an **Azure Internal Load Balancer (ILB)** with backend pool associations, health probes, and load balancing rules.

## Features
- Creates an Internal Load Balancer with a dynamic frontend IP configuration.
- Configures health probes for specified ports.
- Configures load balancing rules for specified ports.
- Associates multiple NICs with the backend pool.

## Inputs

| Variable               | Description                                          | Type   | Example                                   |
|-----------------------|------------------------------------------------------|--------|-------------------------------------------|
| resource_group_name_tmp | Resource Group for the Load Balancer                 | string | `aa-gnd-nonprod2-spoke-test1-rg`          |
| nic_resource_group     | Resource Group for NICs                             | string | `aa-gnd-nonprod2-spoke-test2-rg`          |
| location               | Azure region for deployment                         | string | `North Central US`                        |
| vnet_name              | Name of the existing Virtual Network                | string | `aa-gnd-aznc-test-vnet`                   |
| vnet_resource_group    | Resource Group for the Virtual Network             | string | `aa-gnd-nonprod2-spoke-test1-rg`          |
| subnet_name            | Name of the existing Subnet                        | string | `aa-gnd-aznc-test-snet`                  |
| lb_name                | Name of the Internal Load Balancer                 | string | `internal-load-balancer-test`            |
| nic_names              | List of NIC names to associate with the backend pool | list   | `["test1226", "test2498"]`              |
| health_probe_ports     | Ports for health probes                            | list   | `["8080", "8443"]`                      |
| lb_rule_ports          | Ports for load balancing rules                     | list   | `["80", "443"]`                         |

## Outputs

| Output                      | Description                             |
|-----------------------------|-----------------------------------------|
| load_balancer_id            | ID of the Internal Load Balancer         |
| frontend_ip_configuration_id | ID of the Frontend IP Configuration     |
| backend_pool_id             | ID of the Backend Address Pool          |
| probe_ids                   | IDs of the Load Balancer Probes         |
| lb_rule_ids                 | IDs of the Load Balancer Rules          |

## Usage

```hcl
module "internal_lb" {
  source                  = "./module"
  resource_group_name_tmp = "aa-gnd-nonprod2-spoke-test1-rg"
  location                = "North Central US"
  vnet_name               = "aa-gnd-aznc-test-vnet"
  vnet_resource_group     = "aa-gnd-nonprod2-spoke-test1-rg"
  nic_resource_group      = "aa-gnd-nonprod2-spoke-test2-rg"
  subnet_name             = "aa-gnd-aznc-test-snet"
  lb_name                 = "internal-load-balancer-test"
  nic_names               = ["test1226", "test2498"]
  health_probe_ports      = ["8080", "8443"]
  lb_rule_ports           = ["80", "443"]
}
```

## License

MIT License.





variable "resource_group_name_tmp" {
  description = "The name of the Resource Group to create the Load Balancer in."
  type        = string
}

variable "nic_resource_group" {
  description = "The name of the Resource Group containing the NICs."
  type        = string
}

variable "location" {
  description = "The Azure region where the resources will be deployed."
  type        = string
}

variable "vnet_name" {
  description = "The name of the existing Virtual Network."
  type        = string
}

variable "vnet_resource_group" {
  description = "The name of the Resource Group where the Virtual Network exists."
  type        = string
}

variable "subnet_name" {
  description = "The name of the existing Subnet."
  type        = string
}

variable "lb_name" {
  description = "The name of the Internal Load Balancer."
  type        = string
}

variable "nic_names" {
  description = "List of NIC names to associate with the backend pool."
  type        = list(string)
}

variable "health_probe_ports" {
  description = "List of ports for health probes."
  type        = list(string)
}

variable "lb_rule_ports" {
  description = "List of frontend ports for load balancing rules."
  type        = list(string)
}




module "internal_lb" {
  source                  = "./module"
  resource_group_name_tmp = "aa-gnd-nonprod2-spoke-test1-rg"
  location                = "North Central US"
  vnet_name               = "aa-gnd-aznc-test-vnet"
  vnet_resource_group     = "aa-gnd-nonprod2-spoke-test1-rg"
  nic_resource_group      = "aa-gnd-nonprod2-spoke-test2-rg"
  subnet_name             = "aa-gnd-aznc-test-snet"
  lb_name                 = "internal-load-balancer-test"
  nic_names               = ["test1226", "test2498"]
  health_probe_ports      = ["80", "443"]
  lb_rule_ports           = ["80", "443"]
}
