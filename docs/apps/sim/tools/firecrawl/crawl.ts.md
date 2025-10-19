This TypeScript file defines a "tool configuration" for interacting with the Firecrawl API, specifically to perform a website crawl. It's designed to encapsulate all the necessary details for initiating a crawl, monitoring its progress (which is an asynchronous, long-running operation), and processing the final results.

### Purpose of this file

The primary purpose of this file is to act as a blueprint for a "Firecrawl Crawl" tool within a larger application. It specifies:

1.  **Input Parameters**: What information the user (or an AI/LLM) needs to provide to start a crawl (e.g., URL, API key).
2.  **API Request**: How to construct the initial API call to Firecrawl to start a crawling job.
3.  **Asynchronous Handling**: How to deal with Firecrawl's jobs, which don't return immediate results but rather a job ID. This involves *polling* a status endpoint until the job is complete or fails.
4.  **Output Structure**: What the final successful or failed result of the crawl will look like, including the structured content from crawled pages.
5.  **Logging and Error Handling**: Mechanisms for tracking the job's progress and managing potential issues.

In essence, it provides a robust, pre-configured way to integrate Firecrawl's powerful website crawling capabilities into another system, abstracting away the complexities of API interaction, long-running tasks, and result parsing.

### Simplifying Complex Logic: The `postProcess` Polling Loop

The most complex part of this file is the `postProcess` function. Firecrawl, like many services that perform long-running tasks (like crawling an entire website), doesn't give you the results immediately. Instead, when you initiate a crawl, it returns a "job ID". The actual crawling then happens in the background on Firecrawl's servers.

The `postProcess` function is responsible for monitoring this background job until it's finished. This is done through a technique called **polling**:

1.  **Start a Job, Get an ID**: The initial `request` and `transformResponse` steps send the crawl request to Firecrawl and get back a `jobId`.
2.  **Loop and Check**: The `postProcess` function then enters a `while` loop. In each iteration of this loop, it does the following:
    *   **Wait**: It pauses for a short duration (`POLL_INTERVAL_MS`, which is 5 seconds) to avoid hammering the API.
    *   **Ask for Status**: It sends another request (to a local proxy endpoint, `/api/tools/firecrawl/crawl/${jobId}`) to ask Firecrawl, "Hey, what's the status of job XYZ?".
    *   **Check Response**:
        *   If the job is `completed`, it fetches the final results and returns them.
        *   If the job `failed`, it logs the error and returns a failure message.
        *   If the job is still `running` (or any other status), it goes back to waiting and checking again.
    *   **Timeout**: To prevent infinite loops, there's a `MAX_POLL_TIME_MS` (5 minutes). If the job doesn't complete within this time, the polling stops, and a timeout error is returned.
3.  **Error Handling**: The entire polling process is wrapped in a `try...catch` block to gracefully handle network issues or other errors that might occur during status checks.

This polling mechanism allows the application to respond quickly after initiating the job, and then asynchronously update the user or trigger further actions once the crawl is truly finished.

### Line-by-Line Explanation

```typescript
import { createLogger } from '@/lib/logs/console/logger'
import type { FirecrawlCrawlParams, FirecrawlCrawlResponse } from '@/tools/firecrawl/types'
import type { ToolConfig } from '@/tools/types'
```
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a utility function `createLogger` from a local logging library. This function is used to create a logger instance specific to this file, enabling structured logging.
*   **`import type { FirecrawlCrawlParams, FirecrawlCrawlResponse } from '@/tools/firecrawl/types'`**: Imports TypeScript type definitions.
    *   `FirecrawlCrawlParams`: Defines the structure of the input parameters required to start a Firecrawl crawl.
    *   `FirecrawlCrawlResponse`: Defines the structure of the expected output or response after the crawl is completed.
*   **`import type { ToolConfig } from '@/tools/types'`**: Imports a generic TypeScript type `ToolConfig`. This type likely defines a standard structure for configuring various external tools or integrations within the application.

