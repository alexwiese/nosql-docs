---
title: Configure and manage soft delete
titleSuffix: Azure Cosmos DB
description: Learn how to configure soft delete, recover soft-deleted resources, and purge resources in Azure Cosmos DB.
author: balaksms
ms.author: balaks
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 05/29/2026
appliesto:
  - ✅ NoSQL
---

 # Configure and manage soft delete in Azure Cosmos DB
 
 This article explains how to enable soft delete, recover soft-deleted resources, and permanently purge resources in Azure Cosmos DB.
 
 ## Prerequisites
 
 - An Azure Cosmos DB account.
 - Sufficient permissions (Cosmos DB Account Contributor or Owner role).
 - Your subscription must be registered for the Azure Cosmos DB soft delete preview feature. For registration steps, see [Register your subscription](soft-delete.md#register-your-subscription).

 
 ## Configure soft delete
 
 When soft delete is enabled on an account, any regular delete operation automatically becomes a soft delete. There's no separate action or API required. Whether you delete a resource through the Azure portal, SDK, or ARM API, the resource is soft-deleted and retained for the configured retention period.
 
 Follow these steps to configure soft delete on the database account using Azure portal.

 1. In the Azure portal, go to your Azure Cosmos DB account.
 
 2. In the left menu under **Settings**, select **Soft Delete**.
 
 3. Under **Soft Delete**, select **Enable**.
 
 4. Under **Retention Period**, set how long deleted resources are retained before automatic purge. The default is 1 day.
 
 5. Optionally, configure the **Permanent Delete Protection Period** to set the minimum time that must pass after soft deletion before a manual purge is permitted.
 
 6. Select **Save**.

     :::image type="content" source="media/how-to-softdelete/enable-soft-delete.png" alt-text="Screenshot showing how to enable soft delete and configure retention period in the Azure portal.":::

 
 ## Recover a soft-deleted resource
 
  Follow these steps to recover soft delete cosmos db resource using Azure portal.

 1. In the Azure portal, search for and select **Soft Deleted Resources** under Azure Cosmos DB.
 
 2. Select your subscription from the dropdown.
 
 3. The list shows all soft-deleted resources with their account name, subscription, type, deletion date, expiration date, and region.
 
 4. Find the resource you want to recover and select **Restore** in the **Actions** column.
 
    :::image type="content" source="media/how-to-softdelete/recover-soft-delete.png" alt-text="Screenshot showing the Soft Deleted Resources page with a soft-deleted container and the Restore and Purge action buttons.":::


 ## Purge a soft-deleted resource
 
 Purging permanently deletes a soft-deleted resource before the retention period ends. Once purged, the resource can't be recovered through soft delete.
 
 1. In the Azure portal, search for and select **Soft Deleted Resources** under Azure Cosmos DB.
 
 2. Select your subscription from the dropdown.
 
 3. Find the resource you want to permanently delete and select **Purge** in the **Actions** column.
 
    :::image type="content" source="media/how-to-softdelete/purge-soft-delete.png" alt-text="Screenshot showing the Soft Deleted Resources page with the Purge button highlighted.":::