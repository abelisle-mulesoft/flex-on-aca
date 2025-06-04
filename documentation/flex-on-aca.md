Flex Gateway on ACA Informal Guide
===========================================================

## Table of Contents

- [Introduction](#introduction)
  - [Scope](#scope)
  - [Prerequisites](#prerequisites)
  - [Terminology](#terminology)
- [Running Flex Gateway on Azure Container Apps](#running-flex-gateway-on-azure-container-apps)
  - [Part 1 - Register Flex Gateway Instance](#part-1---register-flex-gateway-instance)
  - [Part 2 - Create Azure Resources](#part-2---create-azure-resources)
- [Conclusion](#conclusion)

<p>&nbsp;</p>

# Introduction
This informal guide provides a suggested approach to establish a baseline or foundation for deploying and running Anypoint Flex Gateway in Connected mode on Azure Container Apps (ACA).

## Scope

This guide and the suggested approach were tested using Flex Gateway versions 1.8.3 and 1.9.3 running in Connected Mode. The current revision utilizes the `FLEX_CONFIG` environment variable, introduced in version 1.7.0, and the [Readiness Probe](https://docs.mulesoft.com/gateway/latest/flex-conn-readiness-liveness#configure-external-components), introduced in version 1.8.0. Therefore, the content applies to Flex Gateway versions 1.8.0 and above. 

Although this guide primarily focuses on running Flex Gateway in [Connected Mode](https://docs.mulesoft.com/gateway/latest/#connected_mode), most of the content still applies if you have experience running Flex Gateway in [Local Mode](https://docs.mulesoft.com/gateway/latest/#local_mode).

Finally, the scope is intentionally limited to establishing a baseline or minimal installation and configuration of Flex Gateway on Azure Container Apps. This guide does not cover advanced topics such as creating a dedicated Azure Virtual Network (VNet), high availability, securing and hardening Azure resources, Flex Gateway, and any APIs it manages.

## Approach Overview
The suggested approach herein leverages the Azure command-line interface (CLI) for provisioning and configuring Azure resources. However, assuming you have experience provisioning and configuring Azure resources, you can complete all the steps using the Azure portal or leverage an Infrastructure as Code (IaC) approach, such as Azure Resource Manager, Ansible, or Terraform.

Furthermore, this guide leverages shell variables whenever possible to store reusable information and eliminate the need for editing commands included in this guide.

> [!NOTE]
> This guide was authored on a Mac computer. If you use a Windows computer, you must adjust the commands in this guide accordingly.

## Prerequisites

- You must have the Azure CLI installed. Please refer to the article [How to install the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) in the [Azure Command-Line Interface (CLI) documentation](https://learn.microsoft.com/en-us/cli/azure/?view=azure-cli-latest) for more information.

- You must authenticate to Azure using the Azure CLI. Please refer to the article [Authenticate to Azure using Azure CLI](https://learn.microsoft.com/en-us/cli/azure/authenticate-azure-cli) in the [Azure Command-Line Interface (CLI) documentation](https://learn.microsoft.com/en-us/cli/azure/?view=azure-cli-latest) for more information.

- You must register the resource providers `Microsoft.App` and `Microsoft.OperationalInsights`, typically done via the Aure CLI as follows:

  ```shell
  az provider register --namespace Microsoft.App
  az provider register --namespace Microsoft.OperationalInsights --wait
  ```

- You must have sufficient privileges in your Azure account to create and configure services and resources, including but not limited to resource group, container apps environment, and container.

> [!TIP]
> The [Tutorial: Deploy your first container app](https://learn.microsoft.com/en-us/azure/container-apps/tutorial-deploy-first-app-cli?tabs=bash) covers several of these prerequisites if you are new to the Azure CLI or unfamiliar with them.

## Terminology

Before proceeding, it is essential to understand the following terminology used in the context of this document.

|   |   |
| - | - |
| **Flex Gateway Instance** | An instance is a logical entity that groups one or more Flex Gateway replicas. When adding Flex Gateway to Anypoint Runtime Manager, you register a new Flex Gateway instance using the `flexctl` utility, which outputs a registration file. The content of the registration file is specific to the instance you registered and the Anypoint organization or business group and environment where you registered it. |
| **Flex Gateway Replica**  | A replica is a runtime unit or a copy of Flex Gateway running on a supported operating system or infrastructure (e.g., Docker, Podman, Kubernetes, OpenShift). You associate replicas to an instance by running Flex Gateway using the same registration file. You run two or more replicas for each Flex Gateway instance to achieve high availability. |

# Running Flex Gateway on Azure Container Apps

Running Anypoint Flex Gateway on Azure Container Apps is relatively straightforward. There are a few minor gotchas, but nothing complex.

- In part 1, you register the Flex Gateway instance.
- In part 2, you create the Azure resources to run a single Flex Gateway replica:
  - First, you create an Azure resource group to hold all resources related to Flex Gateway. 
  - Then, you create a container apps environment, "a secure boundary around one or more container apps and jobs" [1].
  - Next, you create a YAML file to describe how to configure and run the container app for Flex Gateway. This step is unavoidable, as the Azure CLI does not support all the required configurations necessary to run Flex Gateway. You can find more information in section **2.4 - Create Flex Gateway YAML File**, below. 
  - Finally, you create the Flex Gateway container app.

## Part 1 - Register Flex Gateway Instance

The easiest approach is to follow the high-level process ***Add a Self-Managed Flex Gateway*** described in **Anypoint Runtime Manager** because it prepopulates values that simplify the efforts significantly. In this guide, however, you only need to complete steps 1 (***Pull the image***) and 2 (***Register your gateway***) as if adding a Flex Gateway instance on Docker to generate the registration file. Before you begin, it is crucial to understand that a Flex Gateway instance is tied to:

1. An Anypoint organization or business group, and
2. An Anypoint environment (e.g., sandbox, production).

<img src="assets/images/flex-on-aca-1-0-01-runtime-manager-add-flex.png" style="width:6.5in;height:5.2in"/>

In the screen capture above, the **Demo** business group and the **Dev** environment are selected, and the prepopulated values reflect those selections.

### 1.1 - Pull Flex Gateway Docker Image

Step 1 (***Pull the image***) of the **Anypoint Runtime Manager** generic instructions consists of downloading the Flex Gateway Docker image from Docker Hub. You must have Docker installed locally and running (e.g., Docker Desktop) to complete this step and the next.

- Open a terminal (Linux or Mac) and execute the following Docker command.

  ```shell
  docker pull mulesoft/flex-gateway
  ```

  <img src="assets/images/flex-on-aca-1-1-01-docker-pull.png" style="width:4.5in;height:1.3in" />

### 1.2 – Register Flex Gateway Instance

Step 2 (***Register your gateway***) of the **Anypoint Runtime Manager** generic instructions involves running the Docker image to register a new Flex Gateway instance with the Anypoint Platform control plane. As mentioned, the most straightforward approach is to leverage the command generated by Anypoint Runtime Manager, which includes prepopulated values based on the Anypoint organization or business group and the environment selected.

- First, log into the Anypoint Platform (<https://anypoint.mulesoft.com>).

- On the landing page, select **Runtime Manager** in the **Management Center** section on the right.

- In **Runtime Manager**, if applicable, select the root organization or the appropriate business group (top right corner) and the correct environment (top left corner).

- Next, select the **Flex Gateways** option in the left vetical menu.

- On the **Flex Gateways** page, click on the **Self-Managed Flex Gateway** tab.

- Next, click the **Add Self-Managed Flex Gateway** button, click on the **Container** icon, and click the **Docker** logo.

- In the high-level instructions displayed, locate step 2 (***Register your gateway***) and copy the command.

  <img src="assets/images/flex-on-aca-1-2-01-register-gateway-command.png" style="width:6.5in;height:2.8in"/>

- Optionally, paste this command to a text editor to make any necessary changes. Do not forget to replace the `<gateway-name>` placeholder with the name of your instance. The generic command for step 2 is pasted here for convenience.

  ```shell
  docker run --rm --entrypoint flexctl -u $UID \
    -v "$(pwd)":/registration mulesoft/flex-gateway \
    registration create --organization=<organization-id> \
    --token=<registration-token> \
    --output-directory=/registration \
    --connected=true \
    <gateway-name>
  ```

> [!TIP]
> Adding the flag `--rm` before the flag `--entrypoint` disposes of the container automatically once the registration completes, as it is no longer required. Without this flag, the Docker container runs, the process completes, the container stops, and it remains on the system until you delete it, which you often forget to do.

- Finally, paste the command into the terminal (Linux or Mac) and execute it to register your new Flex Gateway instance.

  <img src="assets/images/flex-on-aca-1-2-02-register-gateway-execution.png" style="width:4.5in;height:1.3in"/>

The registration command creates a file named `registration.yaml` in the current directory on your computer. This registration file is specific to the Flex Gateway instance you just registered. As a reminder, it is tied to the selected 1) Anypoint organization or business group and 2) Anypoint environment.

> [!TIP]
> As a reminder, the content of the registration file is specific to the Flex Gateway instance you registered. Save it in a safe location for future reuse, as you will need it to start any new Flex Gateway replica.
> 
> Although optional, you could rename the registration file to reflect the name of the Flex Gateway instance. A suggested naming convention is `registration-<gateway-name>.yaml` (e.g., `registration-sm-flex-demo-dev-01.yaml`). 

## Part 2 - Create Azure Resources

Feel free to review the [Tutorial: Deploy your first container app](https://learn.microsoft.com/en-us/azure/container-apps/tutorial-deploy-first-app-cli?tabs=bash) before proceeding, as this section follows it loosely. 

### 2.1 - Set Environment Variables

As mentioned, this guide leverages shell variables whenever possible to store reusable information and eliminate the need for editing commands included in this guide.

- Copy the following to a text editor and edit the placeholders, replacing them with the values specific to your environment.

  ```shell
  RESOURCE_GROUP="<RESOURCE_GROUP>"
  LOCATION="<LOCATION>"
  ACA_ENVIRONMENT="<CONTAINER_APPS_ENVIRONMENT>"
  CONTAINER_APP_NAME="<CONTAINER_APP_NAME>"
  CONTAINER_APP_YAML_FILE="<CONTAINER_APP_YAML_FILE>"
  ```

  - `RESOURCE_GROUP` is the name of the Azure resource group to create - e.g., `"ACA-Flex-GW-Resource-Group"`.
  - `LOCATION` is the Azure region where you want to create all the Azure resources - e.g., `"eastus"`.
  - `ACA_ENVIRONMENT` is the name of the Azure container apps environment to create - e.g., `"ACA-Flex-GW-Environment"`. 
  - `CONTAINER_APP_NAME` is the name of the Flex Gateway container app - e.g., `"aca-flex-gw-app"`. 
  - `CONTAINER_APP_YAML_FILE` is the name of the YAML file that describes how to configure and run the container app for Flex Gateway - e.g., `"ACA-Flex-GW-App-Config.yaml"`. 

  > [!IMPORTANT]
  > The container app name must consist of lowercase alphanumeric characters and dashes ('-') only. It must start with a letter and end with an alphanumeric character and cannot include double dashes ('--'). The length must be between 2 and 32 characters inclusive.

- Paste the variable definitions into a terminal (Linux or Mac) and execute the implied shell commands.

  <img src="assets/images/flex-on-aca-2-1-01-env-variables.png" style="width:4.5in;height:0.9in"/>

### 2.2 - Create Azure Resource Group

- Resuming from the previous step, copy and execute the following Azure CLI command to create an Azure resource group.

  The resource group holds all resources related to deploying and running your Flex Gateway replicas in Azure Container Apps.

  ```shell
  az group create \
    --name $RESOURCE_GROUP \
    --location "$LOCATION"
  ```

  <img src="assets/images/flex-on-aca-2-2-01-create-resource-group.png" style="width:4.5in;height:1.6in"/>

### 2.3 - Create Container Apps Environment

- Resuming from the previous step, copy and execute the following Azure CLI command to create an Azure Container Apps environment. 

  As defined in the Azure Container Apps documentation, "*a Container Apps environment is a secure boundary around one or more container apps and jobs*" [1].

  ```shell
  az containerapp env create \
    --name $ACA_ENVIRONMENT \
    --resource-group $RESOURCE_GROUP \
    --location "$LOCATION"
  ```

  <img src="assets/images/flex-on-aca-2-3-01-create-container-apps-environment.png" style="width:4.5in;height:1.9in"/>

### 2.4 - Create Flex Gateway YAML File

In this step, you create a YAML file that specifies the required configuration to run a container app for Flex Gateway. This approach is necessary for two reasons. 

1. As mentioned, this guide's current revision utilizes the `FLEX_CONFIG` environment variable to inject the Flex Gateway registration information (i.e., the content of Flex Gateway registration) and some additional configuration (i.e., runtime debug logs, readiness probe). The `FLEX_CONFIG` environment variable, like any other environment variable, is a name-value pair. More to the point, the value of the FLEX_CONFIG environment variable is a string, but Flex Gateway expects configuration information provided in YAML format. As such, you must escape the YAML configuration, generally resulting in a very long string of characters, including escaped characters.

   The Azure CLI commands `az containerapp create` and `az containerapp up` accept the optional `--env-vars` parameter for specifying a list of environment variables for the container app. However, using the `--env-vars` parameter as part of a proof of concept was unsuccessful. It appears that the CLI stores the value of the `FLEX_CONFIG` environment variable in a way that prevents Flex Gateway from successfully parsing its YAML configuration. Using a YAML file that defines the required configuration to run a container app for Flex Gateway solves this first issue.

2. This guide enables ingress for the Flex Gateway container app to expose the container app on the internet (or public web) and avoid creating an Azure Load Balancer, public IP address, or other Azure resources to enable incoming HTTP requests or TCP traffic [2]. As ingress is enabled, Azure Container Apps automatically adds default startup, liveness, and readiness probes. Please refer to the article [Health probes in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/health-probes?tabs=yaml) in the **Azure Container Apps documentation** for more information.

   This guide's current revision overrides the default probes, utilizing the Flex Gateway [Readiness Probe](https://docs.mulesoft.com/gateway/latest/flex-conn-readiness-liveness#configure-external-components) introduced in version 1.8.0. Unfortunately, the Azure CLI commands `az containerapp create` and `az containerapp up` do not accept any parameter for specifying startup, liveness, and readiness probes. Using a YAML file that defines the required configuration to run a container app for Flex Gateway also solves this issue.



### 2.5 - Create the Flex Gateway Container App

- Resuming from step **2.3 - Create Container Apps Environment**, copy and execute the following Azure CLI command to create the Flex Gateway Azure Container App. 

  ```shell
  az containerapp create \
    --name $CONTAINER_APP_NAME \
    --resource-group $RESOURCE_GROUP \
    --yaml $CONTAINER_APP_YAML_FILE \
    --output table
  ```

  <img src="assets/images/flex-on-aca-2-5-01-create-flex-gw-container-app.png" style="width:4.5in;height:1.5in"/>

Notice the underlined URL in the screen capture. This fully qualified domain name (FQDN) points to the Azure container app just created. In other words, this is the base URL for Flex Gateway.

# Conclusion

This informal guide provided a suggested approach to establish a baseline or foundation for deploying and running Anypoint Flex Gateway in Connected mode on Azure Container Apps. In the current revision, the described approach leveraged the `FLEX_CONFIG` environment variable, introduced in version 1.7.0, and the **Readiness Probe**, introduced in version 1.8.0. Therefore, as a reminder, the content only applies to Flex Gateway versions 1.8.0 and above. 

# References

- 1 - https://learn.microsoft.com/en-us/azure/container-apps/environment
- 2 - https://learn.microsoft.com/en-us/azure/container-apps/ingress-overview

