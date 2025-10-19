This TypeScript file defines a powerful "tool" named `Exa Research` that integrates with the Exa AI platform to perform comprehensive research. It's designed to be used within a larger system (like an AI agent or a custom workflow engine) that can execute such tools.

Think of this file as a detailed instruction manual for how to interact with the Exa AI research API, including what inputs it needs, how to make the initial request, how to handle the immediate response, and crucially, how to *wait* for the asynchronous research task to complete.

---

### **Purpose of This File**

The primary purpose of this file is to encapsulate the logic for performing an AI-driven research task using the Exa AI API. It exports a `ToolConfig` object, which is a standardized way for a system to understand and utilize this specific research functionality.

This `ToolConfig` includes:

1.  **Metadata**: Name, description, version, and a unique ID for the tool.
2.  **Input Parameters (`params`)**: Defines what information the tool needs to start a research task (e.g., a query, an API key).
3.  **Initial Request Logic (`request`)**: Specifies how to make the initial API call to Exa AI to *start* a research task, including the URL, HTTP method, headers, and the request body.
4.  **Initial Response Transformation (`transformResponse`)**: How to process the immediate response from starting the task, typically extracting a task ID.
5.  **Post-Processing Logic (`postProcess`)**: The most complex part, it describes how to *monitor* the research task's progress (since research takes time) by repeatedly checking its status until it completes or fails, then retrieving the final results.
6.  **Output Specification (`outputs`)**: Defines the structure of the data the tool will return once the research is finished.

In essence, it's a blueprint for an "Exa Research" capability that can be plugged into a larger system.

---

### **Detailed Explanation**

Let's go through the code line by line, simplifying complex logic as we encounter it.

#### **1. Imports and Setup**

```typescript
import { createLogger } from '@/lib/logs/console/logger'
import type { ExaResearchParams, ExaResearchResponse } from '@/tools/exa/types'
import type { ToolConfig } from '@/tools/types'
```

*   **`import { createLogger } from '@/lib/logs/console/logger'`**: This line imports a function `createLogger` from a local logging utility. This allows the tool to produce log messages (e.g., info, warn, error) during its operation, which is helpful for debugging and monitoring.
*   **`import type { ExaResearchParams, ExaResearchResponse } from '@/tools/exa/types'`**: These are TypeScript `type` imports. They don't import any runnable code but provide type definitions for:
    *   `ExaResearchParams`: The expected input parameters when this tool is invoked.
    *   `ExaResearchResponse`: The expected output structure when the tool successfully completes.
*   **`import type { ToolConfig } from '@/tools/types'`**: This imports the base `ToolConfig` type, which is a generic interface defining the structure for all tools in the system. This ensures that `researchTool` adheres to a consistent pattern.

```typescript
const logger = createLogger('ExaResearchTool')
```

*   **`const logger = createLogger('ExaResearchTool')`**: Here, we create an instance of our logger specifically for this tool, naming it `'ExaResearchTool'`. Any logs from this tool will be tagged with this name, making it easier to identify where messages are coming from in the console.

```typescript
const POLL_INTERVAL_MS = 5000 // 5 seconds between polls
const MAX_POLL_TIME_MS = 300000 // 5 minutes maximum polling time
```

*   **`const POLL_INTERVAL_MS = 5000`**: This constant defines how long the system should wait (in milliseconds) between checking the status of the ongoing research task. In this case, it's 5000 ms, or 5 seconds.
*   **`const MAX_POLL_TIME_MS = 300000`**: This constant sets the maximum total time (in milliseconds) the system will continuously poll for the research task to complete. 300,000 ms is 5 minutes. If the task isn't finished within this duration, the tool will time out.

#### **2. The `researchTool` Configuration Object**

```typescript
export const researchTool: ToolConfig<ExaResearchParams, ExaResearchResponse> = {
  // ... configuration details ...
}
```

