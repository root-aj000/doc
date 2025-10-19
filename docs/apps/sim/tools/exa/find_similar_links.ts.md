This TypeScript file defines a configuration object for an **Exa AI "Find Similar Links" tool**. In essence, it's a blueprint that tells a larger system how to interact with the Exa AI API to find webpages similar to a given URL.

Think of it like a detailed instruction manual for a specific feature:
*   **What it does:** Find similar links using Exa AI.
*   **What information it needs:** A URL, optionally how many results, if full text is desired, and an Exa API key.
*   **How to talk to the Exa API:** The specific URL, method, headers, and body format for the API request.
*   **How to understand the Exa API's response:** How to parse the data it sends back.
*   **What output you'll get:** The structured list of similar links.

This standardized approach allows a system (e.g., an AI agent framework, a UI builder, or a backend service) to discover, call, and manage various external APIs (like Exa AI) in a uniform way without needing to hardcode the specifics for each one.

---

## Detailed Explanation

Let's break down the code section by section.

### 1. Imports

```typescript
import type { ExaFindSimilarLinksParams, ExaFindSimilarLinksResponse } from '@/tools/exa/types'
import type { ToolConfig } from '@/tools/types'
```

*   **`import type { ExaFindSimilarLinksParams, ExaFindSimilarLinksResponse } from '@/tools/exa/types'`**:
    *   This line imports two **type definitions** from a local file (`@/tools/exa/types`).
    *   `ExaFindSimilarLinksParams`: Describes the expected structure for the *input parameters* when calling the Exa `findSimilar` API.
    *   `ExaFindSimilarLinksResponse`: Describes the expected structure for the *response data* received from the Exa `findSimilar` API.
    *   The `type` keyword before `import` indicates that these are only imported for type-checking purposes and won't generate any JavaScript code at runtime, keeping the bundle size small.
*   **`import type { ToolConfig } from '@/tools/types'`**:
    *   This imports the `ToolConfig` type definition from a more general types file (`@/tools/types`).
    *   `ToolConfig` is likely a generic interface that defines the common structure for *any* tool configuration within this project. It sets up the contract for what properties a tool configuration object must have (like `id`, `name`, `params`, `request`, etc.).

### 2. Tool Definition (`findSimilarLinksTool`)

```typescript
export const findSimilarLinksTool: ToolConfig<
  ExaFindSimilarLinksParams,
  ExaFindSimilarLinksResponse
> = {
  // ... tool configuration details
}
```

*   **`export const findSimilarLinksTool`**:
    *   This declares a constant variable named `findSimilarLinksTool` and makes it available for other files to import and use (`export`).
*   **`: ToolConfig<ExaFindSimilarLinksParams, ExaFindSimilarLinksResponse>`**:
    *   This is a **type annotation**. It tells TypeScript that `findSimilarLinksTool` must conform to the `ToolConfig` interface.
    *   `ToolConfig` is a **generic type**, and here we're specializing it with `ExaFindSimilarLinksParams` (for its input parameters) and `ExaFindSimilarLinksResponse` (for its API response). This provides strong type-checking, ensuring that all parts of this tool definition correctly handle these specific types.

### 3. Basic Tool Metadata

```typescript
  id: 'exa_find_similar_links',
  name: 'Exa Find Similar Links',
  description:
    'Find webpages similar to a given URL using Exa AI. Returns a list of similar links with titles and text snippets.',
  version: '1.0.0',
```

*   **`id: 'exa_find_similar_links'`**: A unique identifier for this specific tool. Useful for programmatic lookups.
*   **`name: 'Exa Find Similar Links'`**: A human-readable name for the tool, often used in UIs or documentation.
*   **`description: '...'`**: A detailed explanation of what the tool does. This is crucial for users or AI agents to understand its purpose.
*   **`version: '1.0.0'`**: The version number of this tool configuration. Good for tracking changes and compatibility.

### 4. Input Parameters (`params`)

