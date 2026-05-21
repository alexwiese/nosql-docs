---
title: Soft Delete (Preview)
titleSuffix: Azure Cosmos DB
description: Learn about Azure Cosmos DB soft delete, a data resiliency feature that retains deleted accounts, databases, and containers for a configurable retention period, enabling fast recovery from accidental deletions.
author: balaksms
ms.author: balaks
ms.service: azure-cosmos-db
ms.topic: conceptual
ms.date: 05/21/2026
---

# Soft delete for Azure Cosmos DB (preview)

Azure Cosmos DB soft delete is a data resiliency feature that retains deleted resources (accounts, databases, containers) for a period of time (default 14 days) instead of permanently removing them at once. This enables fast recovery from accidental deletions without requiring a restore from backups. In effect, when soft delete is active, deleting a Azure Cosmos DB resource will mark it as “soft-deleted” like a recycle bin so that it remains internally preserved and can be restored within the retention window. After the retention period expires or if an authorized user explicitly purges it earlier, the resource is permanently deleted.

This feature addresses a critical operational need: accidental deletion of databases or containers can currently lead to extended downtime and complex restore procedures. Soft delete dramatically reduces downtime by allowing in-place “undo” of deletions in minutes. It provides a safety net so that if a Azure Cosmos DB resource is mistakenly removed, it can be quickly recovered with minimal disruption.

## Key benefits and use cases

- **Rapid recovery and minimal downtime**: Soft delete allows a deleted Azure Cosmos DB resource to be restored within minutes, since the data never truly left the service.
This is a huge improvement over traditional restore workflows that could take many hours. Applications can resume quickly after an accidental deletion, reducing
potential downtime from days to within minutes.

- **Protection against human error**: It acts as a safety net for mistakes. If an administrator or automation script erroneously deletes a database or container, the
data is not lost – it can be undeleted promptly. This significantly lowers the risk of catastrophic data loss due to user error and avoids business downtime.

- **Simplified restore process**: Recovery via soft delete is much simpler than a backup-based restore. The resource is brought back in-place with the same name,
settings, and data it had before deletion. There is no need to create new accounts or containers and no need to reconfigure networking or application connections, since the resource identity remains the same. This also means all ancillary configuration (indexes, throughput, permissions) remains intact through the soft-delete/recovery cycle.

- **Reduced operational overhead**: By preventing accidental deletions from turning into major incidents, soft delete saves engineering time and effort.

- **Complements backup and DR strategies**: Soft delete covers the short-term recovery for accidental deletes, while existing backup and point-in-time restore features cover other scenarios like data corruption or disaster recovery. Together, they provide a more complete data protection story. Notably, if a resource is soft deleted, you would use the soft-delete recovery path rather than backups. In fact, the system will not allow restoring a backup over an existing soft-deleted resource – it must be purged first, ensuring the soft-delete mechanism is the first line of defense.

## How soft delete works

### Enabling soft delete

The feature is enabled at the Azure Cosmos DB account level along with the Retention policy. Once soft delete is turned on for an account, all deletions of databases or containers in that account will use soft-delete behavior. You cannot target specific databases or containers to have different settings – it’s an account-wide setting. If soft delete is not enabled on an account, deletions operate normally i.e immediate permanent deletion.

When soft delete is enabled, deleting a Cosmos database account marks the entire account and all child resources as soft-deleted. Deleting a Database within a
soft delete enabled account will soft-delete that database and all containers within it. Deleting an individual Container (collection/graph/table) will soft-delete just that container. This cascading ensures no orphaned resources remain and that a restore of a parent (like an account or database) can bring back all its children except in cases where child resources were soft deleted before the parent resource was soft deleted.

### Soft-deleted state

Once a resource (account, database, or container) is soft-deleted, it enters a retained but inaccessible state as listed below:

- The resource is not listed among active resources. For example, a soft-deleted container will not appear when listing containers in its database.

- All data-plane operations are disabled. Any attempt to access the data will behave as if the resource doesn’t exist. Applications will receive 404 Not Found
errors for queries or updates on a soft-deleted container, identical to the scenario of a permanently deleted one. This ensures the soft-deleted data is not inadvertently used or modified while pending deletion.