```typescript
const logger = createLogger('FirecrawlCrawlTool')
```
*   **`const logger = createLogger('FirecrawlCrawlTool')`**: Creates a logger instance with the name 'FirecrawlCrawlTool'. This logger will be used throughout this file to output informational, warning, and error messages to the console or other configured logging destinations, making it easier to debug and monitor the tool's operations.

```typescript
const POLL_INTERVAL_MS = 5000 // 5 seconds between polls
const MAX_POLL_TIME_MS = 300000 // 5 minutes maximum polling time
```
*   **`const POLL_INTERVAL_MS = 5000`**: Defines a constant for the duration (in milliseconds) to wait between checking the status of the Firecrawl job. `5000` ms equals 5 seconds.
*   **`const MAX_POLL_TIME_MS = 300000`**: Defines a constant for the maximum total duration (in milliseconds) that the system will continue to poll for a Firecrawl job's completion. `300000` ms equals 5 minutes. This prevents the system from waiting indefinitely if a job never completes.

```typescript
export const crawlTool: ToolConfig<FirecrawlCrawlParams, FirecrawlCrawlResponse> = {
  // ... tool configuration properties ...
}
```
*   **`export const crawlTool: ToolConfig<FirecrawlCrawlParams, FirecrawlCrawlResponse> = { ... }`**: This is the main declaration of the Firecrawl crawl tool configuration.
    *   `export`: Makes this `crawlTool` object available for other files to import and use.
    *   `const crawlTool`: Declares a constant variable named `crawlTool`.
    *   `: ToolConfig<FirecrawlCrawlParams, FirecrawlCrawlResponse>`: This is a TypeScript type annotation. It states that `crawlTool` must conform to the `ToolConfig` interface, specifically configured to take `FirecrawlCrawlParams` as input parameters and produce `FirecrawlCrawlResponse` as its output.

```typescript
  id: 'firecrawl_crawl',
  name: 'Firecrawl Crawl',
  description: 'Crawl entire websites and extract structured content from all accessible pages',
  version: '1.0.0',
```
*   **`id: 'firecrawl_crawl'`**: A unique identifier for this tool. Useful for programmatic access or internal referencing.
*   **`name: 'Firecrawl Crawl'`**: A human-readable name for the tool, typically displayed in a user interface.
*   **`description: 'Crawl entire websites and extract structured content from all accessible pages'`**: A brief explanation of what the tool does, intended for users or AI models that might interact with it.
*   **`version: '1.0.0'`**: The version number of this specific tool configuration.

