This TypeScript file defines a "tool" configuration for interacting with the **BrowserUse API**. In essence, it allows an external system (like an AI agent or an automation platform) to programmatically trigger and monitor browser automation tasks executed by the BrowserUse service.

This tool is designed to encapsulate all the necessary details for:
1.  **Defining the input parameters** a user or an AI might provide for a browser task.
2.  **Constructing the HTTP request** to send a new task to the BrowserUse API.
3.  **Processing the initial response** from the API, which typically includes a task ID.
4.  **Monitoring the task's progress** by repeatedly polling the BrowserUse API until the task completes or times out.
5.  **Transforming the final task results** into a standardized output format.

---

### Detailed Explanation

Let's break down the code section by section.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
import type { BrowserUseRunTaskParams, BrowserUseRunTaskResponse } from '@/tools/browser_use/types'
import type { ToolConfig } from '@/tools/types'
```

*   **`import { createLogger } from '@/lib/logs/console/logger'`**: This line imports the `createLogger` function from an internal logging utility. This function will be used to create a logger instance for this specific tool, allowing it to output informative, warning, or error messages during its operation.
*   **`import type { BrowserUseRunTaskParams, BrowserUseRunTaskResponse } from '@/tools/browser_use/types'`**: These are TypeScript `type` imports. They bring in specific type definitions related to the BrowserUse tool:
    *   `BrowserUseRunTaskParams`: Defines the structure of the input parameters that can be provided when running a BrowserUse task.
    *   `BrowserUseRunTaskResponse`: Defines the structure of the expected output response after a BrowserUse task has completed.
*   **`import type { ToolConfig } from '@/tools/types'`**: This imports the generic `ToolConfig` type. This type provides a standard structure for defining any tool within the system, ensuring consistency across different integrations. It's a generic type, parameterized by the input parameters and the output response types for this specific tool.

```typescript
const logger = createLogger('BrowserUseTool')

