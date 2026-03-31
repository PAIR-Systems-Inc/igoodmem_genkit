# GoodMem plugin for Genkit

GoodMem is a memory layer for AI agents with support for semantic storage, retrieval, and summarization. This package exposes GoodMem operations as Genkit tools that can be used with any Genkit agent or flow.

## Prerequisites

### 1. Install GoodMem

You need a running GoodMem instance. Install it on your VM or local machine:

**Visit:** [https://goodmem.ai/](https://goodmem.ai/)

Follow the installation instructions for your platform (Docker, local installation, or cloud deployment).

### 2. Create an Embedder

Before you can create spaces and memories, you need to set up an embedder model in your GoodMem instance.

### 3. Get Your API Key

Obtain an API key from your GoodMem instance (starts with `gm_`).

## Installing the plugin

```bash
npm i --save genkitx-goodmem
```

## Using the plugin

```ts
import { genkit } from 'genkit';
import { goodmem } from 'genkitx-goodmem';

const ai = genkit({
  plugins: [
    goodmem({
      baseUrl: process.env.GOODMEM_BASE_URL || 'http://localhost:8080',
      apiKey: process.env.GOODMEM_API_KEY!,
    }),
  ],
});
```

Once the plugin is loaded, the following tools are automatically registered and available to any Genkit agent or flow:

## Available Tools

### goodmem/createSpace

Create a new space (container for memories) with configurable settings. If a space with the same name already exists, it will be reused instead of creating a duplicate.

**Input:**
- **name** (required) — Unique name for the space
- **embedderId** (required) — ID of the embedder model that converts text to vector embeddings
- **chunkSize** (default: 256) — Number of characters per chunk when splitting documents
- **chunkOverlap** (default: 25) — Overlapping characters between consecutive chunks
- **keepStrategy** (default: "KEEP_END") — Where to attach the separator when splitting ("KEEP_END", "KEEP_START", or "DISCARD")
- **lengthMeasurement** (default: "CHARACTER_COUNT") — How chunk size is measured ("CHARACTER_COUNT" or "TOKEN_COUNT")

### goodmem/createMemory

Store a document or plain text as a memory in a space. The content is automatically chunked and embedded for semantic search.

**Input:**
- **spaceId** (required) — ID of the space to store the memory in
- **filePath** (optional) — Absolute path to a file (PDF, DOCX, TXT, images, etc.). Content type is auto-detected.
- **textContent** (optional) — Plain text content. If both filePath and textContent are provided, the file takes priority.
- **source** (optional) — Where this memory came from (e.g., "google-drive", "gmail")
- **author** (optional) — The author or creator of the content
- **tags** (optional) — Comma-separated tags for categorization (e.g., "legal,research,important")
- **metadata** (optional) — Extra key-value metadata as JSON

### goodmem/retrieveMemories

Perform semantic search across one or more spaces to find relevant memory chunks. Supports advanced post-processing with reranking and LLM-generated contextual responses.

**Input:**
- **query** (required) — Natural language search query
- **spaceIds** (required) — Array of space IDs to search across
- **maxResults** (default: 5) — Limit the number of returned results
- **includeMemoryDefinition** (default: true) — Fetch full memory metadata alongside matched chunks
- **waitForIndexing** (default: true) — Retry for up to 10 seconds when no results found (useful when memories were just added)
- **rerankerId** (optional) — Reranker model ID to improve result ordering
- **llmId** (optional) — LLM ID to generate contextual responses alongside retrieved chunks
- **relevanceThreshold** (optional) — Minimum score (0-1) for including results
- **llmTemperature** (optional) — Creativity setting for LLM generation (0-2)
- **chronologicalResort** (default: false) — Reorder results by creation time instead of relevance score

### goodmem/getMemory

Retrieve a specific memory by its ID, including metadata, processing status, and optionally the original content.

**Input:**
- **memoryId** (required) — The UUID of the memory to fetch
- **includeContent** (default: true) — Fetch the original document content in addition to metadata

### goodmem/deleteMemory

Permanently delete a memory and all its associated chunks and vector embeddings.

**Input:**
- **memoryId** (required) — The UUID of the memory to delete

## Helper Functions

The plugin also exports helper functions for programmatic use:

```ts
import { listSpaces, listEmbedders } from 'genkitx-goodmem';

// List all spaces
const spaces = await listSpaces({ baseUrl: '...', apiKey: '...' });

// List all embedders
const embedders = await listEmbedders({ baseUrl: '...', apiKey: '...' });
```

## Example: Full Workflow

```ts
import { genkit, z } from 'genkit';
import { goodmem, listEmbedders } from 'genkitx-goodmem';

const ai = genkit({
  plugins: [
    goodmem({
      baseUrl: 'http://localhost:8080',
      apiKey: process.env.GOODMEM_API_KEY!,
    }),
  ],
});

// Use the tools programmatically in a flow
const memoryFlow = ai.defineFlow(
  { name: 'memoryFlow', inputSchema: z.string(), outputSchema: z.any() },
  async (query) => {
    // Create a space
    const space = await ai.runTool('goodmem/createSpace', {
      name: 'my-knowledge-base',
      embedderId: '<your-embedder-id>',
    });

    // Store a memory
    const memory = await ai.runTool('goodmem/createMemory', {
      spaceId: space.spaceId!,
      textContent: 'The capital of France is Paris.',
      source: 'manual',
    });

    // Retrieve relevant memories
    const results = await ai.runTool('goodmem/retrieveMemories', {
      query,
      spaceIds: [space.spaceId!],
    });

    return results;
  }
);
```

The sources for this package are in the main [Genkit](https://github.com/firebase/genkit) repo. Please file issues and pull requests against that repo.

Usage information and reference details can be found in [official Genkit documentation](https://genkit.dev/docs/get-started/).

License: Apache 2.0
