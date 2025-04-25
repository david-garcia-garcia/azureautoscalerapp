[TOC]

# Azure Autoscaler Reference

## Installation

The Azure Autoscaler application is distributed as a container image, and needs to be deployed to a runtime of your choice. You also need to ensure the application is granted permissions to act on your Azure Resources in order to perform scaling operations.

### For kubernetes (with terraform examples)

Relevant readings:

* [Use a Microsoft Entra Workload ID on AKS - Azure Kubernetes Service | Microsoft Learn](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview?tabs=dotnet)
* [Deploy and configure an AKS cluster with workload identity - Azure Kubernetes Service | Microsoft Learn](https://learn.microsoft.com/en-us/azure/aks/workload-identity-deploy-cluster)

To deploy this on Kubernetes we will need:

* A managed identity: to grant permissions for the application
* A federated credential in our cluster
* A configmap with the application configuration
* A deployment with the application itself

```yaml
locals {
   resource_group_name = "your-resource-group"
   aks_namespace = "azureautoscaler"
   aks_info = {
      name = "my-aks-cluster-name"
      resource_group_name = "my-aks-cluster-resource-group"
   }
}

# Cluster info
data "azurerm_kubernetes_cluster" "cluster" {
  name                = local.aks_info.name
  resource_group_name = local.aks_info.resource_group_name
}

# Identity for the application
resource "azurerm_user_assigned_identity" "app" {
  name                = "azureautoscaler"
  resource_group_name = local.resource_group_name
  location            = "your-locastion"
}

# Assign permissions to resources that the application will be manipulating
resource "azurerm_role_assignment" "identity_azureresource_contributor" {
  for_each                         = {
    r0 = "/subscriptions/xxx/resourceGroups/rg-a/providers/Microsoft.ContainerService/managedClusters/akscluster/agentPools/linuxpool"
    r1 = "/subscriptions/xxx/resourceGroups/rg-b/providers/Microsoft.DBforMySQL/flexibleServers/mysql" 
  }
  scope                            = each.value
  role_definition_name             = "Contributor"
  principal_id                     = azurerm_user_assigned_identity.app.principal_id
  skip_service_principal_aad_check = false
}

# Service account
resource "kubernetes_service_account" "app" {
  metadata {
    name      = "workloadidentity-azureautoscaler"
    namespace = local.aks_namespace_name
    annotations = {
      "azure.workload.identity/client-id" = var.application_identity.client_id
    }
  }
}

# Federate the credential in your cluster
resource "azurerm_federated_identity_credential" "federated_credential" {
  name                = "k8s-fed-${data.azurerm_kubernetes_cluster.cluster.name}-${kubernetes_service_account.app.metadata[0].name}"
  resource_group_name = local.resource_group_name
  parent_id           = var.application_identity.id
  # Ojo que el formato de esto es imporantisimo y sigue un patrón concreto
  subject  = "system:serviceaccount:${local.aks_namespace}:${kubernetes_service_account.app.metadata[0].name}"
  issuer   = data.azurerm_kubernetes_cluster.cluster.oidc_issuer_url
  audience = ["api://AzureADTokenExchange"]

  # Optionally, you can specify a description and a list of claims_mapping
  # description = "Federated Identity Credential for AKS Workload"
  # claims_mapping = [...]
}

# Namespace
resource "kubernetes_namespace" "app" {
  metadata {
    name = local.aks_namespace
  }
}

# The applicaton configuration
resource "kubernetes_config_map" "app_config_yml" {
  metadata {
    name      = "azureautoscaler-config-yml"
    namespace = kubernetes_namespace.app.metadata[0].name
  }
  data = {
    "config.yml" = file("settings.yml") # Here read your application configuration file
  }
}

# The deployment
resource "kubernetes_deployment" "app" {
  wait_for_rollout = false
  metadata {
    name      = "azureautoscaler"
    namespace = kubernetes_namespace.app.metadata[0].name
    labels = {
      app = "azureautoscaler"
    }
  }
  spec {
    selector {
      match_labels = {
        app = local.deployment_app_label
      }
    }
    template {
      metadata {
        labels = {
          app                           = "azureautoscaler"
          "azure.workload.identity/use" = "true" # Important
        }
      }
      spec {
        image_pull_secrets {
          name = var.application_settings.image_pull_secret
        }

		# Important
        service_account_name = kubernetes_service_account.app.metadata[0].name

        container {
          name  = "app"
          image = local.app_image_to_use

          volume_mount {
            name       = "config-yml"
            mount_path = "/app/config.yml"
            sub_path   = "config.yml"
          }

        }
        volume {
          name = "config-yml"
          config_map {
            name = kubernetes_config_map.app_config_yml.metadata[0].name
          }
        }
      }
    }
  }
}
```

### For container services

Relevant readings:

* [Enable managed identity in container group - Azure Container Instances | Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-managed-identity)
* [Config maps for Azure Container Instances (Preview) - Azure Container Instances | Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-config-map?tabs=cli)

## The configuration file

### Configuration lifecycle

The application will parse, validate and load the configuration once during container startup. Any modifications to the configuration require a container restart.

## Global structure

The configuration file is a yaml object with the properties:

```yaml
DefaultResourceWhatIf: true # Default whatif for resources if not set
DefaultResourceEnabled: true # Default enabled for resources if not set
DefaultResourceFrequency: 10m # Default resource refresh frequency for resource if not set
# Uses .Net 8 logging configuration
# see https://learn.microsoft.com/en-us/dotnet/core/extensions/console-log-formatter#set-formatter-with-configuration
Logging:
  LogLevel:
    Default: "Trace"
    Microsoft.Hosting.Lifetime: "Trace"
  Console:
    FormatterName: "simple"
    FormatterOptions:
      SingleLine: true
      TimestampFormat: "HH:HH:mm:ss"
Resources:
  - R0
  - R1
  - R2
```

The relevant part of the configuration is the Resources array. This array contains a list of Resources that need to be scaled.

> **Note**
>
> A resource is actually a group of resources (of the same type) that share the same scaling configuration.

## Resource structure

### Scaling configurations

In this simple example, we will be scaling two Azure Sql Elastic Pools so that they will have 50 DTU during working hours, 20 DTU during the night, and 10 DTU during weekends.

```yaml
  - Resources:
      sbssqldevshared_dev_sbssqlpooldevshared:
        ResourceId: "/subscriptions/xxx/resourceGroups/rg-dev-shared/providers/Microsoft.Sql/servers/sql0/elasticPools/pool1"
      sbssqldevshared_dev_sbssqlpooldevshared2:
        ResourceId: "/subscriptions/xxx/resourceGroups/rg-dev-shared/providers/Microsoft.Sql/servers/sql0/elasticPools/pool2"
    Frequency: 5m
    WhatIf: true
    Enabled: true
    ScalingConfigurations:
      Nightly:
        TimeWindow:
          Days: ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"]
          Months: All
          StartTime: "21:00"
          EndTime: "05:00"
          TimeZone: Romance Standard Time
        ScalingRules:
          fixed:
            ScalingStrategy: Fixed
            Dimension: Dtu
            ScaleTarget: "(data) => (50).ToString()"
      Daily:
        TimeWindow:
          Days: ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"]
          Months: All
          StartTime: "05:00"
          EndTime: "20:59"
          TimeZone: Romance Standard Time
        ScalingRules:
          fixed:
            ScalingStrategy: Fixed
            Dimension: Dtu
            ScaleTarget: "(data) => (20).ToString()"
      Weekend:
        TimeWindow:
          Days: ["Saturday", "Sunday"]
          Months: All
          StartTime: "00:00"
          EndTime: "23:59"
          TimeZone: Romance Standard Time
        ScalingRules:
          fixed:
            ScalingStrategy: Fixed
            Dimension: Dtu
            ScaleTarget: "(data) => (10).ToString()"
```

As you can see in the previous example, a group of resources share a Resource Configuration, which consists of:

* **A map of Resources**: this is a list of all the resources in Azure this configuration will be operating on. They must all be of the same type.
* **Frequency**: how often should this resource state should be evaluated.
* **WhatIf**: you can set this to true prevent the application from actually manipulating the resources. This will let you see what the application would have done by analyzing the logs and ensure you are comfortable with the scaling configuration.
* **Enabled**: effectively disables the Resource Configuration
* **ScalingConfiguration**: A map containing each one of the scaling configurations that will be evaluated for the resource, it is important to note that:
  * All scaling configurations are always evaluated according to the TimeWindow. Multiple configurations can overlap without issues.
  * When multiple configurations overlap, and they provide different indications on scale dimension targets, the application will always utilize the **greatest** one.