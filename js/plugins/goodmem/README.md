# GoodMem plugin for Genkit

GoodMem is a memory layer for AI agents with support for semantic storage, retrieval, and summarization. This package exposes GoodMem operations as Genkit tools/flows that can be used with any Genkit agent or flow.

## Prerequisites

### 1. Install GoodMem

You need a running GoodMem instance. Install it on your VM or local machine:

**Visit:** [https://goodmem.ai/](https://goodmem.ai/)

Follow the installation instructions for your platform (Docker, local installation, or cloud deployment).

### 2. Create an Embedder

Before you can create spaces and memories, you need to set up an embedder model in your GoodMem instance. You can list embedders programmatically with the `goodmem/list_embedders` tool (or the `listEmbedders()` helper).

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

Once the plugin is loaded, the following 11 tools are automatically registered and available to any Genkit agent or flow.

## Tool naming

All tools are namespaced under `goodmem/` and use snake_case action names — for example `goodmem/create_space`, `goodmem/list_embedders`. The `<plugin>/<action>` shape follows Genkit's convention; snake_case keeps action names consistent across the surface.

## Available Tools

### `goodmem/list_embedders`

List all embedder models available in the GoodMem instance.

**Input:** none.
**Output:** `{ success, embedders, totalResults }`.

### `goodmem/list_spaces`

List all spaces visible to the API key.

**Input:** none.
**Output:** `{ success, spaces, totalResults }`.

### `goodmem/get_space`

Fetch a single space by its UUID.

**Input:**
- **spaceId** (required) — UUID of the space.

**Output:** `{ success, space }`.

### `goodmem/create_space`

Create a new space (container for memories) with configurable settings. If a space with the same name already exists, it is reused instead of creating a duplicate.

**Input:**
- **name** (required) — Unique name for the space.
- **embedderId** (required) — ID of the embedder model that converts text to vector embeddings.
- **chunkSize** (default: 256) — Number of characters per chunk when splitting documents.
- **chunkOverlap** (default: 25) — Overlapping characters between consecutive chunks.
- **keepStrategy** (default: `"KEEP_END"`) — Where to attach the separator when splitting (`"KEEP_END"`, `"KEEP_START"`, or `"DISCARD"`).
- **lengthMeasurement** (default: `"CHARACTER_COUNT"`) — How chunk size is measured (`"CHARACTER_COUNT"` or `"TOKEN_COUNT"`).
- **labels** (optional) — Map of string labels to attach at creation time.

**Output:** `{ success, spaceId, name, embedderId, reused, message }`.

### `goodmem/update_space`

Update a GoodMem space. Supports renaming, toggling `publicRead`, modifying labels, and changing the default chunking config.

**Input:**
- **spaceId** (required) — UUID of the space to update.
- **name** (optional) — New name.
- **publicRead** (optional) — Whether the space is publicly readable.
- **replaceLabels** (optional) — Replace ALL existing labels with this map.
- **mergeLabels** (optional) — Merge these labels into existing labels (adds/overwrites individual keys). Mutually exclusive with `replaceLabels`.
- **defaultChunkingConfig** (optional) — New chunking config (e.g., `{ recursive: { chunkSize: 512, ... } }`).

**Output:** `{ success, space }`.

### `goodmem/delete_space`

Permanently delete a GoodMem space. All memories inside the space are deleted as well.

**Input:**
- **spaceId** (required) — UUID of the space.

**Output:** `{ success, spaceId, message }`.

### `goodmem/create_memory`

Store a document or plain text as a memory in a space. The content is automatically chunked and embedded for semantic search.

**Input:**
- **spaceId** (required) — ID of the space to store the memory in.
- **filePath** (optional) — Absolute path to a file (PDF, DOCX, TXT, images, etc.). Content type is auto-detected.
- **textContent** (optional) — Plain text content. If both `filePath` and `textContent` are provided, the file takes priority.
- **source** (optional) — Where this memory came from (e.g., `"google-drive"`, `"gmail"`).
- **author** (optional) — The author or creator of the content.
- **tags** (optional) — Comma-separated tags for categorization (e.g., `"legal,research,important"`).
- **metadata** (optional) — Extra key-value metadata as JSON.

**Output:** `{ success, memoryId, spaceId, status, contentType, fileName, message }`.

### `goodmem/list_memories`

List the memories stored in a specific GoodMem space (GoodMem scopes memory listing to a single space).

**Input:**
- **spaceId** (required) — UUID of the space.

**Output:** `{ success, spaceId, memories, totalResults }`.

### `goodmem/retrieve_memories`

Perform semantic search across one or more spaces to find relevant memory chunks. Supports advanced post-processing with reranking, LLM-generated contextual responses, score thresholds, and chronological resorting.

**Input:**
- **query** (required) — Natural language search query.
- **spaceIds** (required) — Array of space IDs to search across.
- **maxResults** (default: 5) — Limit the number of returned results.
- **includeMemoryDefinition** (default: true) — Fetch full memory metadata alongside matched chunks.
- **waitForIndexing** (default: true) — Retry for up to **10 seconds** when no results are found (useful when memories were just added). Polls every 2 seconds.
- **rerankerId** (optional) — UUID of a reranker model to improve result ordering.
- **llmId** (optional) — UUID of an LLM that generates a contextual `abstractReply` alongside the retrieved chunks.
- **relevanceThreshold** (optional, 0-1) — Minimum score for including results. Only used when `rerankerId` or `llmId` is set.
- **llmTemperature** (optional, 0-2) — Creativity setting for LLM generation. Only used when `llmId` is set.
- **chronologicalResort** (default: false) — Reorder results by creation time instead of relevance score.

The advanced options (`rerankerId`, `llmId`, `relevanceThreshold`, `llmTemperature`, `maxResults`, `chronologicalResort`) are forwarded to the GoodMem retrieve API as a `postProcessor` block:

```json
{
  "postProcessor": {
    "name": "com.goodmem.retrieval.postprocess.ChatPostProcessorFactory",
    "config": {
      "reranker_id": "...",
      "llm_id": "...",
      "relevance_threshold": 0.5,
      "llm_temp": 0.2,
      "max_results": 5,
      "chronological_resort": false
    }
  }
}
```

**Output:** `{ success, resultSetId, results, memories, totalResults, query, abstractReply?, message? }`.

### `goodmem/get_memory`

Retrieve a specific memory by its ID, including metadata, processing status, and optionally the original content.

**Input:**
- **memoryId** (required) — The UUID of the memory to fetch.
- **includeContent** (default: true) — Fetch the original document content in addition to metadata.

**Output:** `{ success, memory, content?, contentError? }`.

### `goodmem/delete_memory`

Permanently delete a memory and all its associated chunks and vector embeddings.

**Input:**
- **memoryId** (required) — The UUID of the memory to delete.

**Output:** `{ success, memoryId, message }`.

## Helper Functions

The plugin also exports two helper functions for programmatic use (the `goodmem/list_spaces` and `goodmem/list_embedders` tools call into these):

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
import { goodmem } from 'genkitx-goodmem';

const ai = genkit({
  plugins: [
    goodmem({
      baseUrl: 'http://localhost:8080',
      apiKey: process.env.GOODMEM_API_KEY!,
    }),
  ],
});

const memoryFlow = ai.defineFlow(
  { name: 'memoryFlow', inputSchema: z.string(), outputSchema: z.any() },
  async (query) => {
    // Look up an embedder
    const listEmbedders = await ai.registry.lookupAction(
      '/tool/goodmem/list_embedders'
    );
    const embeddersResp = await listEmbedders({});
    const embedderId = embeddersResp.embedders[0].embedderId;

    // Create (or reuse) a space
    const createSpace = await ai.registry.lookupAction(
      '/tool/goodmem/create_space'
    );
    const space = await createSpace({
      name: 'my-knowledge-base',
      embedderId,
    });

    // Store a memory
    const createMemory = await ai.registry.lookupAction(
      '/tool/goodmem/create_memory'
    );
    await createMemory({
      spaceId: space.spaceId,
      textContent: 'The capital of France is Paris.',
      source: 'manual',
    });

    // Retrieve relevant memories with reranking + LLM abstract reply
    const retrieve = await ai.registry.lookupAction(
      '/tool/goodmem/retrieve_memories'
    );
    return retrieve({
      query,
      spaceIds: [space.spaceId],
      rerankerId: process.env.GOODMEM_RERANKER_ID,
      llmId: process.env.GOODMEM_LLM_ID,
      relevanceThreshold: 0.3,
      llmTemperature: 0.2,
      maxResults: 5,
    });
  }
);
```

## Testing

Run the unit test suite (mocked fetch, exercises every tool):

```bash
npm run test
```

The sources for this package are in the main [Genkit](https://github.com/firebase/genkit) repo. Please file issues and pull requests against that repo.

Usage information and reference details can be found in [official Genkit documentation](https://genkit.dev/docs/get-started/).

License: Apache 2.0