```typescript
  params: {
    url: {
      type: 'string',
      required: true,
      visibility: 'user-or-llm',
      description: 'The website URL to crawl',
    },
    limit: {
      type: 'number',
      required: false,
      visibility: 'user-only',
      description: 'Maximum number of pages to crawl (default: 100)',
    },
    onlyMainContent: {
      type: 'boolean',
      required: false,
      visibility: 'user-only',
      description: 'Extract only main content from pages',
    },
    apiKey: {
      type: 'string',
      required: true,
      visibility: 'user-only',
      description: 'Firecrawl API Key',
    },
  },
```
*   **`params: { ... }`**: This object defines all the input parameters that can be passed to this tool. Each parameter has specific properties:
    *   **`url`**:
        *   `type: 'string'`: The `url` parameter must be a string.
        *   `required: true`: This parameter is mandatory for the tool to function.
        *   `visibility: 'user-or-llm'`: This parameter can be provided by either a human user or an AI/Large Language Model (LLM).
        *   `description: 'The website URL to crawl'`: A descriptive explanation of what the `url` parameter represents.
    *   **`limit`**:
        *   `type: 'number'`: The `limit` parameter must be a number.
        *   `required: false`: This parameter is optional.
        *   `visibility: 'user-only'`: Only a human user can provide this parameter; an LLM is not expected to generate it.
        *   `description: 'Maximum number of pages to crawl (default: 100)'`: Explains its purpose and default value.
    *   **`onlyMainContent`**:
        *   `type: 'boolean'`: The `onlyMainContent` parameter must be a boolean (true/false).
        *   `required: false`: This parameter is optional.
        *   `visibility: 'user-only'`: Only a human user can provide this.
        *   `description: 'Extract only main content from pages'`: Explains that it controls whether only the main article content or the entire page content is extracted.
    *   **`apiKey`**:
        *   `type: 'string'`: The `apiKey` parameter must be a string.
        *   `required: true`: This parameter is mandatory.
        *   `visibility: 'user-only'`: Only a human user can provide this (it's a sensitive credential).
        *   `description: 'Firecrawl API Key'`: The API key required to authenticate with the Firecrawl service.

```typescript
  request: {
    url: 'https://api.firecrawl.dev/v1/crawl',
    method: 'POST',
    headers: (params) => ({
      'Content-Type': 'application/json',
      Authorization: `Bearer ${params.apiKey}`,
    }),
    body: (params) => ({
      url: params.url,
      limit: Number(params.limit) || 100,
      scrapeOptions: {
        formats: ['markdown'],
        onlyMainContent: params.onlyMainContent || false,
      },
    }),
  },
```
*   **`request: { ... }`**: This object defines how the initial API call to Firecrawl is constructed.
    *   **`url: 'https://api.firecrawl.dev/v1/crawl'`**: The target URL for the Firecrawl API endpoint that initiates a crawl job.
    *   **`method: 'POST'`**: Specifies that an HTTP POST request should be used to send the crawl initiation request.
    *   **`headers: (params) => ({ ... })`**: A function that generates the HTTP headers for the request. It takes the `params` (from the `params` definition above) as an argument.
        *   **`'Content-Type': 'application/json'`**: Indicates that the request body will be in JSON format.
        *   **`Authorization: \`Bearer ${params.apiKey}\``**: Provides the Firecrawl API key as a Bearer token for authentication.
    *   **`body: (params) => ({ ... })`**: A function that generates the HTTP request body in JSON format. It also takes the `params` as an argument.
        *   **`url: params.url`**: Passes the target URL provided by the user.
        *   **`limit: Number(params.limit) || 100`**: Passes the page limit. It converts `params.limit` to a number (as it might come as a string) and defaults to `100` if no limit is provided or it's not a valid number.
        *   **`scrapeOptions: { ... }`**: An object containing options for how Firecrawl should scrape the pages.
            *   **`formats: ['markdown']`**: Requests Firecrawl to return the scraped content in Markdown format.
            *   **`onlyMainContent: params.onlyMainContent || false`**: Passes the user's preference for extracting only main content, defaulting to `false` (extract all content) if not specified.

```typescript
  transformResponse: async (response: Response) => {
    const data = await response.json()

    return {
      success: true,
      output: {
        jobId: data.jobId || data.id,
        pages: [],
        total: 0,
        creditsUsed: 0,
      },
    }
  },
```
*   **`transformResponse: async (response: Response) => { ... }`**: This `async` function processes the *immediate* response received from Firecrawl after the initial `POST` request.
    *   **`const data = await response.json()`**: Parses the JSON body of the API response.
    *   **`return { ... }`**: Returns an object representing the initial state of the tool's output.
        *   **`success: true`**: Indicates that the initial request to *start* the crawl was successful (not that the crawl itself is complete).
        *   **`output: { ... }`**: Contains the initial output data.
            *   **`jobId: data.jobId || data.id`**: Extracts the job ID from the response. Firecrawl might return it as `jobId` or `id`, so it tries both. This `jobId` is crucial for polling.
            *   **`pages: []`, `total: 0`, `creditsUsed: 0`**: These are initialized as empty or zero values because the actual crawl results are not available yet. They will be populated later by the `postProcess` function.

```typescript
  postProcess: async (result, params) => {
    if (!result.success) {
      return result
    }

    const jobId = result.output.jobId
    logger.info(`Firecrawl crawl job ${jobId} created, polling for completion...`)

    let elapsedTime = 0

    while (elapsedTime < MAX_POLL_TIME_MS) {
      try {
        const statusResponse = await fetch(`/api/tools/firecrawl/crawl/${jobId}`, {
          method: 'GET',
          headers: {
            Authorization: `Bearer ${params.apiKey}`,
          },
        })

        if (!statusResponse.ok) {
          throw new Error(`Failed to get crawl status: ${statusResponse.statusText}`)
        }

        const crawlData = await statusResponse.json()
        logger.info(`Firecrawl crawl job ${jobId} status: ${crawlData.status}`)

        if (crawlData.status === 'completed') {
          result.output = {
            pages: crawlData.data || [],
            total: crawlData.total || 0,
            creditsUsed: crawlData.creditsUsed || 0,
          }
          return result
        }

        if (crawlData.status === 'failed') {
          return {
            ...result,
            success: false,
            error: `Crawl job failed: ${crawlData.error || 'Unknown error'}`,
          }
        }

        await new Promise((resolve) => setTimeout(resolve, POLL_INTERVAL_MS))
        elapsedTime += POLL_INTERVAL_MS
      } catch (error: any) {
        logger.error('Error polling for crawl job status:', {
          message: error.message || 'Unknown error',
          jobId,
        })

        return {
          ...result,
          success: false,
          error: `Error polling for crawl job status: ${error.message || 'Unknown error'}`,
        }
      }
    }

    logger.warn(
      `Crawl job ${jobId} did not complete within the maximum polling time (${MAX_POLL_TIME_MS / 1000}s)`
    )
    return {
      ...result,
      success: false,
      error: `Crawl job did not complete within the maximum polling time (${MAX_POLL_TIME_MS / 1000}s)`,
    }
  },
```
*   **`postProcess: async (result, params) => { ... }`**: This `async` function handles the asynchronous monitoring and final result retrieval of the Firecrawl job. It takes the `result` from `transformResponse` and the original `params`.
    *   **`if (!result.success) { return result }`**: If the initial `transformResponse` already indicated a failure (e.g., failed to get a job ID), this function immediately exits.
    *   **`const jobId = result.output.jobId`**: Extracts the `jobId` obtained from the initial response.
    *   **`logger.info(...)`**: Logs that a crawl job has been created and polling is starting.
    *   **`let elapsedTime = 0`**: Initializes a counter to track how long the polling has been active.
    *   **`while (elapsedTime < MAX_POLL_TIME_MS) { ... }`**: This is the core polling loop. It continues as long as the elapsed time is less than the maximum allowed polling time.
        *   **`try { ... } catch (error: any) { ... }`**: A `try-catch` block to handle any errors that occur during the polling process.
            *   **`const statusResponse = await fetch(\`/api/tools/firecrawl/crawl/${jobId}\`, { ... })`**: Makes a `GET` request to a *local API endpoint* (`/api/tools/firecrawl/crawl/${jobId}`). This endpoint is likely a proxy that communicates with Firecrawl's actual status API, abstracting it from the client.
            *   **`headers: { Authorization: \`Bearer ${params.apiKey}\` }`**: The API key is passed again for authentication to the local proxy.
            *   **`if (!statusResponse.ok) { throw new Error(...) }`**: Checks if the `GET` request for status was successful (HTTP status 200). If not, it throws an error.
            *   **`const crawlData = await statusResponse.json()`**: Parses the JSON response from the status check. This `crawlData` object contains the current status and potentially the results if completed.
            *   **`logger.info(...)`**: Logs the current status of the Firecrawl job.
            *   **`if (crawlData.status === 'completed') { ... }`**: If the job status is 'completed':
                *   **`result.output = { ... }`**: Updates the `output` property of the `result` object with the actual crawled `data` (pages), `total` pages, and `creditsUsed` from `crawlData`.
                *   **`return result`**: Returns the updated `result` object, indicating success and containing the final data.
            *   **`if (crawlData.status === 'failed') { ... }`**: If the job status is 'failed':
                *   **`return { ...result, success: false, error: ... }`**: Returns a modified `result` object, setting `success` to `false` and including an error message from `crawlData.error`.
            *   **`await new Promise((resolve) => setTimeout(resolve, POLL_INTERVAL_MS))`**: Pauses the execution for `POLL_INTERVAL_MS` (5 seconds) before the next poll.
            *   **`elapsedTime += POLL_INTERVAL_MS`**: Increments the `elapsedTime` counter.
        *   **`catch (error: any) { ... }`**: If an error occurs within the `try` block (e.g., network error, invalid response):
            *   **`logger.error(...)`**: Logs the error message.
            *   **`return { ...result, success: false, error: ... }`**: Returns a modified `result` object indicating failure with the error details.
    *   **`logger.warn(...)`**: If the `while` loop completes without the job reaching 'completed' or 'failed' status, it means the `MAX_POLL_TIME_MS` was exceeded. This line logs a warning.
    *   **`return { ...result, success: false, error: ... }`**: Returns a modified `result` object indicating failure due to a timeout.

```typescript
  outputs: {
    pages: {
      type: 'array',
      description: 'Array of crawled pages with their content and metadata',
      items: {
        type: 'object',
        properties: {
          markdown: { type: 'string', description: 'Page content in markdown format' },
          html: { type: 'string', description: 'Page HTML content' },
          metadata: {
            type: 'object',
            description: 'Page metadata',
            properties: {
              title: { type: 'string', description: 'Page title' },
              description: { type: 'string', description: 'Page description' },
              language: { type: 'string', description: 'Page language' },
              sourceURL: { type: 'string', description: 'Source URL of the page' },
              statusCode: { type: 'number', description: 'HTTP status code' },
            },
          },
        },
      },
    },
    total: { type: 'number', description: 'Total number of pages found during crawl' },
    creditsUsed: {
      type: 'number',
      description: 'Number of credits consumed by the crawl operation',
    },
  },
```
*   **`outputs: { ... }`**: This object defines the structure and types of the *final successful output* of the tool. This is essentially a schema for the `FirecrawlCrawlResponse` type imported earlier.
    *   **`pages`**:
        *   **`type: 'array'`**: The `pages` property will be an array.
        *   **`description: 'Array of crawled pages with their content and metadata'`**: Explains what the array contains.
        *   **`items: { ... }`**: Defines the structure of each item within the `pages` array. Each item is an `object`.
            *   **`properties: { ... }`**: Describes the properties of each page object:
                *   **`markdown: { type: 'string', description: 'Page content in markdown format' }`**: The page content, formatted as a string in Markdown.
                *   **`html: { type: 'string', description: 'Page HTML content' }`**: The raw HTML content of the page.
                *   **`metadata: { ... }`**: An object containing various metadata about the page.
                    *   **`type: 'object'`, `description: 'Page metadata'`**: Defines the `metadata` property.
                    *   **`properties: { ... }`**: Describes the properties within the `metadata` object:
                        *   `title: { type: 'string', description: 'Page title' }`: The HTML title of the page.
                        *   `description: { type: 'string', description: 'Page description' }`: The meta description of the page.
                        *   `language: { type: 'string', description: 'Page language' }`: The detected language of the page content.
                        *   `sourceURL: { type: 'string', description: 'Source URL of the page' }`: The original URL of the crawled page.
                        *   `statusCode: { type: 'number', description: 'HTTP status code' }`: The HTTP status code received when accessing the page.
    *   **`total: { type: 'number', description: 'Total number of pages found during crawl' }`**: The total count of pages that were found and processed during the crawl.
    *   **`creditsUsed: { type: 'number', description: 'Number of credits consumed by the crawl operation' }`**: The amount of Firecrawl credits consumed by this specific crawl job.