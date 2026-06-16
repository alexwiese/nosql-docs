---
title: Best practices in GitHub Copilot for Visual Studio Code
description: Enable context-aware AI assistance with Azure Cosmos DB in GitHub Copilot for Visual Studio Code. Get optimized code suggestions, query patterns, and best practices to improve your NoSQL development workflow.
author: sajeetharan
ms.author: sasinnat
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.topic: feature-guide
ms.date: 06/12/2026
ms.update-cycle: 180-days
ms.custom: sfi-image-nochange
ms.collection:
  - ce-skilling-ai-copilot
ai-usage: ai-assisted
appliesto:
  - ✅ NoSQL
---

# Azure Cosmos DB for NoSQL best practices in GitHub Copilot for Visual Studio Code

[GitHub Copilot](https://github.com/features/copilot) in Visual Studio Code provides intelligent code suggestions, but those suggestions become even more powerful when they understand your specific database context. The [Azure Cosmos DB for Visual Studio Code extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-cosmosdb) automatically installs the [Azure Cosmos DB Agent Kit](gen-ai/agent-kit.md), enabling context-aware AI assistance for your database development.

When you connect to your Azure Cosmos DB account through the Visual Studio Code extension, it automatically installs the Agent Kit, which provides GitHub Copilot with expert-level Azure Cosmos DB best practices across data modeling, partition key design, query optimization, SDK usage, and more.

## Why context-aware AI matters

AI coding assistants like GitHub Copilot are only as effective as the context they have. Without understanding how your application interacts with Azure Cosmos DB—such as partitioning strategies, indexing patterns, or query design—Copilot might generate suggestions that aren't optimized for your database setup.

With the Agent Kit automatically installed, Copilot gains immediate access to:

- **Data modeling** best practices for structuring documents and relationships
- **Partition key design** guidelines for choosing effective partition keys
- **Query optimization** techniques to reduce RU consumption and improve performance
- **SDK best practices** for proper client initialization, retry logic, and error handling
- **Vector search and full-text search** patterns for AI-powered applications
- **Indexing strategies** for efficient index policies
- **Throughput and scaling** guidance for autoscale and capacity planning

## Prerequisites

- [Visual Studio Code](https://code.visualstudio.com/) installed on your machine
- [GitHub Copilot subscription](https://github.com/features/copilot) (individual, business, or enterprise)
- An Azure Cosmos DB account (any API)
- Azure subscription with appropriate permissions to access Cosmos DB resources

## Step 1: Install required extensions

First, ensure you have both the Azure Cosmos DB extension and GitHub Copilot extension installed:

### Install the Azure Cosmos DB extension

1. Open Visual Studio Code
2. Go to the **Extensions** view by selecting **View > Extensions** or pressing **Ctrl+Shift+X** (Windows/Linux) or **Cmd+Shift+X** (macOS)
3. Search for "Azure Cosmos DB" or use the extension ID: `ms-azuretools.vscode-cosmosdb`
4. Select **Install** on the **Azure Cosmos DB** extension by Microsoft
5. Reload Visual Studio Code if prompted

### Install GitHub Copilot

1. In the **Extensions** view, search for "GitHub Copilot"
2. Install the **GitHub Copilot** extension by GitHub
3. Sign in to your GitHub account when prompted
4. Activate your Copilot subscription if you haven't already

## Step 2: Connect to your Azure Cosmos DB account

Connect your Azure Cosmos DB account to enable the context-aware features:

### For Azure Cosmos DB for NoSQL

1. In Visual Studio Code, open the **Azure** pane by selecting the Azure icon in the Activity Bar
2. Sign in to your Azure account using Microsoft Entra ID
3. In the Azure tree view, expand your subscription
4. Locate and expand **Azure Cosmos DB**
5. Right-click on your Azure Cosmos DB for NoSQL account or select it to connect

> [!NOTE]
> If you're in a corporate environment with network restrictions, you might need to configure firewall rules to allow your IP address. Consider initially allowing all IP addresses (0.0.0.0 - 255.255.255.255) for testing, then refining the allowlist for production use.

## Step 3: Verify Agent Kit installation

Once connected, the extension automatically installs the Azure Cosmos DB Agent Kit. To verify the installation:

1. Open a terminal in Visual Studio Code
1. Check that the `.agent-skills` folder exists in your workspace with the Azure Cosmos DB rules
1. Alternatively, verify that GitHub Copilot provides Azure Cosmos DB-specific suggestions when you start writing database code

> [!NOTE]
> The `.agent-skills` folder is generated locally and isn't intended to be committed to source control. Add `.agent-skills/` to your `.gitignore` file so each contributor's environment manages its own copy.

The Agent Kit provides a curated set of rules organized into prioritized categories that cover real-world Azure Cosmos DB scenarios. For the current rule inventory and what's included, see [Azure Cosmos DB Agent Kit for AI coding assistants](gen-ai/agent-kit.md).

## Step 4: Experience context-aware suggestions

Now GitHub Copilot provides Azure Cosmos DB-optimized suggestions. Here are some examples of how the enhanced context improves your development experience:

### Optimized query suggestions

When you're writing queries, Copilot now understands Azure Cosmos DB-specific patterns:

```javascript
// Copilot will suggest optimized query patterns
const query = {
    query: "SELECT * FROM c WHERE c.partitionKey = @partitionKey AND c.status = @status",
    parameters: [
        { name: "@partitionKey", value: userId },
        { name: "@status", value: "active" }
    ]
};
```

### Best practice recommendations

Copilot suggests following Azure Cosmos DB best practices for data modeling:

```javascript
// Copilot suggests embedding related data for single-partition queries
const userDocument = {
    id: userId,
    partitionKey: userId, // Copilot suggests consistent partition key usage
    profile: {
        name: "John Doe",
        email: "john@example.com",
        preferences: {
            theme: "dark",
            notifications: true
        }
    },
    // Embed frequently accessed related data
    recentActivity: [
        { action: "login", timestamp: "2024-01-01T10:00:00Z" },
        { action: "purchase", timestamp: "2024-01-01T11:30:00Z" }
    ]
};
```

### Error handling and diagnostics

Copilot suggests proper error handling patterns specific to Azure Cosmos DB:

```javascript
try {
    const response = await container.items.create(document);
    console.log("Document created successfully");
} catch (error) {
    // Copilot suggests Azure Cosmos DB-specific error handling
    if (error.code === 429) {
        console.log("Rate limited. Retry after:", error.retryAfterInMilliseconds);
        // Implement exponential backoff
    } else if (error.code === 409) {
        console.log("Document already exists");
    } else {
        console.error("Error creating document:", error.message);
        // Log diagnostic information for troubleshooting
        console.log("Diagnostic info:", response.diagnostics);
    }
}
```

## Developer benefits

This integration transforms the Azure Cosmos DB extension into more than just a management tool—it becomes an AI knowledge layer within your development environment. With context-aware assistance, you can:

✅ **Write optimized queries and SDK calls** guided by contextual best practices  
✅ **Avoid common pitfalls** early in development  
✅ **Receive suggestions aligned** with real-world Azure Cosmos DB guidance, not generic database logic  
✅ **Improve performance** through AI-suggested optimizations  
✅ **Reduce development time** with relevant, context-aware code completions

## Customizing the guidance

The Agent Kit is open-source and extensible. You can:

1. **Review the rules** to understand what context is being provided to Copilot
1. **Add project-specific rules** for your team's coding standards
1. **Contribute improvements** by submitting pull requests to the [Agent Kit GitHub repository](https://github.com/AzureCosmosDB/cosmosdb-agent-kit)
1. **Report issues or suggestions** in the [issues section](https://github.com/AzureCosmosDB/cosmosdb-agent-kit/issues)

For full documentation on Agent Kit capabilities and manual installation, see [Azure Cosmos DB Agent Kit for AI coding assistants](gen-ai/agent-kit.md).

## Best practices for maximum benefit

To get the most from this context-aware AI assistance:

### 1. Keep your extension updated
Regularly update the Azure Cosmos DB extension to receive the latest best practices and improvements.

### 2. Use descriptive variable names
Copilot provides better suggestions when your code uses clear, descriptive names that indicate Azure Cosmos DB concepts:

```javascript
// Good - Clear Azure Cosmos DB context
const cosmosClient = new CosmosClient({ endpoint, key });
const database = cosmosClient.database("productCatalog");
const container = database.container("products");

// Less optimal - Generic naming
const client = new CosmosClient({ endpoint, key });
const db = client.database("productCatalog");
const coll = db.container("products");
```

### 3. Include comments about your data model

Help Copilot provide more relevant suggestions by adding comments about your partitioning strategy and data model:

```javascript
// Partition by userId to ensure user data is co-located
// Embed user preferences to minimize cross-partition queries
const userDocument = {
    id: generateUserId(),
    partitionKey: userId, // Using userId as partition key for user isolation
    // ... rest of document
};
```

### 4. Use the extension's data management features
Use the extension's built-in features alongside Copilot for a complete development experience:

- **Query editor** for testing and optimizing queries
- **Document management** for real-time editing
- **Performance metrics** for understanding query costs
- **Export capabilities** for data analysis

## Troubleshooting

### Copilot not providing Azure Cosmos DB-specific suggestions

1. **Verify Agent Kit installation**: Check that the `.agent-skills` folder exists in your workspace with Azure Cosmos DB rules
1. **Restart Visual Studio Code**: Sometimes a restart is needed for changes to take effect
1. **Check GitHub Copilot status**: Ensure Copilot is active and properly authenticated
1. **Reconnect to Azure Cosmos DB**: Try disconnecting and reconnecting your account

### Agent Kit not installed automatically

1. **Update the extension**: Ensure you have the latest version of the Azure Cosmos DB extension
1. **Check permissions**: Verify Visual Studio Code has write permissions to the workspace folder
1. **Manual installation**: Run `npx add-skill AzureCosmosDB/cosmosdb-agent-kit` in your project directory

## Related content

- [Azure Cosmos DB Agent Kit for AI coding assistants](gen-ai/agent-kit.md)
- [Azure Cosmos DB Visual Studio Code extension](vscode-extension/overview.md)
- [Azure Cosmos DB scaling provisioned throughput best practices](scaling-provisioned-throughput-best-practices.md)
- [Contributing to the Azure Cosmos DB Visual Studio Code extension](https://github.com/microsoft/vscode-cosmosdb)