const POLL_INTERVAL_MS = 5000 // 5 seconds between polls
const MAX_POLL_TIME_MS = 180000 // 3 minutes maximum polling time
```

*   **`const logger = createLogger('BrowserUseTool')`**: An instance of a logger is created specifically for this tool, identified by the name `'BrowserUseTool'`. This allows for clear log messages that indicate their origin.
*   **`const POLL_INTERVAL_MS = 5000`**: Defines a constant for the polling interval in milliseconds. This means the system will wait 5 seconds between each check for the task's status.
*   **`const MAX_POLL_TIME_MS = 180000`**: Defines a constant for the maximum time (in milliseconds) the system will wait for a BrowserUse task to complete. If the task doesn't finish within 3 minutes (180,000 ms), the polling will stop, and an error indicating a timeout will be returned.

---

### `runTaskTool` Configuration Object

This is the core of the file, defining the `ToolConfig` object that encapsulates all the logic for the "Browser Use" tool.

```typescript
export const runTaskTool: ToolConfig<BrowserUseRunTaskParams, BrowserUseRunTaskResponse> = {
```

*   **`export const runTaskTool: ToolConfig<BrowserUseRunTaskParams, BrowserUseRunTaskResponse> = { ... }`**: This declares and exports a constant named `runTaskTool`. Its type is `ToolConfig`, specialized with `BrowserUseRunTaskParams` for its input parameters and `BrowserUseRunTaskResponse` for its output. This `export` makes the tool definition available for other parts of the application.

#### Basic Tool Metadata

```typescript
  id: 'browser_use_run_task',
  name: 'Browser Use',
  description: 'Runs a browser automation task using BrowserUse',
  version: '1.0.0',
```

*   **`id: 'browser_use_run_task'`**: A unique identifier for this specific tool. Used internally to reference it.
*   **`name: 'Browser Use'`**: A human-readable name for the tool, often displayed in user interfaces.
*   **`description: 'Runs a browser automation task using BrowserUse'`**: A brief explanation of what the tool does.
*   **`version: '1.0.0'`**: The version number of this tool definition.

#### `params` - Defining Input Parameters

This section describes all the inputs that a user or an LLM (Large Language Model) can provide when invoking this tool.

```typescript
  params: {
    task: {
      type: 'string',
      required: true,
      visibility: 'user-or-llm',
      description: 'What should the browser agent do',
    },
    variables: {
      type: 'json',
      required: false,
      visibility: 'user-only',
      description: 'Optional variables to use as secrets (format: {key: value})',
    },
    save_browser_data: {
      type: 'boolean',
      required: false,
      visibility: 'user-only',
      description: 'Whether to save browser data',
    },
    model: {
      type: 'string',
      required: false,
      visibility: 'user-only',
      description: 'LLM model to use (default: gpt-4o)',
    },
    apiKey: {
      type: 'string',
      required: true,
      visibility: 'user-only',
      description: 'API key for BrowserUse API',
    },
  },
```

Each property within `params` defines an input field:
*   **`task`**:
    *   `type: 'string'`: Expects a text description of the browser task.
    *   `required: true`: This parameter is mandatory.
    *   `visibility: 'user-or-llm'`: Both a human user and an AI model can provide this parameter.
    *   `description: 'What should the browser agent do'`: Explains its purpose.
*   **`variables`**:
    *   `type: 'json'`: Expects data in JSON format, which means it can be an object or an array.
    *   `required: false`: This parameter is optional.
    *   `visibility: 'user-only'`: Only a human user should provide this, not directly an LLM (likely because it contains sensitive information like secrets).
    *   `description: 'Optional variables to use as secrets (format: {key: value})'`: Explains its use for sensitive data.
*   **`save_browser_data`**:
    *   `type: 'boolean'`: Expects a true/false value.
    *   `required: false`: Optional.
    *   `visibility: 'user-only'`: User-provided only.
    *   `description: 'Whether to save browser data'`: Controls whether the browser session data should be saved.
*   **`model`**:
    *   `type: 'string'`: Expects a text value.
    *   `required: false`: Optional.
    *   `visibility: 'user-only'`: User-provided only.
    *   `description: 'LLM model to use (default: gpt-4o)'`: Allows specifying a different LLM model for the browser agent, overriding the default.
*   **`apiKey`**:
    *   `type: 'string'`: Expects a text value.
    *   `required: true`: This parameter is mandatory.
    *   `visibility: 'user-only'`: User-provided only (as it's a secret API key).
    *   `description: 'API key for BrowserUse API'`: The authentication key needed to access the BrowserUse service.

#### `request` - Defining the API Call

This section describes how to make the initial HTTP request to the BrowserUse API to start a task.

```typescript
  request: {
    url: 'https://api.browser-use.com/api/v1/run-task',
    method: 'POST',
    headers: (params) => ({
      'Content-Type': 'application/json',
      Authorization: `Bearer ${params.apiKey}`,
    }),
    body: (params) => {
      const requestBody: Record<string, any> = {
        task: params.task,
      }

      if (params.variables) {
        let secrets: Record<string, string> = {}

        if (Array.isArray(params.variables)) {
          logger.info('Converting variables array to dictionary format')
          params.variables.forEach((row) => {
            if (row.cells?.Key && row.cells.Value !== undefined) {
              secrets[row.cells.Key] = row.cells.Value
              logger.info(`Added secret for key: ${row.cells.Key}`)
            } else if (row.Key && row.Value !== undefined) {
              secrets[row.Key] = row.Value
              logger.info(`Added secret for key: ${row.Key}`)
            }
          })
        } else if (typeof params.variables === 'object' && params.variables !== null) {
          logger.info('Using variables object directly')
          secrets = params.variables
        }

        if (Object.keys(secrets).length > 0) {
          logger.info(`Found ${Object.keys(secrets).length} secrets to include`)
          requestBody.secrets = secrets
        } else {
          logger.warn('No usable secrets found in variables')
        }
      }

      if (params.model) {
        requestBody.llm_model = params.model
      }

      if (params.save_browser_data) {
        requestBody.save_browser_data = params.save_browser_data
      }

      requestBody.use_adblock = true
      requestBody.highlight_elements = true

      return requestBody
    },
  },
```

*   **`url: 'https://api.browser-use.com/api/v1/run-task'`**: The specific API endpoint for initiating a new browser task.
*   **`method: 'POST'`**: The HTTP method used to send the request (creating a new resource).
*   **`headers: (params) => ({ ... })`**: A function that generates the HTTP headers for the request.
    *   `'Content-Type': 'application/json'`: Specifies that the request body will be in JSON format.
    *   `Authorization: `Bearer ${params.apiKey}``: Sets the `Authorization` header, which is essential for authenticating with the BrowserUse API. It uses the `apiKey` provided in the tool's parameters.
*   **`body: (params) => { ... }`**: A function that dynamically constructs the JSON body of the HTTP request based on the provided `params`.
    *   **`const requestBody: Record<string, any> = { task: params.task, }`**: Initializes the request body with the mandatory `task` description.
    *   **`if (params.variables) { ... }`**: This block handles the `variables` parameter, which is designed to accept "secrets." This is the most complex part of the `body` construction.
        *   **`let secrets: Record<string, string> = {}`**: An empty object is initialized to store the parsed secrets.
        *   **`if (Array.isArray(params.variables)) { ... }`**: This condition checks if `params.variables` is an array. This likely supports a UI input where secrets might be entered as rows in a table.
            *   `logger.info('Converting variables array to dictionary format')`: Logs an informational message.
            *   **`params.variables.forEach((row) => { ... })`**: It iterates over each `row` in the `variables` array.
                *   **`if (row.cells?.Key && row.cells.Value !== undefined) { ... }`**: Checks if the `row` has a `cells` object with `Key` and `Value` properties (common in UI table components). If found, it adds `Key` and `Value` to the `secrets` object.
                *   **`else if (row.Key && row.Value !== undefined) { ... }`**: If not in `cells` format, it checks if `row` directly has `Key` and `Value` properties.
        *   **`else if (typeof params.variables === 'object' && params.variables !== null) { ... }`**: If `params.variables` is not an array but a direct object (e.g., `{ "mySecret": "value" }`), it's used directly as the `secrets` object.
        *   **`if (Object.keys(secrets).length > 0) { ... }`**: If any secrets were successfully parsed, they are added to the `requestBody` under the key `secrets`.
        *   **`else { logger.warn('No usable secrets found in variables') }`**: If `params.variables` was provided but no valid secrets could be extracted, a warning is logged.
    *   **`if (params.model) { requestBody.llm_model = params.model }`**: If an `llm_model` is specified in `params`, it's added to the request body.
    *   **`if (params.save_browser_data) { requestBody.save_browser_data = params.save_browser_data }`**: If `save_browser_data` is `true` in `params`, it's added to the request body.
    *   **`requestBody.use_adblock = true`**: Hardcodes the `use_adblock` option to `true` (enabling ad blocker for the browser task).
    *   **`requestBody.highlight_elements = true`**: Hardcodes `highlight_elements` to `true` (likely for visual debugging or user feedback during automation).
    *   **`return requestBody`**: Returns the complete and constructed request body.

#### `transformResponse` - Initial Response Processing

This function handles the *immediate* response from the `run-task` API call, which typically only confirms the task's initiation and provides its ID.

```typescript
  transformResponse: async (response: Response) => {
    const data = (await response.json()) as { id: string }
    return {
      success: true,
      output: {
        id: data.id,
        success: true,
        output: null,
        steps: [],
      },
    }
  },
```

*   **`transformResponse: async (response: Response) => { ... }`**: This asynchronous function takes the raw HTTP `response` object received after the `POST` request.
*   **`const data = (await response.json()) as { id: string }`**: It parses the response body as JSON. It's assumed the initial response will contain an `id` property, which is the unique identifier for the newly created browser task.
*   **`return { success: true, output: { ... } }`**: It returns a standardized output object.
    *   `success: true`: Indicates that the task was successfully *initiated* (not necessarily completed).
    *   `output`: Contains the initial output structure:
        *   `id: data.id`: The task ID obtained from the API response.
        *   `success: true`: Reflects the initiation status.
        *   `output: null`: The actual result of the browser task is not yet available, so it's `null`.
        *   `steps: []`: Similarly, the execution steps are not yet known.

#### `postProcess` - Monitoring and Finalizing the Task

This is the most intricate part, responsible for actively monitoring the BrowserUse task until it finishes, fails, or times out.

```typescript
  postProcess: async (result, params) => {
    if (!result.success) {
      return result
    }

    const taskId = result.output.id
    let liveUrlLogged = false

    try {
      const initialTaskResponse = await fetch(`https://api.browser-use.com/api/v1/task/${taskId}`, {
        method: 'GET',
        headers: {
          Authorization: `Bearer ${params.apiKey}`,
        },
      })

      if (initialTaskResponse.ok) {
        const initialTaskData = await initialTaskResponse.json()
        if (initialTaskData.live_url) {
          logger.info(
            `BrowserUse task ${taskId} launched with live URL: ${initialTaskData.live_url}`
          )
          liveUrlLogged = true
        }
      }
    } catch (error) {
      logger.warn(`Failed to get initial task details for ${taskId}:`, error)
    }

    let elapsedTime = 0

    while (elapsedTime < MAX_POLL_TIME_MS) {
      try {
        const statusResponse = await fetch(
          `https://api.browser-use.com/api/v1/task/${taskId}/status`,
          {
            method: 'GET',
            headers: {
              Authorization: `Bearer ${params.apiKey}`,
            },
          }
        )

        if (!statusResponse.ok) {
          throw new Error(`Failed to get task status: ${statusResponse.statusText}`)
        }

        const status = await statusResponse.json()

        logger.info(`BrowserUse task ${taskId} status: ${status}`)

        if (['finished', 'failed', 'stopped'].includes(status)) {
          const taskResponse = await fetch(`https://api.browser-use.com/api/v1/task/${taskId}`, {
            method: 'GET',
            headers: {
              Authorization: `Bearer ${params.apiKey}`,
            },
          })

          if (taskResponse.ok) {
            const taskData = await taskResponse.json()
            result.output = {
              id: taskId,
              success: status === 'finished',
              output: taskData.output,
              steps: taskData.steps || [],
            }
          }

          return result
        }

        if (!liveUrlLogged && status === 'running') {
          const taskResponse = await fetch(`https://api.browser-use.com/api/v1/task/${taskId}`, {
            method: 'GET',
            headers: {
              Authorization: `Bearer ${params.apiKey}`,
            },
          })

          if (taskResponse.ok) {
            const taskData = await taskResponse.json()
            if (taskData.live_url) {
              logger.info(`BrowserUse task ${taskId} running with live URL: ${taskData.live_url}`)
              liveUrlLogged = true
            }
          }
        }

        await new Promise((resolve) => setTimeout(resolve, POLL_INTERVAL_MS))
        elapsedTime += POLL_INTERVAL_MS
      } catch (error: any) {
        logger.error('Error polling for task status:', {
          message: error.message || 'Unknown error',
          taskId,
        })

        return {
          ...result,
          error: `Error polling for task status: ${error.message || 'Unknown error'}`,
        }
      }
    }

    logger.warn(
      `Task ${taskId} did not complete within the maximum polling time (${MAX_POLL_TIME_MS / 1000}s)`
    )
    return {
      ...result,
      error: `Task did not complete within the maximum polling time (${MAX_POLL_TIME_MS / 1000}s)`,
    }
  },
```

*   **`postProcess: async (result, params) => { ... }`**: This asynchronous function takes the `result` from `transformResponse` (which contains the `taskId`) and the original `params`.
*   **`if (!result.success) { return result }`**: If the initial task launch failed, there's nothing to poll, so it returns the existing result.
*   **`const taskId = result.output.id`**: Extracts the task ID that was returned from the initial `transformResponse`.
*   **`let liveUrlLogged = false`**: A flag to ensure the "live URL" (a link to watch the task in real-time) is logged only once.

    *   **Initial `live_url` Fetch (Optional):**
        *   **`try { ... } catch (error) { ... }`**: Attempts to fetch full task details immediately after initiation. This is a best-effort attempt to get the `live_url` as early as possible.
        *   **`await fetch(...`api/v1/task/${taskId}`...)`**: Makes a GET request to retrieve the full task details.
        *   **`if (initialTaskResponse.ok) { ... }`**: If the request is successful.
        *   **`const initialTaskData = await initialTaskResponse.json()`**: Parses the task data.
        *   **`if (initialTaskData.live_url) { ... }`**: If a `live_url` is present, it's logged, and `liveUrlLogged` is set to `true`.
        *   **`logger.warn(...)`**: Catches and logs any errors during this initial fetch, but doesn't stop the process.

*   **`let elapsedTime = 0`**: Initializes a counter for how long the polling has been active.
*   **`while (elapsedTime < MAX_POLL_TIME_MS) { ... }`**: This is the main polling loop. It continues as long as the `elapsedTime` is less than the `MAX_POLL_TIME_MS`.
    *   **`try { ... } catch (error: any) { ... }`**: A `try-catch` block to gracefully handle errors during each polling attempt.
        *   **`const statusResponse = await fetch(`.../api/v1/task/${taskId}/status`...)`**: Makes a GET request to a specific endpoint to get *only* the current status of the task.
        *   **`if (!statusResponse.ok) { throw new Error(...) }`**: If the status request fails, it throws an error, which is caught by the outer `catch` block.
        *   **`const status = await statusResponse.json()`**: Parses the response to get the task's status (e.g., 'running', 'finished', 'failed').
        *   **`logger.info(`BrowserUse task ${taskId} status: ${status}`) `**: Logs the current status of the task.
        *   **`if (['finished', 'failed', 'stopped'].includes(status)) { ... }`**: Checks if the task has reached a terminal state (finished, failed, or stopped).
            *   **`const taskResponse = await fetch(`.../api/v1/task/${taskId}`...)`**: If terminal, it makes *another* GET request to fetch the *full* task details, including its output and steps.
            *   **`if (taskResponse.ok) { ... }`**: If this fetch is successful.
            *   **`const taskData = await taskResponse.json()`**: Parses the full task data.
            *   **`result.output = { ... }`**: Updates the `result.output` with the final data:
                *   `id: taskId`: The task ID.
                *   `success: status === 'finished'`: Sets `success` based on whether the task `finished` successfully.
                *   `output: taskData.output`: The actual output produced by the browser automation.
                *   `steps: taskData.steps || []`: The steps taken during execution, defaulting to an empty array if not present.
            *   **`return result`**: The function returns the updated `result`, ending the polling.
        *   **`if (!liveUrlLogged && status === 'running') { ... }`**: This block is similar to the initial `live_url` fetch but occurs *during* polling. If the `live_url` hasn't been logged yet and the task is now reported as 'running', it attempts to fetch and log the `live_url`. This ensures the live URL is reported as soon as the task truly starts executing.
        *   **`await new Promise((resolve) => setTimeout(resolve, POLL_INTERVAL_MS))`**: This line pauses the execution for the `POLL_INTERVAL_MS` (5 seconds) before the next iteration of the loop.
        *   **`elapsedTime += POLL_INTERVAL_MS`**: Increments the `elapsedTime` counter.
    *   **`catch (error: any) { ... }`**: If any error occurs within the polling loop (e.g., network issue, API error).
        *   **`logger.error(...)`**: Logs the error message.
        *   **`return { ...result, error: ... }`**: Returns the current `result` with an added `error` message, terminating the polling due to an unexpected issue.

*   **Timeout Handling:**
    *   **`logger.warn(...)`**: If the `while` loop completes (meaning `elapsedTime` reached `MAX_POLL_TIME_MS`) without the task reaching a terminal state, a warning is logged indicating a timeout.
    *   **`return { ...result, error: ... }`**: Returns the current `result` with an `error` message indicating that the task timed out.

#### `outputs` - Defining Output Structure

This section defines the structure of the data that this tool will produce upon successful completion.

```typescript
  outputs: {
    id: { type: 'string', description: 'Task execution identifier' },
    success: { type: 'boolean', description: 'Task completion status' },
    output: { type: 'json', description: 'Task output data' },
    steps: { type: 'json', description: 'Execution steps taken' },
  },
}
```

*   **`outputs`**: Describes the final structured output from the tool.
    *   **`id: { type: 'string', description: 'Task execution identifier' }`**: The unique ID of the BrowserUse task that was executed.
    *   **`success: { type: 'boolean', description: 'Task completion status' }`**: A boolean indicating whether the browser task finished successfully (`true`) or failed/stopped (`false`).
    *   **`output: { type: 'json', description: 'Task output data' }`**: The main data result returned by the BrowserUse task, typically an object or array in JSON format.
    *   **`steps: { type: 'json', description: 'Execution steps taken' }`**: A detailed list of steps the browser agent performed during the task, also in JSON format.

---

### Simplified Complex Logic

1.  **`variables` Parsing (within `request.body`):**
    *   **Why it's complex:** The `variables` parameter, meant for "secrets" or extra data, needs to be flexible. It might come from a user interface where data is entered into a table (which often gets sent as an array of objects) or directly as a JSON object.
    *   **How it's simplified:** The code checks for both scenarios:
        *   **If it's an array:** It loops through each item, looking for keys named `Key` and `Value` either directly on the item or nested within a `cells` property (like `row.cells.Key`). This allows it to extract key-value pairs from different UI-generated array structures.
        *   **If it's an object:** It simply uses the object as-is.
    *   The goal is to convert whatever format `params.variables` comes in into a simple `{ [key: string]: string }` object called `secrets`, which is then sent to the BrowserUse API.

2.  **`postProcess` Polling Loop:**
    *   **Why it's complex:** After initiating a browser task, it doesn't complete instantly. We need to wait for it. The BrowserUse API doesn't use webhooks in this context, so we have to ask it repeatedly for the task's status.
    *   **How it's simplified:** This section implements a standard "polling" pattern:
        *   **Launch and Get ID:** The initial API call (handled by `request` and `transformResponse`) gets a `taskId`.
        *   **Initial `live_url` check:** A quick attempt is made to get a live viewing URL for the task right after it's launched.
        *   **Loop and Wait:** A `while` loop runs for a maximum duration (3 minutes).
        *   **Check Status:** Inside the loop, it periodically (every 5 seconds) calls a specific API endpoint (`/task/${taskId}/status`) to get the task's current state (e.g., 'running', 'finished').
        *   **Act on Status:**
            *   If the task is 'running' and we haven't logged the `live_url` yet, it tries to fetch and log it.
            *   If the task is 'finished', 'failed', or 'stopped', it knows the task is done. It then makes *one final call* to get all the detailed results (`/task/${taskId}`) and updates the tool's output before stopping the loop.
        *   **Error Handling & Timeout:** If the API calls fail or the task takes too long, appropriate errors or warnings are logged, and the process stops to prevent infinite waiting.

This file provides a robust and flexible way to integrate browser automation tasks into a larger system, handling everything from input validation to long-running task monitoring.