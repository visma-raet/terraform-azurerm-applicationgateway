# terraform-azurerm-applicationgateway

[![Terraform](https://github.com/visma-raet/terraform-azurerm-applicationgateway/actions/workflows/terraform.yml/badge.svg)](https://github.com/visma-raet/terraform-azurerm-applicationgateway/actions/workflows/terraform.yml)

## Deploys a Azure Application Gateway

This Terraform module deploys an Application Gateway on Azure

### NOTES

* Default SKU Tier is set to Standard_V2
* Default SKU Capacity is set to 1

## Usage in Terraform 0.15

```terraform
data "azurerm_resource_group" "appgwvnetrsg" {
  name = "vnetrsg-aks"
}

data "azurerm_virtual_network" "appgwvnet" {
  name                = "vnet-aks"
  resource_group_name = data.azurerm_resource_group.aksvnetrsg.name
}

resource "azurerm_subnet" "appgwsubnet" {
  name                 = "subnet-agic"
  resource_group_name  = data.azurerm_resource_group.aksvnetrsg.name
  virtual_network_name = data.azurerm_virtual_network.aksvnet.name
  address_prefixes     = ["10.100.1.0/24"]
}

resource "azurerm_subnet" "appgwsubnet" {
  name                 = "subnet-agic"
  resource_group_name  = "rsg-network"
  virtual_network_name = data.azurerm_virtual_network.aksvnet.name
  address_prefixes     = ["10.10.1.0/24"]
}

module "appgateway" {
  source                = "github.com/visma-raet/terraform-azurerm-applicationgateway"
  name                  = var.appgw_name
  resource_group_name   = var.appgw_rsg
  location              = var.location
  create_resource_group = true
  vnet_subnet_id        = azurerm_subnet.appgwsubnet.id
}
```

To connect the Application Gateway with a Kubernetes Cluster (AKS) the following setup adds the required roles and routes to work with `kubenet`:

```terraform
data "azurerm_user_assigned_identity" "appgw" {
  name                = format("ingressapplicationgateway-%s", module.aks.cluster_name)
  resource_group_name = module.aks.node_resource_group

  depends_on = [
    module.appgateway
  ]
}

resource "azurerm_role_assignment" "app_gw" {
  scope                = module.appgateway.id
  role_definition_name = "Contributor"
  principal_id         = data.azurerm_user_assigned_identity.appgw.principal_id
}

resource "azurerm_role_assignment" "appgw_resource_group" {
  scope                = module.appgateway.resource_group_id
  role_definition_name = "Reader"
  principal_id         = data.azurerm_user_assigned_identity.appgw.principal_id
}

data "azurerm_resources" "routetables" {
  resource_group_name = module.aks.node_resource_group
  type                = "Microsoft.Network/routeTables"
}

resource "azurerm_subnet_route_table_association" "appgwroute" {
  subnet_id      = azurerm_subnet.appgwsubnet.id
  route_table_id = data.azurerm_resources.routetables.resources[0].id
  depends_on = [
    data.azurerm_resources.routetables
  ]
}

```

## Authors

Originally created by [Visma-raet](http://github.com/visma-raet)

## License

[MIT](LICENSE)