*   **`export const researchTool: ToolConfig<ExaResearchParams, ExaResearchResponse> = { ... }`**: This is the main declaration. We are exporting a constant named `researchTool`. Its type is explicitly `ToolConfig<ExaResearchParams, ExaResearchResponse>`, indicating it's a tool configuration that takes `ExaResearchParams` as input and produces `ExaResearchResponse` as output.

Let's break down the properties within this object:

##### **Metadata**

```typescript
  id: 'exa_research',
  name: 'Exa Research',
  description:
    'Perform comprehensive research using AI to generate detailed reports with citations',
  version: '1.0.0',
```

*   **`id: 'exa_research'`**: A unique identifier for this tool within the system.
*   **`name: 'Exa Research'`**: A human-readable name for the tool.
*   **`description: 'Perform comprehensive research using AI to generate detailed reports with citations'`**: A brief explanation of what the tool does, useful for users or other AI models interacting with it.
*   **`version: '1.0.0'`**: The version of this tool configuration.

##### **Input Parameters (`params`)**

This section defines the arguments or inputs that the `Exa Research` tool expects when it's called.

```typescript
  params: {
    query: {
      type: 'string',
      required: true,
      visibility: 'user-or-llm',
      description: 'Research query or topic',
    },
    includeText: {
      type: 'boolean',
      required: false,
      visibility: 'user-only',
      description: 'Include full text content in results',
    },
    apiKey: {
      type: 'string',
      required: true,
      visibility: 'user-only',
      description: 'Exa AI API Key',
    },
  },
```

*   **`params`**: This object lists all the parameters the tool accepts. Each parameter has its own configuration:
    *   **`query`**:
        *   **`type: 'string'`**: The `query` must be a text string.
        *   **`required: true`**: This parameter is mandatory for the tool to function.
        *   **`visibility: 'user-or-llm'`**: This indicates that the `query` can be provided either by a human user or by a Large Language Model (LLM) if the tool is being used by an AI agent.
        *   **`description: 'Research query or topic'`**: Explains the purpose of this parameter.
    *   **`includeText`**:
        *   **`type: 'boolean'`**: This parameter expects a true/false value.
        *   **`required: false`**: This parameter is optional.
        *   **`visibility: 'user-only'`**: This parameter is typically set by a human user and not expected to be inferred or generated by an LLM.
        *   **`description: 'Include full text content in results'`**: Explains that if true, the research results will contain full text.
    *   **`apiKey`**:
        *   **`type: 'string'`**: The `apiKey` must be a text string.
        *   **`required: true`**: This parameter is mandatory.
        *   **`visibility: 'user-only'`**: The API key should be provided by a user, not an LLM. This is a security-conscious decision.
        *   **`description: 'Exa AI API Key'`**: Explains what this parameter is for.

##### **Initial Request Logic (`request`)**

This section defines how to make the *first* API call to Exa AI to initiate a research task.

```typescript
  request: {
    url: 'https://api.exa.ai/research/v0/tasks',
    method: 'POST',
    headers: (params) => ({
      'Content-Type': 'application/json',
      'x-api-key': params.apiKey,
    }),
    body: (params) => {
      const body: any = {
        instructions: params.query,
        model: 'exa-research',
        output: {
          schema: {
            type: 'object',
            properties: {
              results: {
                type: 'array',
                items: {
                  type: 'object',
                  properties: {
                    title: { type: 'string' },
                    url: { type: 'string' },
                    summary: { type: 'string' },
                    text: { type: 'string' },
                    publishedDate: { type: 'string' },
                    author: { type: 'string' },
                    score: { type: 'number' },
                  },
                },
              },
            },
            required: ['results'],
          },
        },
      }

      return body
    },
  },
```