```typescript
  params: {
    url: {
      type: 'string',
      required: true,
      visibility: 'user-or-llm',
      description: 'The URL to find similar links for',
    },
    numResults: {
      type: 'number',
      required: false,
      visibility: 'user-only',
      description: 'Number of similar links to return (default: 10, max: 25)',
    },
    text: {
      type: 'boolean',
      required: false,
      visibility: 'user-or-llm',
      description: 'Whether to include the full text of the similar pages',
    },
    apiKey: {
      type: 'string',
      required: true,
      visibility: 'user-only',
      description: 'Exa AI API Key',
    },
  },
```

This section defines the **schema for the inputs** that this tool expects. Each property here (`url`, `numResults`, `text`, `apiKey`) describes one parameter.

*   **`url`**:
    *   **`type: 'string'`**: The parameter must be a text string.
    *   **`required: true`**: This parameter is mandatory; the tool cannot function without it.
    *   **`visibility: 'user-or-llm'`**: Indicates that either a human user or a Large Language Model (LLM) can provide this value. This suggests the tool might be integrated into an AI agent system.
    *   **`description: '...'`**: Explains what this parameter is for.
*   **`numResults`**:
    *   **`type: 'number'`**: The parameter must be a number.
    *   **`required: false`**: This parameter is optional. If not provided, a default will likely be used (as per its description: "default: 10").
    *   **`visibility: 'user-only'`**: This is a security or design constraint. Only a human user (or the system acting on their behalf) can provide this value. An LLM agent should *not* decide this parameter.
    *   **`description: '...'`**: Explains the purpose and provides default/max values.
*   **`text`**:
    *   **`type: 'boolean'`**: The parameter must be a true/false value.
    *   **`required: false`**: Optional.
    *   **`visibility: 'user-or-llm'`**: Can be provided by a user or an LLM.
    *   **`description: '...'`**: Explains its purpose (whether to get full page text).
*   **`apiKey`**:
    *   **`type: 'string'`**: A text string.
    *   **`required: true`**: Mandatory.
    *   **`visibility: 'user-only'`**: **Crucially, the API key should only be provided by the user** (or the system managing user credentials). An LLM should never be trusted to generate or handle sensitive credentials like API keys directly. This is a vital security measure.
    *   **`description: '...'`**: Self-explanatory.

### 5. API Request Configuration (`request`)

```typescript
  request: {
    url: 'https://api.exa.ai/findSimilar',
    method: 'POST',
    headers: (params) => ({
      'Content-Type': 'application/json',
      'x-api-key': params.apiKey,
    }),
    body: (params) => {
      const body: Record<string, any> = {
        url: params.url,
      }

      // Add optional parameters if provided
      if (params.numResults) body.numResults = params.numResults

      // Add contents.text parameter if text is true
      if (params.text) {
        body.contents = {
          text: true,
        }
      }

      return body
    },
  },
```

This section dictates how to construct the actual HTTP request to the Exa AI API.

*   **`url: 'https://api.exa.ai/findSimilar'`**: The target endpoint URL for the Exa AI `findSimilar` API.
*   **`method: 'POST'`**: The HTTP method to use for the request. `POST` is typical for sending data to a server.
*   **`headers: (params) => ({...})`**:
    *   This is a function that receives the tool's input parameters (`params`) and returns an object representing the HTTP headers for the request.
    *   **`'Content-Type': 'application/json'`**: Specifies that the request body will be in JSON format.
    *   **`'x-api-key': params.apiKey`**: This is how the mandatory `apiKey` provided in the tool's parameters is passed to the Exa API – as a custom HTTP header named `x-api-key`. This is a common and secure way to authenticate API requests.
*   **`body: (params) => {...}`**:
    *   This is another function that receives the tool's input parameters (`params`) and constructs the **HTTP request body** (the data sent with the `POST` request).
    *   **`const body: Record<string, any> = { url: params.url }`**: Initializes a mutable `body` object. It uses `Record<string, any>` to indicate it's an object where keys are strings and values can be of any type, providing flexibility. The mandatory `url` parameter is added immediately.
    *   **`if (params.numResults) body.numResults = params.numResults`**: This line **conditionally adds** the `numResults` parameter to the request body *only if* it was provided in the tool's input parameters (`params.numResults` is not `undefined` or `0`). This ensures optional parameters are handled correctly.
    *   **`if (params.text) { body.contents = { text: true } }`**: Similarly, if the `text` parameter is `true`, a nested `contents` object with `text: true` is added to the request body. This matches the specific structure Exa AI expects when you want to retrieve the full text content of similar pages.
    *   **`return body`**: The fully constructed request body is returned.

