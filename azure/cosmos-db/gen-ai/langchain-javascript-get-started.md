---
title: Get started with LangChain JS/TS integration
titleSuffix: Azure Cosmos DB
description: Learn how to integrate Azure Cosmos DB with LangChain JS/TS to build LLM applications with vector search and retrieval-augmented generation (RAG).
author: sajeetharan
ms.author: sasinnat
ms.service: azure-cosmos-db
ms.topic: tutorial
ms.date: 06/15/2026
ms.collection:
  - ce-skilling-ai-copilot
appliesto:
  - ✅ NoSQL
ai-usage: ai-assisted
---

# Get started with the LangChain JS/TS integration for Azure Cosmos DB

You can integrate Azure Cosmos DB vector search with [LangChain](https://www.langchain.com/) to build LLM-powered applications and implement retrieval-augmented generation (RAG). This tutorial demonstrates how to use the [`@langchain/azure-cosmosdb`](https://www.npmjs.com/package/@langchain/azure-cosmosdb) package to perform vector search and build a RAG implementation with Azure Cosmos DB as the vector store.

In this tutorial, you:

1. Set up the environment.
1. Store documents in Azure Cosmos DB.
1. Perform vector search queries.
1. Implement RAG to answer questions on your data.

## Background

LangChain is an open-source framework that simplifies the creation of LLM applications through composable components. By integrating Azure Cosmos DB with LangChain, you can use Azure Cosmos DB as a vector database to store and retrieve semantically similar documents using vector search.

The `@langchain/azure-cosmosdb` package provides:

- `AzureCosmosDBNoSQLVectorStore` — vector store for similarity search
- `AzureCosmosDBNoSQLSemanticCache` — semantic cache for LLM responses
- `AzureCosmosDBNoSQLChatMessageHistory` — persistent chat history

## Prerequisites

- An Azure Cosmos DB account with the vector search feature enabled. If you don't have one, [create a free account](https://azure.microsoft.com/free/) and then enable vector search:
  1. In the Azure portal, go to your Azure Cosmos DB resource page.
  1. Under **Settings**, select **Features**.
  1. Select **Vector Search for NoSQL API** and then select **Enable**.

  Alternatively, enable the feature by using the Azure CLI:

    ```azurecli
    az cosmosdb update \
        --resource-group <resource-group-name> \
        --name <account-name> \
        --capabilities EnableNoSQLVectorSearch
    ```

  The registration request is autoapproved, but might take 15 minutes to take effect. For more information, see [Enable the vector indexing and search feature](/azure/cosmos-db/vector-search#enable-the-vector-indexing-and-search-feature).
- One of the following model providers:
  - An Azure OpenAI account with deployments for embeddings (such as `text-embedding-3-small`) and completions (such as `gpt-4o`).
  - A [Microsoft AI Foundry](https://ai.azure.com/) project with model deployments for embeddings and completions.
- [Node.js](https://nodejs.org/) (v18 or later) and npm installed.
- A terminal and code editor.

## Set up the environment

Set up the environment for this tutorial by initializing a Node.js project and installing the required packages.

1. Initialize your Node.js project. Run the following commands in your terminal to create a new directory and initialize your project:

    ```bash
    mkdir langchain-cosmosdb
    cd langchain-cosmosdb
    npm init -y
    ```

1. Install dependencies. Run the following command:

    ```bash
    npm install @langchain/azure-cosmosdb @langchain/core @langchain/openai @langchain/textsplitters
    ```

1. Update your `package.json` file. Configure your project to use [ES modules](https://nodejs.org/api/esm.html) by adding `"type": "module"` to your `package.json` file. Open `package.json` and add the highlighted line:

    ```json
    {
      "name": "langchain-cosmosdb",
      "version": "1.0.0",
      "type": "module",
      "dependencies": {
        ...
      }
    }
    ```

    > [!IMPORTANT]
    > Add `"type": "module"` as a new field in your existing `package.json`. Don't replace the entire file contents - the file must remain valid JSON with all existing fields intact.

1. Set environment variables. Configure the following environment variables with your credentials before running the application:

    | Variable | Description |
    | --- | --- |
    | `AZURE_OPENAI_API_KEY` | Your Azure OpenAI or AI Foundry API key |
    | `AZURE_OPENAI_API_INSTANCE_NAME` | The subdomain of your Azure OpenAI endpoint (for example, `my-resource` from `https://my-resource.openai.azure.com`) |
    | `AZURE_OPENAI_API_VERSION` | The API version (for example, `2024-06-01`) |
    | `AZURE_COSMOSDB_NOSQL_CONNECTION_STRING` | Your Azure Cosmos DB connection string |

    ```bash
    export AZURE_OPENAI_API_KEY="<your-azure-openai-key>"
    export AZURE_OPENAI_API_INSTANCE_NAME="<your-azure-openai-instance>"
    export AZURE_OPENAI_API_VERSION="2024-06-01"
    export AZURE_COSMOSDB_NOSQL_CONNECTION_STRING="<your-cosmosdb-connection-string>"
    ```

    For local development, you can store these values in a `.env` file at the project root and load them by using the `dotenv` package:

    ```bash
    npm install dotenv
    ```

    Create a `.env` file:

    ```text
    AZURE_OPENAI_API_KEY=<your-azure-openai-key>
    AZURE_OPENAI_API_INSTANCE_NAME=<your-azure-openai-instance>
    AZURE_OPENAI_API_VERSION=2024-06-01
    AZURE_COSMOSDB_NOSQL_CONNECTION_STRING=<your-cosmosdb-connection-string>
    ```

    Then add this line at the top of `get-started.js`:

    ```javascript
    import "dotenv/config";
    ```

    > [!WARNING]
    > Never commit your `.env` file to source control. Add `.env` to your `.gitignore` file to prevent accidental exposure of secrets. For production workloads, use Azure Managed Identity or a secret manager instead of API keys.

    - **Azure OpenAI**: Use your API key and instance name from the Azure portal under your Azure OpenAI resource.
    - **AI Foundry**: Use the project API key and endpoint from the [AI Foundry portal](https://ai.azure.com/). Set `AZURE_OPENAI_API_INSTANCE_NAME` to your project endpoint (for example, `https://<your-project>.services.ai.azure.com/openai/deployments`). Find your key under your project's **Overview** > **Endpoints and keys** section.

    > [!TIP]
    > You can find your Azure Cosmos DB connection string in the Azure portal under **Settings** > **Keys** for your account.

1. Create a file named `get-started.js` and paste the following code. This code imports required packages, reads configuration from environment variables, and configures the connection to Azure Cosmos DB.

    ```javascript
    import { AzureCosmosDBNoSQLVectorStore, AzureCosmosDBNoSQLSearchType } from "@langchain/azure-cosmosdb";
    import { AzureChatOpenAI, AzureOpenAIEmbeddings } from "@langchain/openai";
    import { ChatPromptTemplate } from "@langchain/core/prompts";
    import { StringOutputParser } from "@langchain/core/output_parsers";
    import { RunnableSequence } from "@langchain/core/runnables";
    import { RecursiveCharacterTextSplitter } from "@langchain/textsplitters";
    import { Document } from "@langchain/core/documents";
    import { readFileSync } from "fs";

    // Azure OpenAI configuration (reads from environment variables)
    const azureOpenAIConfig = {
      azureOpenAIApiKey: process.env.AZURE_OPENAI_API_KEY,
      azureOpenAIApiInstanceName: process.env.AZURE_OPENAI_API_INSTANCE_NAME,
      azureOpenAIApiVersion: process.env.AZURE_OPENAI_API_VERSION,
    };
    ```

## Store documents in Azure Cosmos DB

In this section, you load documents, split them into chunks, and store them in Azure Cosmos DB as vector embeddings.

Add the following code to your `get-started.js` file:

```javascript
async function run() {
  try {
    // Load and split documents
    const text = readFileSync("./sample-data.txt", "utf-8");
    const rawDocuments = [new Document({ pageContent: text })];

    const splitter = new RecursiveCharacterTextSplitter({
      chunkSize: 1000,
      chunkOverlap: 100,
    });
    const documents = await splitter.splitDocuments(rawDocuments);

    console.log(`Split into ${documents.length} document chunks.`);

    // Create the Azure Cosmos DB vector store
    const store = await AzureCosmosDBNoSQLVectorStore.fromDocuments(
      documents,
      new AzureOpenAIEmbeddings({
        ...azureOpenAIConfig,
        azureOpenAIApiDeploymentName: "<your-embeddings-deployment-name>",
      }),
      {
        databaseName: "langchain",
        containerName: "documents",
      }
    );

    console.log("Documents stored in Azure Cosmos DB.");
```

This code performs the following actions:

- Reads raw data from a text file by using Node.js `readFileSync` and wraps it in a LangChain `Document`.
- Splits the data into smaller chunks by using a text splitter. Chunk parameters determine the size and overlap of each document.
- Creates a vector store by calling `AzureCosmosDBNoSQLVectorStore.fromDocuments`, which generates vector embeddings for each document and stores them in your Azure Cosmos DB container.

When you store documents, the method sends each text chunk to the Azure OpenAI embeddings model, which returns a numerical vector (an array of floats) representing the semantic meaning of that text. Each chunk is then saved as a JSON item in your Azure Cosmos DB container with:

- The original text content (`pageContent`)
- Any associated metadata
- The vector embedding array used for similarity search

This approach keeps your text and its vector representation colocated in a single database, eliminating the need for a separate vector store.

Create a file named `sample-data.txt` in your project directory with the following content:

```text
Azure Cosmos DB is a fully managed NoSQL database service that provides guaranteed single-digit millisecond response times and 99.999-percent availability, backed by SLAs, with automatic and instant scalability.

Key features of Azure Cosmos DB include turnkey global distribution, automatic indexing of all data without requiring schema or index management, and support for multiple data models including document, key-value, graph, and column-family through a single backend.

Azure Cosmos DB supports multiple consistency levels: strong, bounded staleness, session, consistent prefix, and eventual. This flexibility allows developers to make precise tradeoffs between consistency, availability, latency, and throughput for their applications.

The service offers turnkey global distribution with single-digit millisecond latency at the 99th percentile anywhere in the world. You can add or remove regions at any time with a click of a button, and Azure Cosmos DB will seamlessly replicate your data to all associated regions.

Azure Cosmos DB provides comprehensive SLAs covering throughput, latency, availability, and consistency. It is the only database service that offers 99.999% availability SLA for both reads and writes, making it ideal for mission-critical applications.

For AI and machine learning workloads, Azure Cosmos DB offers integrated vector search capabilities. You can store vector embeddings alongside your operational data in the same database, eliminating the need for a separate vector database and reducing architectural complexity.

Vector search in Azure Cosmos DB supports high-dimensional vectors with multiple similarity metrics, including cosine, dot product, and Euclidean distance. For scoped searches over logical partitions up to roughly 50,000 to 100,000 vectors, quantizedFlat provides a simple, efficient indexing option. For larger-scale searches across 50,000 or more vectors, DiskANN delivers low-latency, high-recall vector queries with performance designed to scale.

Vector search can also be scoped by partition key, or by a vector index shard key for DiskANN indexes, making it well-suited for multi-tenant applications that need isolation, scale, and predictable performance.

Azure Cosmos DB integrates natively with Azure AI services, LangChain, Semantic Kernel, and other AI frameworks. This makes it easy to build retrieval-augmented generation (RAG) applications that combine your operational data with large language models.

The serverless capacity mode in Azure Cosmos DB lets you pay only for the request units consumed and storage used by your database operations, making it cost-effective for development, testing, and intermittent workloads.

Azure Cosmos DB supports change feed, which provides a sorted list of documents that were changed in the order in which they were modified. The change feed can be used to build event-driven architectures, maintain materialized views, and synchronize data across services.
```

## Run vector search queries

Now that your data is stored, you can perform vector search queries. Add the following code inside the `try` block of your `run` function:

### Semantic search

Basic semantic search returns documents ranked by relevance to your query.

```javascript
    // Perform a similarity search
    const searchResults = await store.similaritySearch(
      "What are the key features?",
      4
    );

    console.log("Semantic search results:");
    for (const doc of searchResults) {
      console.log(`- ${doc.pageContent.substring(0, 100)}...`);
    }
```

### Vector search with score threshold

Filter results based on a minimum similarity score.

```javascript
    // Vector search with score threshold
    const scoredResults = await store.similaritySearchWithScore(
      "What are the key features?",
      10,
      {
        searchType: AzureCosmosDBNoSQLSearchType.VectorScoreThreshold,
        threshold: 0.8,
      }
    );

    console.log("Results with scores:");
    for (const [doc, score] of scoredResults) {
      console.log(`  Score: ${score}, Content: ${doc.pageContent.substring(0, 80)}...`);
    }
```

### Maximal Marginal Relevance (MMR) search

MMR search balances relevance with diversity in the results.

```javascript
    // MMR search for diverse results
    const mmrResults = await store.maxMarginalRelevanceSearch(
      "What are the key features?",
      {
        k: 5,
        fetchK: 20,
        lambda: 0.5, // 0 = max diversity, 1 = max relevance
      }
    );

    console.log("MMR search results:");
    for (const doc of mmrResults) {
      console.log(`- ${doc.pageContent.substring(0, 100)}...`);
    }
```

## Implement RAG to answer questions on your data

Use Azure Cosmos DB vector search with an LLM to generate context-aware responses. Add the following code inside the `try` block:

```javascript
    // Set up the LLM
    const model = new AzureChatOpenAI({
      ...azureOpenAIConfig,
      azureOpenAIApiDeploymentName: "<your-completions-deployment-name>",
      temperature: 0,
    });

    // Create a retriever from the vector store
    const retriever = store.asRetriever();

    // Helper to format documents as context string
    const formatDocs = (docs) => docs.map((d) => d.pageContent).join("\n\n");

    // Create a prompt template
    const prompt = ChatPromptTemplate.fromMessages([
      [
        "system",
        "Answer the user's questions based on the below context:\n\n{context}",
      ],
      ["human", "{question}"],
    ]);

    // Build the RAG chain
    const chain = RunnableSequence.from([
      {
        context: async (input) => {
          const docs = await retriever.invoke(input.question);
          return formatDocs(docs);
        },
        question: (input) => input.question,
      },
      prompt,
      model,
      new StringOutputParser(),
    ]);

    // Ask a question
    const question = "What are the main benefits described in the document?";
    const answer = await chain.invoke({ question });

    console.log(`\nQuestion: ${question}`);
    console.log(`Answer: ${answer}`);

    // Retrieve and display source documents
    const sourceDocs = await retriever.invoke(question);
    console.log("\nSource documents:");
    for (const doc of sourceDocs) {
      console.log(`- ${doc.pageContent.substring(0, 100)}...`);
    }
```

This code:

- Creates a retriever from the vector store to find semantically similar documents.
- Defines a prompt template that instructs the LLM to answer questions using the retrieved context.
- Builds a chain using `RunnableSequence` that retrieves documents, formats them as context, and passes the result to the LLM.
- Invokes the chain with a sample question and displays the answer along with source documents.

## Complete the function

Add the following code to close the `try` block and run the function:

```javascript
    // Clean up (optional - remove if you want to keep the data)
    // await store.delete();
  } catch (error) {
    console.error("Error:", error);
  }
}

run();
```

## Run the application

Before running, confirm that your project directory contains the following files:

- `get-started.js` — the main application file with all the code from this tutorial
- `sample-data.txt` — a text file with content you want to search and query
- `package.json` — with `"type": "module"` added

1. Save the file and run the following command to execute your application:

    ```bash
    node get-started.js
    ```

1. Review the output. You should see output similar to the following:

    ```output
    Split into 12 document chunks.
    Documents stored in Azure Cosmos DB.
    Semantic search results:
    - Azure Cosmos DB is a fully managed NoSQL database service that provides guarant...
    - Azure Cosmos DB supports multiple data models including document, key-value, gr...
    Results with scores:
      Score: 0.92, Content: Azure Cosmos DB is a fully managed NoSQL database service...
      Score: 0.87, Content: Azure Cosmos DB supports multiple data models including do...
    MMR search results:
    - Azure Cosmos DB is a fully managed NoSQL database service that provides guarant...
    - The service offers turnkey global distribution with single-digit millisecond la...

    Question: What are the main benefits described in the document?
    Answer: The main benefits include guaranteed single-digit millisecond latency,
    turnkey global distribution, automatic scaling, and support for multiple data models.

    Source documents:
    - Azure Cosmos DB is a fully managed NoSQL database service that provides guarant...
    - The service offers turnkey global distribution with single-digit millisecond la...
    ```

    > [!NOTE]
    > The exact output depends on the content of your `sample-data.txt` file. The generated answer might vary between runs.

1. Verify the stored data. In the Azure portal, navigate to your Azure Cosmos DB account and open **Data Explorer**. You should see the `langchain` database with a `documents` container containing your vectorized document chunks.

## Use Azure Managed Identity (recommended)

For production workloads, use Azure Managed Identity instead of connection strings for authentication:

```javascript
import { AzureCosmosDBNoSQLVectorStore } from "@langchain/azure-cosmosdb";
import { AzureOpenAIEmbeddings } from "@langchain/openai";

const store = new AzureCosmosDBNoSQLVectorStore(new AzureOpenAIEmbeddings(), {
  endpoint: "https://<your-account>.documents.azure.com:443/",
  databaseName: "langchain",
  containerName: "documents",
});
```

> [!IMPORTANT]
> When using Azure Managed Identity and role-based access control, you must ensure that the database and container exist. RBAC doesn't provide permissions to create databases and containers. For more information, see [set up role-based access control](/azure/cosmos-db/how-to-setup-rbac#permission-model).

## Complete sample

To get the complete working application with all the code from this tutorial combined into a single file, clone the GitHub sample:

```bash
git clone https://github.com/AzureCosmosDB/langchainjs-cosmosdb-rag-quickstart.git
cd langchainjs-cosmosdb-rag-quickstart
npm install
```

Set the required environment variables as described in [Set up the environment](#set-up-the-environment), and then run `node get-started.js`.

## Related content

- [Azure Cosmos DB integrations for AI applications](integrations.md)
- [Vector search in Azure Cosmos DB](vector-search-overview.md)
- [LangChain JS Azure Cosmos DB documentation](https://docs.langchain.com/oss/javascript/integrations/vectorstores/azure_cosmosdb_nosql)
- [Azure Cosmos DB + Azure OpenAI Node.js Developer Guide](https://github.com/AzureCosmosDB/Azure-OpenAI-Node.js-Developer-Guide)
- [What is retrieval-augmented generation (RAG)?](rag.md)
