provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}

provider "azurerm" {
  alias           = "sub1"
  subscription_id = "11111111-1111-1111-1111-111111111111"
  features        {}
}

provider "azurerm" {
  alias           = "sub2"
  subscription_id = "22222222-2222-2222-2222-222222222222"
  features        {}
}

resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = var.location
}

resource "azurerm_virtual_network" "vnet" {
  name                = var.vnet_name
  address_space       = [var.vnet_address_space]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_subnet" "subnet" {
  name                 = var.subnet_name
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = [var.subnet_address_prefix]
}

resource "azurerm_lb" "ilb" {
  name                = var.lb_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "Standard"
  frontend_ip_configuration {
    name                          = "LoadBalancerFrontEnd"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_lb_backend_address_pool" "bpepool" {
  name                = "BackendPool"
  resource_group_name = azurerm_resource_group.rg.name
  loadbalancer_id     = azurerm_lb.ilb.id
}

resource "azurerm_lb_probe" "health_probe" {
  name                = "HealthProbe"
  resource_group_name = azurerm_resource_group.rg.name
  loadbalancer_id     = azurerm_lb.ilb.id
  protocol            = "Tcp"
  port                = 80
  interval_in_seconds = 5
  number_of_probes    = 2
}

resource "azurerm_lb_rule" "lb_rule" {
  name                           = "LBRule"
  resource_group_name            = azurerm_resource_group.rg.name
  loadbalancer_id                = azurerm_lb.ilb.id
  protocol                      = "Tcp"
  frontend_port                 = 80
  backend_port                  = 80
  frontend_ip_configuration_name = "LoadBalancerFrontEnd"
  backend_address_pool_id        = azurerm_lb_backend_address_pool.bpepool.id
  probe_id                      = azurerm_lb_probe.health_probe.id
}

resource "azurerm_network_interface_backend_address_pool_association" "vm1_nic_association" {
  network_interface_id    = var.vm_nics["vm1"].nic_id
  ip_configuration_name   = var.vm_nics["vm1"].ip_configuration_name
  backend_address_pool_id = azurerm_lb_backend_address_pool.bpepool.id
  provider                = azurerm.sub1
}

resource "azurerm_network_interface_backend_address_pool_association" "vm2_nic_association" {
  network_interface_id    = var.vm_nics["vm2"].nic_id
  ip_configuration_name   = var.vm_nics["vm2"].ip_configuration_name
  backend_address_pool_id = azurerm_lb_backend_address_pool.bpepool.id
  provider                = azurerm.sub2
}

variable "subscription_id" {
  description = "Azure Subscription ID"
  type        = string
  default     = "33333333-3333-3333-3333-333333333333"
}

variable "resource_group_name" {
  description = "Name of the Resource Group"
  type        = string
  default     = "rg-ilb"
}

variable "location" {
  description = "Azure region for resources"
  type        = string
  default     = "East US"
}

variable "vnet_name" {
  description = "Virtual Network name"
  type        = string
  default     = "vnet-ilb"
}

variable "vnet_address_space" {
  description = "Address space for the VNet"
  type        = string
  default     = "10.0.0.0/16"
}

variable "subnet_name" {
  description = "Subnet name"
  type        = string
  default     = "subnet-ilb"
}

variable "subnet_address_prefix" {
  description = "Subnet address prefix"
  type        = string
  default     = "10.0.1.0/24"
}

variable "lb_name" {
  description = "Internal Load Balancer name"
  type        = string
  default     = "ilb-test"
}

variable "vm_nics" {
  description = "Map of NICs with subscription, resource group, NIC ID, and IP configuration name"
  type = map(object({
    subscription_id        = string
    resource_group_name    = string
    nic_id                 = string
    ip_configuration_name  = string
  }))
  default = {
    "vm1" = {
      subscription_id       = "11111111-1111-1111-1111-111111111111"
      resource_group_name   = "rg-vm1"
      nic_id                = "/subscriptions/11111111-1111-1111-1111-111111111111/resourceGroups/rg-vm1/providers/Microsoft.Network/networkInterfaces/vm1-nic"
      ip_configuration_name = "ipconfig1"
    }
    "vm2" = {
      subscription_id       = "22222222-2222-2222-2222-222222222222"
      resource_group_name   = "rg-vm2"
      nic_id                = "/subscriptions/22222222-2222-2222-2222-222222222222/resourceGroups/rg-vm2/providers/Microsoft.Network/networkInterfaces/vm2-nic"
      ip_configuration_name = "ipconfig1"
    }
  }
}

output "load_balancer_private_ip" {
  value = azurerm_lb.ilb.frontend_ip_configuration[0].private_ip_address
}
