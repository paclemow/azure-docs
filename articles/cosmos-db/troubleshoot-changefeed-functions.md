---
title: Diagnose and troubleshoot issues when using Azure Cosmos DB Trigger in Azure Functions
description: Common issues, workarounds, and diagnostic steps, when using the Azure Cosmos DB Trigger with Azure Functions
author: ealsur
ms.service: cosmos-db
ms.date: 04/16/2019
ms.author: maquaran
ms.topic: troubleshooting
ms.reviewer: sngun
---

# Diagnose and troubleshoot issues when using Azure Cosmos DB Trigger in Azure Functions

This article covers common issues, workarounds, and diagnostic steps, when you use the [Azure Cosmos DB Trigger](change-feed-functions.md) with Azure Functions.

## Dependencies

The Azure Cosmos DB Trigger and bindings depend on the extension packages over the base Azure Functions runtime. Always keep these packages updated, as they might include fixes and new features that might address any potential issues you may encounter:

* For Azure Functions V2, see [Microsoft.Azure.WebJobs.Extensions.CosmosDB](https://www.nuget.org/packages/Microsoft.Azure.WebJobs.Extensions.CosmosDB).
* For Azure Functions V1, see [Microsoft.Azure.WebJobs.Extensions.DocumentDB](https://www.nuget.org/packages/Microsoft.Azure.WebJobs.Extensions.DocumentDB).

This article will always refer to Azure Functions V2 whenever the runtime is mentioned, unless explicitly specified.

## Consuming the Cosmos DB SDK separately from the Trigger and bindings

The key functionality of the extension package is to provide support for the Azure Cosmos DB trigger and bindings. It also includes the [Azure Cosmos DB .NET SDK](sql-api-sdk-dotnet-core.md), which is helpful if you want to interact with Azure Cosmos DB programmatically without using the trigger and bindings.

If want to to use the Azure Cosmos DB SDK, make sure that you don't add to your project another NuGet package reference. Instead, **let the SDK reference resolve through the Azure Functions' Extension package**.

Additionally, if you are manually creating your own instance of the [Azure Cosmos DB SDK client](./sql-api-sdk-dotnet-core.md), you should follow the pattern of having only one instance of the client [using a Singleton pattern approach](../azure-functions/manage-connections.md#documentclient-code-example-c). This process will avoid the potential socket issues in your operations.

## Common known scenarios and workarounds

### Azure Function fails with error message "Either the source collection 'collection-name' (in database 'database-name') or the lease collection 'collection2-name' (in database 'database2-name') does not exist. Both collections must exist before the listener starts. To automatically create the lease collection, set 'CreateLeaseCollectionIfNotExists' to 'true'"

This means that either one or both of the Azure Cosmos containers required for the trigger to work do not exist or are not reachable to the Azure Function. **The error itself will tell you which Azure Cosmos database and containers is the trigger looking for** based on your configuration.

1. Verify the `ConnectionStringSetting` attribute and that it **references a setting that exists in your Azure Function App**. The value on this attribute shouldn't be the Connection String itself, but the name of the Configuration Setting.
2. Verify that the `databaseName` and `collectionName` exist in your Azure Cosmos account. If you are using automatic value replacement (using `%settingName%` patterns), make sure the name of the setting exists in your Azure Function App.
3. If you don't specify a `LeaseCollectionName/leaseCollectionName`, the default is "leases". Verify that such container exists. Optionally you can set the `CreateLeaseCollectionIfNotExists` attribute in your Trigger to `true` to automatically create it.
4. Verify your [Azure Cosmos account's Firewall configuration](how-to-configure-firewall.md) to see to see that it's not it's not blocking the Azure Function.

### Azure Function fails to start with "Shared throughput collection should have a partition key"

The previous versions of the Azure Cosmos DB Extension did not support using a leases container that was created within a [shared throughput database](./set-throughput.md#set-throughput-on-a-database). To resolve this issue, update the [Microsoft.Azure.WebJobs.Extensions.CosmosDB](https://www.nuget.org/packages/Microsoft.Azure.WebJobs.Extensions.CosmosDB) extension to get the latest version.

### Azure Function fails to start with "The lease collection, if partitioned, must have partition key equal to id."

This error means that your current leases container is partitioned, but the partition key path is not `/id`. To resolve this issue, you need to recreate the leases container with `/id` as the partition key.

### You see a "Value cannot be null. Parameter name: o" in your Azure Functions logs when you try to Run the Trigger

This issue appears if you are using the Azure portal and you try to select the **Run** button on the screen when inspecting an Azure Function that uses the trigger. The trigger does not require for you to select Run to start, it will automatically start when the Azure Function is deployed. If you want to check the Azure Function's log stream on the Azure portal, just go to your monitored container and insert some new items, you will automatically see the Trigger executing.

### My changes take too long be received

This scenario can have multiple causes and all of them should be checked:

1. Is your Azure Function deployed in the same region as your Azure Cosmos account? For optimal network latency, both the Azure Function and your Azure Cosmos account should be colocated in the same Azure region.
2. Are the changes happening in your Azure Cosmos container continuous or sporadic?
If it's the latter, there could be some delay between the changes being stored and the Azure Function picking them up. This is because internally, when the trigger checks for changes in your Azure Cosmos container and finds none pending to be read, it will sleep for a configurable amount of time (5 seconds, by default) before checking for new changes (to avoid high RU consumption). You can configure this sleep time through the `FeedPollDelay/feedPollDelay` setting in the [configuration](../azure-functions/functions-bindings-cosmosdb-v2.md#trigger---configuration) of your trigger (the value is expected to be in milliseconds).
3. Your Azure Cosmos container might be [rate-limited](./request-units.md).
4. You can use the `PreferredLocations` attribute in your trigger to specify a comma-separated list of Azure regions to define a custom preferred connection order.

### Some changes are missing in my Trigger

If you find that some of the changes that happened in your Azure Cosmos container are not being picked up by the Azure Function, there is an initial investigation step that needs to take place.

When your Azure Function receives the changes, it often processes them, and could optionally, send the result to another destination. When you are investigating missing changes, make sure you **measure which changes are being received at the ingestion point** (when the Azure Function starts), not on the destination.

If some changes are missing on the destination, this could mean that is some error happening during the Azure Function execution after the changes were received.

In this scenario, the best course of action is to add `try/catch blocks` in your code and inside the loops that might be processing the changes, to detect any failure for a particular subset of items and handle them accordingly (send them to another storage for further analysis or retry). 

> **The Azure Cosmos DB Trigger, by default, won't retry a batch of changes if there was an unhandled exception** during your code execution. This means that the reason that the changes did not arrive at the destination is because that you are failing to process them.

If, you find that some changes were not received at all by your trigger, the most common scenario is that there is **another Azure Function running**. It could be another Azure Function deployed in Azure or an Azure Function running locally on a developer's machine that has **exactly the same configuration** (same monitored and lease containers), and this Azure Function is stealing a subset of the changes you would expect your Azure Function to process.

Additionally, the scenario can be validated, if you know how many Azure Function App instances you have running. If you inspect your leases container and count the number of lease items within, the distinct values of the `Owner` property in them should be equal to the number of instances of your Function App. If there are more Owners than the known Azure Function App instances, it means that these extra owners are the one "stealing" the changes.

One easy way to workaround this situation, is to apply a `LeaseCollectionPrefix/leaseCollectionPrefix` to your Function with a new/different value or, alternatively, test with a new leases container.

## Next steps

* [Enable monitoring for your Azure Functions](../azure-functions/functions-monitoring.md)
* [Azure Cosmos DB .NET SDK Troubleshooting](./troubleshoot-dot-net-sdk.md)