*   **`request`**: This object configures the HTTP request.
    *   **`url: 'https://api.exa.ai/research/v0/tasks'`**: The endpoint to which the initial request will be sent. This is where Exa AI expects new research tasks to be created.
    *   **`method: 'POST'`**: The HTTP method used. `POST` is appropriate for creating new resources (like a research task).
    *   **`headers: (params) => ({ ... })`**: This is a function that takes the tool's input `params` and returns an object of HTTP headers.
        *   **`'Content-Type': 'application/json'`**: Tells the server that the request body is in JSON format.
        *   **`'x-api-key': params.apiKey`**: Sends the user-provided Exa AI API key in the `x-api-key` header, which is a common way to authenticate API requests.
    *   **`body: (params) => { ... }`**: This is also a function that takes the tool's input `params` and constructs the JSON body for the `POST` request.
        *   **`const body: any = { ... }`**: Declares a variable `body` to hold the request payload. `any` is used here for flexibility, but in a strictly typed system, it would be `ExaResearchRequestBody`.
        *   **`instructions: params.query`**: Passes the user's research query directly to Exa AI as "instructions".
        *   **`model: 'exa-research'`**: Specifies which Exa AI model to use for the research.
        *   **`output: { schema: { ... } }`**: This is crucial. It tells Exa AI the *desired JSON schema* for the research results. This ensures that Exa AI formats its output in a predictable and usable way for our tool.
            *   It defines that the output should be an `object` with a property called `results`.
            *   `results` should be an `array` (`items` specifies the structure of each item in the array).
            *   Each item in the `results` array should be an `object` with properties like `title`, `url`, `summary`, `text`, `publishedDate`, `author`, and `score`, all with specified types (mostly `string`, `number`).
            *   **`required: ['results']`**: Ensures that the `results` array must be present in Exa AI's output.

##### **Initial Response Transformation (`transformResponse`)**

After the initial `POST` request is sent, the Exa AI API will immediately respond with a task ID. This function processes that immediate response.

```typescript
  transformResponse: async (response: Response) => {
    const data = await response.json()

    return {
      success: true,
      output: {
        taskId: data.id,
        research: [], // Initialize research as empty, it will be filled in postProcess
      },
    }
  },
```

*   **`transformResponse: async (response: Response) => { ... }`**: This asynchronous function takes the raw HTTP `response` from the initial `POST` request.
    *   **`const data = await response.json()`**: It parses the JSON body of the response, which is expected to contain a `taskId`.
    *   **`return { success: true, output: { taskId: data.id, research: [] } }`**: It returns a standardized object:
        *   **`success: true`**: Indicates that the initial request was successful in creating a task.
        *   **`output: { taskId: data.id, research: [] }`**: Contains the `taskId` (extracted from `data.id`) which is essential for polling later, and an empty `research` array. The actual research results will be populated in the `postProcess` step.

##### **Post-Processing Logic (`postProcess`) - The Polling Mechanism**

This is the most complex part of the tool. Since Exa AI research tasks are asynchronous (they take time to complete), this function continuously checks the status of the task until it's done, failed, or times out.

```typescript
  postProcess: async (result, params) => {
    if (!result.success) {
      return result // If initial request failed, just return the failure
    }

    const taskId = result.output.taskId
    logger.info(`Exa research task ${taskId} created, polling for completion...`)

    let elapsedTime = 0

    while (elapsedTime < MAX_POLL_TIME_MS) { // Loop until max time reached
      try {
        // 1. Fetch task status
        const statusResponse = await fetch(`https://api.exa.ai/research/v0/tasks/${taskId}`, {
          method: 'GET',
          headers: {
            'x-api-key': params.apiKey,
          },
        })

        if (!statusResponse.ok) {
          throw new Error(`Failed to get task status: ${statusResponse.statusText}`)
        }

        const taskData = await statusResponse.json()
        logger.info(`Exa research task ${taskId} status: ${taskData.status}`)

        // 2. Check task status
        if (taskData.status === 'completed') {
          result.output = {
            research: taskData.data?.results || [
              {
                title: 'Research Complete',
                url: '',
                summary: taskData.data || 'Research completed successfully',
                text: undefined,
                publishedDate: undefined,
                author: undefined,
                score: 1.0,
              },
            ],
          }
          return result // Task completed, return results
        }

        if (taskData.status === 'failed') {
          return { // Task failed, return an error
            ...result,
            success: false,
            error: `Research task failed: ${taskData.error || 'Unknown error'}`,
          }
        }

        // 3. Wait before next poll
        await new Promise((resolve) => setTimeout(resolve, POLL_INTERVAL_MS))
        elapsedTime += POLL_INTERVAL_MS

      } catch (error: any) { // Handle errors during polling
        logger.error('Error polling for research task status:', {
          message: error.message || 'Unknown error',
          taskId,
        })

        return {
          ...result,
          success: false,
          error: `Error polling for research task status: ${error.message || 'Unknown error'}`,
        }
      }
    }

    // 4. Timeout if loop finishes without completion/failure
    logger.warn(
      `Research task ${taskId} did not complete within the maximum polling time (${MAX_POLL_TIME_MS / 1000}s)`
    )
    return {
      ...result,
      success: false,
      error: `Research task did not complete within the maximum polling time (${MAX_POLL_TIME_MS / 1000}s)`,
    }
  },
