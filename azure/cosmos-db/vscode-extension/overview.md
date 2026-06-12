---
title: Visual Studio Code extension for Azure Cosmos DB
description: Learn about the Azure Cosmos DB Visual Studio Code extension, how to install it, and how to connect, query, and manage your databases from within your editor.
author: sajeetharan
ms.author: sasinnat
ms.service: azure-cosmos-db
ms.topic: overview
ms.date: 05/12/2026
ms.update-cycle: 180-days
ai-usage: ai-assisted
---

# Visual Studio Code extension for Azure Cosmos DB

The Azure Cosmos DB extension for Visual Studio Code is a versatile tool that you can use to connect to Azure Cosmos DB accounts, browse databases and containers, query data, and manage documents directly from your editor. It supports both cloud accounts and local emulator environments so you can stay in your development workflow without switching to the Azure portal.

:::image type="content" source="media/browse-accounts-databases/browse-accounts-databases.png" alt-text="Screenshot of the Azure Cosmos DB extension showing the resource tree in Visual Studio Code." lightbox="media/browse-accounts-databases/browse-accounts-databases.png":::

## Key features

- **Browse accounts and databases**: Navigate your Azure Cosmos DB resource hierarchy directly from Visual Studio Code.
- **Query editor with syntax highlighting**: Write and run SQL queries with editor support, multiple result views (table, JSON, tree), and query metrics.
- **Document management**: Create, read, update, and delete documents with real-time editing and JSON import.
- **Local emulator support**: Connect to the Azure Cosmos DB emulator for local development and testing.
- **Import and export data**: Move data in and out of containers using JSON and CSV formats.
- **Azure portal integration**: Jump to portal views for advanced configuration and diagnostics.
- **Natural language to query (preview)**: Generate query drafts from plain language prompts.
- **Copilot Chat query assistance (preview)**: Get AI guidance to refine queries and troubleshoot query logic.
- **IntelliSense and auto-completion (preview)**: Use schema-aware suggestions to author queries faster.
- **Migration Assistant (preview)**: Get guided help when migrating relational data models to Azure Cosmos DB.

## Prerequisites

- [Visual Studio Code](https://code.visualstudio.com/docs) 1.85 or later installed on Windows, macOS, or Linux.
- An Azure Cosmos DB account configured with a database and container. Use any of these quickstarts to set up a resource:
  - [Azure portal](../quickstart-portal.md)
  - [Azure CLI](../quickstart-template-bicep.md)

## Install the extension

1. Open Visual Studio Code.
1. Select **View** > **Extensions** or use the shortcut `Ctrl+Shift+X` on Windows (`Cmd+Shift+X` on macOS).
1. In the search bar, enter **Azure Cosmos DB** and select the [Azure Cosmos DB extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-cosmosdb).
1. Select **Install**.
1. Reload Visual Studio Code if prompted.

## Connect to your account

1. In Visual Studio Code, select the **Azure** icon in the Activity Bar to open the Azure pane.
1. Sign in to your Azure account via Microsoft Entra ID.
1. In the Azure tree view, find your subscription and expand **Azure Cosmos DB**.
1. Select an existing account or right-click to create a new resource.

> [!NOTE]
> Use Microsoft Entra ID role-based access control when accessing your Azure Cosmos DB resources.

## Use cases

- **Development and testing**: Quick access to data during development without context switching.
- **Query authoring**: Write, run, and optimize queries with real-time metrics and AI assistance.
- **Data exploration**: Browse containers and inspect documents interactively.
- **Data operations**: Import, export, and manage documents for seeding, backup, and migration tasks.
- **Migration planning**: Use the Migration Assistant to map relational schemas to Azure Cosmos DB models.

## Documentation

- [Browse accounts and databases](browse-accounts-databases.md)
- [Execute SQL queries](sql-query-editor.md)
- [Manage documents and collections](manage-documents-collections.md)
- [Connect to local emulator](connect-local-emulator.md)
- [Import and export data](import-export-data.md)
- [Azure portal integration](azure-portal-integration.md)
- [Natural language to query (preview)](natural-language-to-query-preview.md)
- [Copilot Chat query assistance (preview)](copilot-chat-query-assistance-preview.md)
- [IntelliSense and auto-completion (preview)](intellisense-auto-completion-preview.md)
- [Migration Assistant (preview)](cosmos-db-migration-assistant.md)

## Related content

- [Get started with Azure Cosmos DB](../quickstart-dotnet.md)
- [Node.js quickstart](../quickstart-nodejs.md)
- [Python quickstart](../quickstart-python.md)
- [Java quickstart](../quickstart-java.md)
- [Go quickstart](../quickstart-go.md)
