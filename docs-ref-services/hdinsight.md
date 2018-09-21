---
title: Azure HDInsight Java SDK
description: Reference for Azure HDInsight Java SDK
keywords: Azure, Java, SDK, API, HDInsight
author: tylerfox
ms.author: tyfox
manager: arindamc
ms.date: 9/20/2018
ms.topic: article
ms.prod: azure
ms.technology: azure
ms.devlang: java
ms.service: hdinsight
---

# HDInsight Java Management SDK Preview

## Overview

The HDInsight Java SDK provides classes and methods that allow you to manage your HDInsight clusters. It includes operations to create, delete, update, list, scale, execute script actions, monitor, get properties of HDInsight clusters, and more.

## Prerequisites

* An Azure account. If you don't have one, [get a free trial](https://azure.microsoft.com/free/).
* [Java JDK](https://www.oracle.com/technetwork/java/javase/downloads/index.html)
* [Maven](https://maven.apache.org/install.html)

## SDK Installation

The HDInsight Java SDK is available through Maven [here](https://mvnrepository.com/artifact/com.microsoft.azure.hdinsight.v2018_06_01_preview/azure-mgmt-hdinsight). Add the following dependency to your pom.xml:

```
<dependency>
    <groupId>com.microsoft.azure.hdinsight.v2018_06_01_preview</groupId>
    <artifactId>azure-mgmt-hdinsight</artifactId>
    <version>1.0.0-beta-1</version>
</dependency>
```

You will also need to add the following dependencies to your pom.xml:

* [Azure Client Authentication Library:](https://mvnrepository.com/artifact/com.microsoft.azure/azure-client-authentication/1.6.2)
```
<dependency>
    <groupId>com.microsoft.azure</groupId>
    <artifactId>azure-client-authentication</artifactId>
    <version>1.6.2</version>
    <scope>test</scope>
</dependency>
```

* [Azure Java Client Runtime For ARM:](https://mvnrepository.com/artifact/com.microsoft.azure/azure-arm-client-runtime/1.6.2)
```
<dependency>
    <groupId>com.microsoft.azure</groupId>
    <artifactId>azure-arm-client-runtime</artifactId>
    <version>1.6.2</version>
</dependency>
```

## Authentication

The SDK first needs to be authenticated with your Azure subscription.  Follow the example below to create a service principal and use it to authenticate. After this is done, you will have an instance of an `HDInsightManagementClientImpl`, which contains many methods (outlined in below sections) that can be used to perform management operations.

> [!NOTE]
> There are other ways to authenticate besides the below example that could potentially be better suited for your needs. All methods are outlined here: [Authenticate with the Azure management libraries for Java](https://docs.microsoft.com/en-us/java/azure/java-sdk-azure-authenticate?view=azure-java-stable)

### Authentication Example Using a Service Principal

First, login to [Azure Cloud Shell](https://shell.azure.com/bash). Verify you are currently using the subscription in which you want the service principal created. 

```azurecli-interactive
az account show
```

Your subscription information is displayed as JSON.

```json
{
  "environmentName": "AzureCloud",
  "id": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
  "isDefault": true,
  "name": "XXXXXXX",
  "state": "Enabled",
  "tenantId": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
  "user": {
    "cloudShellID": true,
    "name": "XXX@XXX.XXX",
    "type": "user"
  }
}
```

If you're not logged into the correct subscription, select the correct one by running: 
```azurecli-interactive
az account set -s <name or ID of subscription>
```

> [!IMPORTANT]
> If you have not already registered the HDInsight Resource Provider by another method (such as by creating an HDInsight Cluster through the Azure Portal), you need to do this once before you can authenticate. This can be done from the [Azure Cloud Shell](https://shell.azure.com/bash) by running the following command:
>```azurecli-interactive
>az provider register --namespace Microsoft.HDInsight
>```

Next, choose a name for your service principal and create it with the following command:

```azurecli-interactive
az ad sp create-for-rbac --name <Service Principal Name> --sdk-auth
```

The service principal information is displayed as JSON.

```json
{
  "clientId": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
  "clientSecret": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
  "subscriptionId": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
  "tenantId": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
```
Copy the below snippet and fill in `TENANT_ID`, `CLIENT_ID`, `CLIENT_SECRET`, and `SUBSCRIPTION_ID` with the strings from the JSON that was returned after running the command to create the service principal.

```java
import com.microsoft.azure.management.hdinsight.v2018_06_01_preview.*;
import com.microsoft.azure.management.hdinsight.v2018_06_01_preview.implementation.HDInsightManagementClientImpl;

public class Main {
    public static void main (String[] args) {

        // Tenant ID for your Azure Subscription
        String TENANT_ID = "";
        // Your Service Principal App Client ID
        String CLIENT_ID = "";
        // Your Service Principal Client Secret
        String CLIENT_SECRET = "";
        // Azure Subscription ID
        String SUBSCRIPTION_ID = "";

        ApplicationTokenCredentials credentials = new ApplicationTokenCredentials(
                CLIENT_ID,
                TENANT_ID,
                CLIENT_SECRET,
                AzureEnvironment.AZURE);

        HDInsightManagementClientImpl client = new HDInsightManagementClientImpl(credentials);
```


## Cluster Management

> [!NOTE]
> This section assumes you have already authenticated and constructed an `HDInsightManagementClientImpl` instance and store it in a variable called `client`. Instructions for authenticating and obtaining an `HDInsightManagementClientImpl` can be found in the Authentication section above.

### Create a Cluster

A new cluster can be created by calling `client.clusters().create()`.

#### Example

This example demonstrates how to create a Spark cluster with 2 head nodes and 1 worker node.

> [!NOTE]
> You first need to create a Resource Group and Storage Account, as explained below. If you have already created these, you can skip these steps.

##### Creating a Resource Group

You can create a resource group using the [Azure Cloud Shell](https://shell.azure.com/bash) by running
```azurecli-interactive
az group create -l <Region Name (i.e. eastus)> --n <Resource Group Name>
```
##### Creating a Storage Account

You can create a storage account using the [Azure Cloud Shell](https://shell.azure.com/bash) by running:
```azurecli-interactive
az storage account create -n <Storage Account Name> -g <Existing Resource Group Name> -l <Region Name (i.e. eastus)> --sku <SKU i.e. Standard_LRS>
```
Now run the following command to get the key for your storage account (you will need this to create a cluster):
```azurecli-interactive
az storage account keys list -n <Storage Account Name>
```
---
The below Java snippet creates a Spark cluster with 2 head nodes and 1 worker node. Fill in the blank variables as explained in the comments and feel free to change other parameters to suit your specific needs.

```java
// The name for the cluster you are creating
String clusterName = "";
// The name of your existing Resource Group
String resourceGroupName = "";
// Choose a username
String username = "";
// Choose a password
String password = "";
// Replace <> with the name of your storage account
String storageAccount = "<>.blob.core.windows.net";
// Storage account key you obtained above
String storageAccountKey = "";
// Choose a region
String location = "";
String container = "default";

HashMap<String, HashMap<String, String>> configurations = new HashMap<String, HashMap<String, String>>();
        HashMap<String, String> gateway = new HashMap<String, String>();
        gateway.put("restAuthCredential.enabled_credential", "True");
        gateway.put("restAuthCredential.username", username);
        gateway.put("restAuthCredential.password", password);
        configurations.put("gateway", gateway);

        ClusterCreateParametersExtended parameters = new ClusterCreateParametersExtended()
            .withLocation(location)
            .withTags(Collections.EMPTY_MAP)
            .withProperties(
                new ClusterCreateProperties()
                    .withClusterVersion("3.6")
                    .withOsType(OSType.LINUX)
                    .withClusterDefinition(new ClusterDefinition()
                            .withKind("spark")
                            .withConfigurations(configurations)
                    )
                    .withTier(Tier.STANDARD)
                    .withComputeProfile(new ComputeProfile()
                        .withRoles(List.of(
                            new Role()
                                .withName("headnode")
                                .withTargetInstanceCount(2)
                                .withHardwareProfile(new HardwareProfile()
                                    .withVmSize("Large")
                                )
                                .withOsProfile(new OsProfile()
                                    .withLinuxOperatingSystemProfile(new LinuxOperatingSystemProfile()
                                            .withUsername(username)
                                            .withPassword(password)
                                    )
                                ),
                            new Role()
                                    .withName("workernode")
                                    .withTargetInstanceCount(1)
                                    .withHardwareProfile(new HardwareProfile()
                                        .withVmSize("Large")
                                    )
                                    .withOsProfile(new OsProfile()
                                        .withLinuxOperatingSystemProfile(new LinuxOperatingSystemProfile()
                                            .withUsername(username)
                                            .withPassword(password)
                                        )
                                    )
                        ))
                    )
                    .withStorageProfile(new StorageProfile()
                        .withStorageaccounts(List.of(
                                new StorageAccount()
                                    .withName(storageAccount)
                                    .withKey(storageAccountKey)
                                    .withContainer(container)
                                    .withIsDefault(true)
                        ))
                    )
            );
        client.clusters().create(resourceGroupName, clusterName, parameters);
```
### List Clusters

#### List Clusters Under The Subscription

```java
client.clusters.list();
```
#### List Clusters By Resource Group

```java
client.clusters.listByResourceGroup("<Resource Group Name>");
```
> [!NOTE]
> Both `List()` and `ListByResourceGroup()` return a `PagedList<ClusterInner>` object. Calling `loadNext()` returns a list of clusters on that page and advances the `ClusterPaged` object to the next page. This can be repeated until `hasNextPage()` return `false`, indicating that there are no more pages.

#### Example

The following example prints the properties of all clusters for the current subscription:

```java
PagedList<ClusterInner> clusterPages = client.clusters().list();
while (true) {
    for (ClusterInner cluster : clusterPages.currentPage().items()) {
        System.out.println(cluster.name());
    }
    if (clusterPages.hasNextPage()) {
        clusterPages.loadNextPage();
    } else {
        break;
    }
}
```

### Get Cluster Details

To get properties for a given cluster:

```java
client.clusters.getByResourceGroup("<Resource Group Name>", "<Cluster Name>");
```

### Delete a Cluster

To delete a cluster:

```java
client.clusters.delete("<Resource Group Name>", "<Cluster Name>");
```

### Update Cluster Tags

You can update the tags of a given cluster like so:

```java
client.clusters.update("<Resource Group Name>", "<Cluster Name>", <Map<String,String> of Tags>);
```

### Scale Cluster

You can scale a given cluster's number of worker nodes by specifying a new size like so:

```java
client.clusters.resize("<Resource Group Name>", "<Cluster Name>", <Num of Worker Nodes (int)>)
```

## Cluster Monitoring

The HDInsight Management SDK can also be used to manage monitoring on your clusters via the Operations Management Suite (OMS).

### Enable OMS Monitoring

> [!NOTE]
> To enable OMS Monitoring, you must have an existing Log Analytics workspace. If you have not already created one, you can learn how to do that here: [Create a Log Analytics workspace in the Azure portal](https://docs.microsoft.com/en-us/azure/log-analytics/log-analytics-quick-create-workspace).

To enable OMS Monitoring on your cluster:

```java
client.extensions().enableMonitoring("<Resource Group Name", "<Cluster Name>", new ClusterMonitoringRequest().withWorkspaceId("<Workspace Id>"));
```

### View Status Of OMS Monitoring

To get the status of OMS on your cluster:

```java
client.extensions().getMonitoringStatus("<Resource Group Name", "Cluster Name");
```

### Disable OMS Monitoring

To disable OMS on your cluster:

```java
client.extensions().disableMonitoring("<Resource Group Name>", "<Cluster Name>");
```

## Script Actions

HDInsight provides a configuration method called script actions that invokes custom scripts to customize the cluster.
> [!NOTE]
> More information on how to use script actions can be found here: [Customize Linux-based HDInsight clusters using script actions](https://docs.microsoft.com/en-us/azure/hdinsight/hdinsight-hadoop-customize-cluster-linux)

### Execute Script Actions

You can execute script actions on a given cluster like so:

```java
RuntimeScriptAction scriptAction1 = new RuntimeScriptAction()
    .withName("<Script Name>")
    .withUri("<URL To Script>")
    .withRoles(<List<String> of roles>);
client.clusters().executeScriptActions(
    resourceGroupName, 
    clusterName, 
    new ExecuteScriptActionParameters().withPersistOnSuccess(false).withScriptActions(new LinkedList<>(Arrays.asList(scriptAction1)))); //add more RuntimeScriptActions to the list to execute multiple scripts
```

### Delete Script Action

To delete a specified persisted script action on a given cluster:

```java
client.scriptActions().delete("<Resource Group Name>", "<Cluster Name>", "<Script Name>");
```

### List Persisted Script Actions

> [!NOTE]
> Both `listByCluster()` returns a `PagedList<RuntimeScriptActionDetailInner>` object. Calling `currentPage().items()` returns a list of `RuntimeScriptActionDetailInner`, and `loadNextPage()` advances to the next page. This can be repeated until `hasNextPage()` returns `false`, indicating that there are no more pages.

To list all persisted script actions for the specified cluster:
```java
client.scriptActions().listByCluster("<Resource Group Name>", "<Cluster Name>");
```

#### Example

```java
PagedList<RuntimeScriptActionDetailInner> scriptsPaged = client.scriptActions().listByCluster(resourceGroupName, clusterName);
while (true) {
    for (RuntimeScriptActionDetailInner script : scriptsPaged.currentPage().items()) {
        System.out.println(script.name()); //There are methods to get other properties of RuntimeScriptActionDetail besides name(), such as status(), operation(), startTime(), endTime(), etc. See reference documentation.
    }
    if (scriptsPaged.hasNextPage()) {
        scriptsPaged.loadNextPage();
    } else {
        break;
    }
}
```

### List All Scripts' Execution History

To list all scripts' execution history for the specified cluster:

```java
client.scriptExecutionHistorys().listByCluster("<Resource Group Name>", "<Cluster Name>");
```

#### Example

This example prints all the details for all past script executions.

```java
PagedList<RuntimeScriptActionDetailInner> scriptExecutionsPaged = client.scriptExecutionHistorys().listByCluster(resourceGroupName, clusterName);
while (true) {
    for (RuntimeScriptActionDetailInner script : scriptExecutionsPaged.currentPage().items()) {
        System.out.println(script.name()); //There are methods to get other properties of RuntimeScriptActionDetail besides name(), such as status(), operation(), startTime(), endTime(), etc. See reference documentation.
    }
    if (scriptExecutionsPaged.hasNextPage()) {
        scriptExecutionsPaged.loadNextPage();
    } else {
        break;
    }
}
```