```

*   **`postProcess: async (result, params) => { ... }`**: This asynchronous function takes the `result` from `transformResponse` (which contains the `taskId`) and the original `params` (to access the `apiKey` for subsequent calls).
*   **`if (!result.success) { return result }`**: If the initial task creation failed, there's nothing to poll for, so it immediately returns the failure `result`.
*   **`const taskId = result.output.taskId`**: Extracts the task ID that was obtained in the previous step.
*   **`logger.info(...)`**: Logs that polling has started.
*   **`let elapsedTime = 0`**: Initializes a counter to track how much time has passed during polling.
*   **`while (elapsedTime < MAX_POLL_TIME_MS)`**: This loop is the heart of the polling mechanism. It continues to run as long as the `elapsedTime` is less than the `MAX_POLL_TIME_MS` (5 minutes).

    *   **`try { ... } catch (error: any) { ... }`**: A `try...catch` block wraps the polling logic to gracefully handle any network errors or other exceptions that might occur during the `fetch` calls.
        *   **`const statusResponse = await fetch(...)`**: Makes a `GET` request to `https://api.exa.ai/research/v0/tasks/${taskId}` to fetch the current status of the task.
        *   **`headers: { 'x-api-key': params.apiKey }`**: Again, the API key is sent for authentication.
        *   **`if (!statusResponse.ok) { throw new Error(...) }`**: Checks if the HTTP response status is *not* OK (e.g., 404, 500). If it's not, it throws an error.
        *   **`const taskData = await statusResponse.json()`**: Parses the JSON response from the status check, which contains information about the task's current status.
        *   **`logger.info(...)`**: Logs the current status of the Exa research task (e.g., "processing", "completed").
        *   **`if (taskData.status === 'completed') { ... }`**:
            *   **Success Condition**: If the `taskData.status` is `'completed'`, the research is done.
            *   **`result.output = { research: taskData.data?.results || [...] }`**: Updates the `output` property of our `result` object. It tries to assign `taskData.data.results` (which contains the actual research findings). If `taskData.data.results` is empty or undefined, it provides a default single research item indicating completion.
            *   **`return result`**: The function successfully completes and returns the `result` with the gathered research. This exits the `while` loop and the `postProcess` function.
        *   **`if (taskData.status === 'failed') { ... }`**:
            *   **Failure Condition**: If the `taskData.status` is `'failed'`, something went wrong with the Exa AI task.
            *   **`return { ...result, success: false, error: ... }`**: The function returns a modified `result` object, setting `success` to `false` and providing an error message based on `taskData.error` or a generic "Unknown error". This exits the `while` loop and the `postProcess` function.
        *   **`await new Promise((resolve) => setTimeout(resolve, POLL_INTERVAL_MS))`**: If the task is neither `completed` nor `failed`, the code pauses execution for `POLL_INTERVAL_MS` (5 seconds) using `setTimeout` wrapped in a Promise. This prevents hammering the API with requests and allows the Exa AI task to make progress.
        *   **`elapsedTime += POLL_INTERVAL_MS`**: Increments the `elapsedTime` counter by the poll interval.
    *   **`catch (error: any) { ... }`**: If any error occurs *during* a `fetch` call or processing its response within the `try` block:
        *   **`logger.error(...)`**: Logs the error message.
        *   **`return { ...result, success: false, error: ... }`**: Returns a failure `result` with an appropriate error message. This exits the `while` loop and the `postProcess` function.