- Internally, however, Azure Cosmos DB retains all the data and metadata for the resource. The deletion is logical – the underlying data files and configurations are
preserved for the duration of the retention period. The soft-deleted resource is essentially hidden from the user’s perspective but still present in the system.

## Minimum minutes before permanent delete

**Retention Period**: Soft-deleted resources remain recoverable for a configured Retention Period. By default, this is 14 days (plans to be reduced to 1 day in public preview), but administrators can adjust this setting per account. You might choose a shorter retention (minimum 3 day) for lower storage overhead or a longer period (e.g., 30 days) for extra safety. During this retention window, the resource can be restored at any time. Once the
retention period elapses, the service will automatically permanently delete (purge) the resource, irreversibly removing it through soft delete recovery. However, user can still restore backup.

- Example: If the retention is set to 14 days and a container is deleted on May 1, it will be kept until May 15. On or shortly after May 15, if not recovered, Azure Cosmos DB will purge that container and its data permanently. Between May 1 and May 15, the container can be recovered with all its content intac. or if an authorized user explicitly purges it earlier, the resource is permanently deleted.

- While a resource is in soft-deleted state, its name is reserved. You cannot create a new resource with the same name in that Cosmos account until the original is
purged. For instance, if a database named “OrdersDB” was soft-deleted, trying to create a new database called “OrdersDB” will result in an error that the name
already exists. This prevents any confusion or collision between the soft-deleted resource and new resources.

## Minimum retention before purge

**MinMinutesBeforePermanentDeletionAllowed** - Defines the minimum retention period (in minutes) before a soft-deleted resource can be permanently purged. The minimum allowed is 0 minute.

- Enforces a safety window — Purge attempts before this time elapses are rejected
- Recovery is always allowed — Resources can be recovered at any time during retention
- Applies to all levels — Containers, databases, and accounts

Example: If set to 60 minutes, a resource deleted at 2:00 PM cannot be purged until 3:00 PM

## Recovery (undelete)

At any point during the retention period, an authorized user can undelete or recover the resource. Recovering a soft-deleted resource returns it to the active state:

- The resource reappears in the account with the same URI/name and all its data, provisioned throughput, and settings as it had at the moment it was deleted. It’s
as if the deletion never happened.

- Any downstream applications can resume using the resource without change.

- Recovery is typically very fast because no data copy is needed – it’s a metadata operation to flip the resource back to active. From a user’s perspective, this is
essentially instantaneous recovery.

- The retention period and soft-delete status are cleared for that resource (if needed, you could delete it again, which would start a new soft-delete cycle).

To recover a resource, you will use Azure management tools (once the feature is broadly released).

For example, Azure CLI commands would be available for recover. of database or container. The Azure Portal will also offer a user-friendly interface (e.g., a “Recycle Bin” or recover options in Data Explorer) to select a soft-deleted item and recover it. All recover operations are protected by Azure RBAC (e.g., only Azure Cosmos DB Account Contributors or higher roles can initiate an undelete).

## Permanent deletion (purge)

If you do not recover a soft-deleted resource within the retention period, Azure Cosmos DB will automatically purge it after the time elapses. However, there may be cases where you want to permanently delete a soft-deleted resource before the retention window ends – for example, if a test resource was deleted and you’re sure you won’t need it, or for compliance reasons you need it gone immediately. The soft delete feature supports manual purge of a soft-deleted resource:

- An authorized user can issue a Purge command (via CLI, PowerShell, or Portal) to immediately hard-delete the resource even if it’s still within the retention period.
- Purging will free up the resource name and stop any further billing for that resource.
- Once purged, the data is unrecoverable except through restoring from backup.

It’s worth noting that all delete operations go through the soft-delete path when the feature is enabled. There is no way to “bypass” soft delete via a direct hard delete unless you disable the feature entirely on the account or perform a purge after soft deletion. This ensures consistency: any deletion is reversible by default (within the time window) and only becomes final by explicit choice (waiting out retention or purging).

