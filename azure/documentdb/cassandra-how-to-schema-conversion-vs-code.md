---
title: Convert a Cassandra schema to Azure DocumentDB using the Schema Migrator extension
description: Use the Azure DocumentDB Schema Migrator extension in Visual Studio Code to convert Apache Cassandra schemas, SOLR indexes, and query patterns to Azure DocumentDB.
author: fonsecamar
ms.author: marcrod
ms.topic: how-to
ms.date: 06/16/2026
ai-usage: ai-assisted
---

# Convert a Cassandra schema to Azure DocumentDB by using the Schema Migrator extension

In this guide, you use the [Azure DocumentDB Schema Migrator](https://github.com/fonsecamar/azure-documentdb-schema-migrator) extension in Visual Studio Code to convert an Apache Cassandra schema to Azure DocumentDB. The extension uses AI to analyze your Cassandra schema files, including SOLR search configurations and query patterns, and generates an Azure DocumentDB data model with collection definitions, shard key recommendations, indexes, and translated queries.

The extension supports two AI providers: GitHub Copilot (through your existing Copilot subscription) and Azure OpenAI (using your own resource and API key).

## Prerequisites

- [Visual Studio Code](https://code.visualstudio.com/) 1.85.0 or higher.
- One of the following AI providers:
  - **GitHub Copilot**: an active Copilot subscription with the VS Code Copilot extension installed.
  - **Azure OpenAI**: an Azure OpenAI resource with a deployed model, such as `gpt-4o`.
- Exported Cassandra schema files from your source cluster. See [Export your schema files](#export-your-schema-files) for the file types supported.

## Export your schema files

Before you start, export the schema definitions and query patterns from your Cassandra cluster. The extension accepts the following file types:

| File type | Description |
| :-- | :-- |
| CQL schema files | Files containing `CREATE TABLE`, `CREATE TYPE`, and `CREATE INDEX` statements |
| SOLR configuration files | `schema.xml` files |
| DML files | Files containing DML statements |

You can import individual files or entire folders containing a mix of these file types. When you analyze a folder, the extension consolidates the results from all files into a single artifact.

> [!TIP]
> Export as many query examples as possible. The AI uses those patterns to suggest equivalent MongoDB Query Language (MQL) expressions and optimized indexes for Azure DocumentDB.

## Install the extension

The extension is currently available as a preview. To run it, clone the repository and launch the extension host in Visual Studio Code.

1. Clone the [azure-documentdb-schema-migrator](https://github.com/fonsecamar/azure-documentdb-schema-migrator) repository.
1. Open the cloned folder in Visual Studio Code.
1. Open a terminal and run the following command to install dependencies:

   ```bash
   npm install
   ```

1. Press `F5` to open a new VS Code Extension Development Host window with the extension loaded.

## Create a migration project

Projects organize your schema files, analysis artifacts, and conversion output in a single workspace.

1. Open the **Azure DocumentDB Schema Migrator** panel in the VS Code Activity Bar.
1. Select **Create New Project**.
1. Enter a name for your project.
1. Select your AI provider and configure it:
   - **GitHub Copilot** (recommended): select the model you want to use from the list of available Copilot models.
   - **Azure OpenAI**: enter your endpoint URL, deployment name, and API key. The extension stores the API key securely in VS Code's secret storage.
1. Select **Create Project**.

## Import schema files

1. Expand your project in the **Explorer** panel.
1. Select **Import Schema Files** in the **Schema Files** section.
1. Select the exported schema folder from your file system.

The imported files appear under the **Schema Files** section of your project, organized using the same folder hierarchy as your source selection.

## Analyze schema files

The extension analyzes each imported file or folder and extracts entities, indexes, and query patterns as artifacts.

In the **Schema Files** section, right-click a file or folder and select **Analyze File** or **Analyze Folder**.

The extension detects each file type automatically and applies the appropriate analysis:

| Source file type | Artifacts generated |
| :-- | :-- |
| CQL schema file | Schema, indexes, and queries |
| SOLR configuration file (`schema.xml`) | Indexes only |
| DML file | Queries only |
| Mixed folder | Consolidated schema from CQL files, and consolidated indexes and queries from all files |

> [!NOTE]
> User-defined types (UDTs) declared with `CREATE TYPE` specify the name and data type of nested attributes. The extension expands UDTs inline into the tables that reference them, preserving the original field names. Anonymous nested types (such as collections of tuples) are translated with generic field names like `field1`, `field2`. UDT definitions appear in the artifact for reference but aren't converted directly to Azure DocumentDB collections.

Analysis results appear under the **Artifacts** section of your project. Each artifact is labeled with its type: **Schema**, **Indexes**, or **Queries**.

## Convert to Azure DocumentDB

After you add at least one artifact, run the conversion to generate a full Azure DocumentDB migration schema.

1. In the **Generated Artifacts** section, right-click the section header.
1. Select **Convert to Azure DocumentDB**.

The extension runs a multistage AI analysis:

1. **Schema modeling**: maps Cassandra tables to Azure DocumentDB collections, determines shard keys, and recommends embedding strategies for related entities (UDTs).
1. **Query translation**: converts CQL and SOLR query patterns to MQL equivalents.
1. **Index recommendations**: generates compound and partial index definitions following the ESR rule (Equality, Sort, Range).
1. **Index optimization**: consolidates redundant or duplicated indexes and validates the final index set.

The conversion output is saved as a JSON file under the **Conversion Output** section of your project and includes:

- Collection definitions with shard key selections.
- Index recommendations for all queryable fields.
- Translated query patterns in MQL.

The extension uses this file as input for the next step, which generates the MongoDB shell scripts to apply the schema to your cluster.

> [!IMPORTANT]
> Review all AI-generated recommendations before applying them to a production cluster. Validate shard key choices against your actual query patterns and data distribution requirements.

## Generate MongoDB shell scripts

After the conversion finishes, generate ready-to-run scripts to apply the schema to your Azure DocumentDB cluster.

1. In the **Conversion Output** section, right-click the generated output file.
1. Select **Generate Mongo Shell Scripts**.

The extension creates MongoDB shell scripts that:

- Create collections with the recommended shard keys.
- Create all recommended indexes.
- Include a JSON document schema template showing the expected document structure for each collection.

Apply these scripts to your Azure DocumentDB cluster by using the [MongoDB shell](how-to-connect-mongo-shell.md) or the [DocumentDB for Visual Studio Code](how-to-connect-visual-studio-code-extension.md) extension.

## Related content

- [Cassandra migration benefits](cassandra-migration-benefits.md)
- [Create indexes in Azure DocumentDB](how-to-create-indexes.md)
- [Connect to Azure DocumentDB using the MongoDB shell](how-to-connect-mongo-shell.md)
- [Partitioning in Azure DocumentDB](partitioning.md)
- [Indexing in Azure DocumentDB](indexing.md)