*   **After the `while` loop (Timeout)**:
    *   If the `while` loop finishes without the task being `completed` or `failed`, it means `elapsedTime` has reached `MAX_POLL_TIME_MS`.
    *   **`logger.warn(...)`**: A warning is logged indicating that the task timed out.
    *   **`return { ...result, success: false, error: ... }`**: A failure `result` is returned, explaining that the task did not complete within the maximum allowed time.

##### **Output Specification (`outputs`)**

This section formally describes the structure of the data that this tool will produce upon successful completion. This is useful for systems that need to know what kind of data to expect from the tool.

```typescript
  outputs: {
    research: {
      type: 'array',
      description: 'Comprehensive research findings with citations and summaries',
      items: {
        type: 'object',
        properties: {
          title: { type: 'string' },
          url: { type: 'string' },
          summary: { type: 'string' },
          text: { type: 'string' },
          publishedDate: { type: 'string' },
          author: { type: 'string' },
          score: { type: 'number' },
        },
      },
    },
  },
```

*   **`outputs`**: This object describes the final output structure.
    *   **`research`**: The main output property.
        *   **`type: 'array'`**: Indicates that `research` will be an array.
        *   **`description: 'Comprehensive research findings with citations and summaries'`**: Explains what the array contains.
        *   **`items: { ... }`**: Describes the structure of each individual item *within* the `research` array.
            *   **`type: 'object'`**: Each item is an object.
            *   **`properties: { ... }`**: Defines the expected properties of each research result object: `title`, `url`, `summary`, `text`, `publishedDate`, `author`, and `score`, all with their respective `type` declarations. This mirrors the `output.schema` defined in the `request.body` earlier, ensuring consistency.

---

### **Simplified Complex Logic: The Polling Mechanism**

The most intricate part is the `postProcess` function, which implements a "polling" pattern. Here's a simplified explanation:

1.  **Start Task, Get ID**: When you ask Exa AI to do research, it doesn't give you the results right away. Instead, it says, "Okay, I've started your task; here's a `taskId` (like an order number)."
2.  **Keep Checking**: Since research takes time, your system (via `postProcess`) can't just wait. It needs to periodically ask Exa AI, "Hey, what's the status of order number `[taskId]`?"
3.  **The Loop**: This "asking" happens in a `while` loop:
    *   It asks Exa AI for the task's status.
    *   It checks the response:
        *   **"Completed!"**: If Exa AI says the task is `completed`, great! Grab the results, and the tool is done.
        *   **"Failed!"**: If Exa AI says it `failed`, then the tool reports an error and stops.
        *   **"Still working..."**: If Exa AI gives any other status, the tool waits for a few seconds (the `POLL_INTERVAL_MS`), then goes back to the beginning of the loop to ask again.
4.  **Time Limit**: This checking doesn't go on forever. There's a `MAX_POLL_TIME_MS` (5 minutes). If the task isn't completed or failed within this time, the tool assumes something went wrong (or it's taking too long) and reports a timeout error.
5.  **Safety Net**: If anything unexpected happens during any of these status checks (like a network issue), the `try...catch` block catches it, logs the error, and reports a failure.

This polling strategy is common for APIs where operations are asynchronous and can take an unpredictable amount of time.

---

This detailed configuration makes the `Exa Research` tool highly reusable and integrates seamlessly into systems designed to manage and execute such defined tools.