## Interactions and additional details

### Backup restore conflict

If you attempt to restore a backup of a resource that is currently soft-deleted, the system will not allow it. You must either restore the soft-deleted resource directly or permanently delete it first. This is to prevent, for example, restoring an older backup on top of a retained newer copy, which could create inconsistencies. In practice, if a resource is soft-deleted, using the soft-delete recovery is the preferred approach. Backups come into play if the data itself is corrupted or if the deletion wasn’t noticed until after retention expired.

### Billing and costs

Enabling soft delete does not incur any additional service fees – it’s a built-in feature. However, while a resource is soft-deleted, it continues to be billed as if it were active. This is because the resource is still consuming storage (and potentially the reserved throughput is still allocated). For example, if you soft delete a container that had 10,000 RU/s provisioned, those RU/s remain allocated (and charged) until the container is purged. This is important to understand: soft delete is for protection, not cost savings. Administrators should purge soft-deleted resources when they are confident the data is no longer needed, to avoid unnecessary charges. This feature thus offers safety at the cost of some temporary resource usage overhead.

### Performance impact

There is no significant performance impact on an account with soft delete enabled, except for the slight overhead when a deletion occurs.
Normal operations on existing data are unaffected. The system’s background tasks handle the retention and purge, which are designed to be low-priority and not
interfere with foreground workloads.

### Security and access control

Only users with sufficient privileges (for example, Azure Cosmos DB Account Contributor or Owner roles) can soft-delete or restore resources. Soft-deleted data is not accessible to any read or write operations, so it remains secure in that interim state.

### Management interfaces

Soft delete can be managed through all standard Azure interfaces:

- **Azure portal**: UI to enable/disable the feature at the account level and to enumerate and restore soft-deleted resources (e.g., a “Deleted Items”
list with the option to recover). This feature is not available in the gated preview.

- **Azure CLI / PowerShell**: Commands to configure retention, list soft-deleted resources, and invoke restore or purge operations will be available (for
scripting and automation scenarios).

- **Azure Resource Manager (REST API)**: New properties on the Azure Cosmos DB account resource for soft delete settings (like retention duration) and new
APIs to list and restore deleted resources are being introduced.

## Frequently asked questions

### What is soft delete in Azure Cosmos DB?

Soft delete is a feature that retains deleted Azure Cosmos DB resources (accounts, databases, containers) for a configurable retention period (default 14 days (plans to change it to 1 day), allowing recovery before permanent deletion.

### How do I enable soft delete?

Soft delete is enabled at the Azure Cosmos DB account level. Once enabled, all deletions within the account follow soft-delete behavior. In preview, enablement is managed by the Azure Cosmos DB product team.

### Can I recover a deleted container or database?

**Yes**. During the retention period, authorized users can restore soft-deleted resources to their original state. In preview, recovery must be requested through the Azure Cosmos DB team.

### What happens after the retention period ends?

If a soft-deleted resource is not recovered within the retention period, it is automatically purged and permanently deleted.

### Can I purge a soft-deleted resource before the retention period ends?

**Yes**. Authorized users can manually purge resources. However, if a minimum retention policy is configured, purge may be restricted until that period has passed.

### Are soft-deleted resources billed?

**Yes**. Soft-deleted resources continue to incur charges for provisioned throughput and storage until they are purged.

### Can I create a new resource with the same name as a soft-deleted one?

No. Resource names are reserved during the retention period. You must purge the soft-deleted resource before reusing its name.

### Is soft delete available for all APIs?

**Yes**. Soft delete supports SQL, MongoDB, Cassandra, Gremlin, and Table APIs.

### Is soft delete available in the Azure portal?

**Yes**. Soft delete is available in the Azure portal.

### Can I configure the retention period?

**Yes**. The retention period is configurable per account, from 1 to 30 days. In private preview, this setting is fixed and managed by the Azure Cosmos DB team.

## Related content

- [Back up and restore introduction](online-backup-and-restore.md)
- [Continuous backup and restore](continuous-backup-restore-introduction.md)
- [Prevent changes or deletion with resource locks](resource-locks.md)