### 6. Response Transformation (`transformResponse`)

```typescript
  transformResponse: async (response: Response) => {
    const data = await response.json()

    return {
      success: true,
      output: {
        similarLinks: data.results.map((result: any) => ({
          title: result.title || '',
          url: result.url,
          text: result.text || '',
          score: result.score || 0,
        })),
      },
    }
  },
```

This asynchronous function takes the raw HTTP response received from the Exa AI API and processes it into a standardized, clean output format for the tool.

*   **`async (response: Response)`**: This is an `async` function, meaning it can use `await`. It receives a standard `Response` object from the browser's Fetch API (or Node.js equivalent).
*   **`const data = await response.json()`**:
    *   This line asynchronously parses the body of the HTTP response as JSON. The `response.json()` method returns a Promise, so `await` is used to wait for the parsing to complete and store the resulting JavaScript object in `data`.
*   **`return { success: true, output: { ... } }`**:
    *   The function returns an object with a `success` flag and an `output` property. This is a common pattern for standardizing tool results.
*   **`similarLinks: data.results.map((result: any) => ({ ... }))`**:
    *   This is the core of the transformation.
    *   It assumes the Exa API response (`data`) has a `results` array.
    *   **`.map(...)`**: This array method iterates over each `result` object in the `data.results` array. For each `result`, it creates a *new* object in a specified format.
    *   **`title: result.title || ''`**: Takes the `title` from the Exa result. If `result.title` is `null` or `undefined` (or an empty string), it defaults to an empty string (`''`). This makes the output more robust and prevents potential errors if the API occasionally omits a title.
    *   **`url: result.url`**: Takes the `url` directly from the Exa result.
    *   **`text: result.text || ''`**: Similar to `title`, defaults to an empty string if `text` is missing.
    *   **`score: result.score || 0`**: Defaults the `score` to `0` if it's missing from the Exa result.
    *   This `map` operation ensures that the `similarLinks` array in the tool's output always contains objects with `title`, `url`, `text`, and `score` properties, even if the raw API response is slightly inconsistent.

### 7. Output Schema (`outputs`)

```typescript
  outputs: {
    similarLinks: {
      type: 'array',
      description: 'Similar links found with titles, URLs, and text snippets',
      items: {
        type: 'object',
        properties: {
          title: { type: 'string', description: 'The title of the similar webpage' },
          url: { type: 'string', description: 'The URL of the similar webpage' },
          text: {
            type: 'string',
            description: 'Text snippet or full content from the similar webpage',
          },
          score: {
            type: 'number',
            description: 'Similarity score indicating how similar the page is',
          },
        },
      },
    },
  },
```

This section defines the **structure and types of the data that this tool will produce** after `transformResponse` has processed the raw API output. This is essentially a public contract for what consumers of this tool can expect.

*   **`similarLinks`**: This is the top-level output property, matching the `similarLinks` key in the `transformResponse` output.
    *   **`type: 'array'`**: Specifies that `similarLinks` will be an array.
    *   **`description: '...'`**: Describes the content of the array.
    *   **`items: {...}`**: This nested object describes the structure of *each element* within the `similarLinks` array.
        *   **`type: 'object'`**: Each item in the array will be an object.
        *   **`properties: {...}`**: Defines the individual properties that each object in the `similarLinks` array will have:
            *   **`title: { type: 'string', description: '...' }`**: Each item will have a `title` (string).
            *   **`url: { type: 'string', description: '...' }`**: Each item will have a `url` (string).
            *   **`text: { type: 'string', description: '...' }`**: Each item will have `text` content (string).
            *   **`score: { type: 'number', description: '...' }`**: Each item will have a `score` (number).
    *   This detailed schema is invaluable for validating the tool's output, generating documentation, or enabling other systems to automatically process the results.

---

In summary, this `findSimilarLinksTool` object is a comprehensive, self-describing configuration that encapsulates everything needed to integrate with and use Exa AI's "Find Similar" functionality in a robust, type-safe, and standardized manner within a larger application.