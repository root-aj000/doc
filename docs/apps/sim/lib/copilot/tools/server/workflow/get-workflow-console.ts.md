This TypeScript file defines a server-side tool responsible for retrieving, processing, and formatting execution logs for a specific workflow. Essentially, it provides a "console" view of how a workflow ran, including details about individual blocks, potential errors, and final outputs.

### Purpose of this file and what it does

**Purpose:**
The primary purpose of this file is to create a robust and standardized way to fetch and present detailed execution history for workflows. It's designed to be used as a "server tool" within a larger "Copilot" or AI assistant framework, allowing external systems (like an AI model or a UI) to query workflow execution details in a human-readable and structured format.

**What it does:**
1.  **Receives a workflow ID:** It takes the ID of a workflow as input, along with optional parameters like how many recent executions to fetch and whether to include detailed block-level information.
2.  **Queries the database:** It connects to a database (`@sim/db`) and queries the `workflowExecutionLogs` table to find the most recent executions for the given workflow ID.
3.  **Parses raw execution data:** The raw log data often contains complex, nested "trace spans" (representing individual steps or blocks within the workflow) and various error formats. The tool intelligently parses this data.
4.  **Extracts block-level details:** If requested (`includeDetails`), it traverses the trace spans to identify and format individual "block executions" – what each step of the workflow did, when it started/ended, its status, input, output, and any associated costs.
5.  **Identifies and normalizes errors:** It has sophisticated logic to look for error messages across different parts of the raw log data (top-level execution data, specific error details, or within trace spans) and standardizes them into a simple string. It also tries to pinpoint which block caused the error.
6.  **Determines final output:** It attempts to find the most recent successful block's output to represent the overall "output" of the workflow execution.
7.  **Formats output:** It transforms all this raw and processed data into a clean, easy-to-consume `ExecutionEntry` format, providing a summary of each workflow execution.
8.  **Acts as a server tool:** It exports a `BaseServerTool` object, which means it can be invoked by a higher-level system (e.g., an AI agent) to get insights into workflow runs.

### Simplified Complex Logic

The most complex parts of this file are:

1.  **Handling nested trace spans:** Workflow execution can be represented as a tree of operations (spans). `extractBlockExecutionsFromTraceSpans` recursively flattens this tree to get a list of all distinct block executions.
2.  **Normalizing error messages:** Errors can appear in many shapes (strings, `Error` objects, complex objects, `null`, `undefined`). The `normalizeErrorMessage` function provides a single, robust way to convert any error into a readable string or `undefined` if no error is present.
3.  **Prioritizing error detection:** An error could be in a specific block's log, in a top-level execution summary, or somewhere deep within the trace spans. `deriveExecutionErrorSummary` orchestrates the search for errors, giving precedence to more specific error sources to provide the most accurate error context (e.g., an error explicitly flagged in a block is more informative than a generic workflow error).
4.  **Finding final output:** A workflow might have many blocks producing output. The logic identifies the *last* successful block that produced meaningful output as the overall workflow output.

### Explanation of each line of code

Let's go through the code section by section.

---

#### **Imports**

```typescript
import { db } from '@sim/db'
import { workflowExecutionLogs } from '@sim/db/schema'
import { desc, eq } from 'drizzle-orm'
import type { BaseServerTool } from '@/lib/copilot/tools/server/base-tool'
import { createLogger } from '@/lib/logs/console/logger'
```

*   `import { db } from '@sim/db'`: Imports the database connection instance from the `@sim/db` module. This `db` object is used to interact with the database.
*   `import { workflowExecutionLogs } from '@sim/db/schema'`: Imports the Drizzle ORM schema definition for the `workflowExecutionLogs` table. This schema is used to define the structure of the table and allows type-safe queries.
*   `import { desc, eq } from 'drizzle-orm'`: Imports specific functions from the `drizzle-orm` library.
    *   `desc`: Used to specify descending order for `ORDER BY` clauses in database queries.
    *   `eq`: Used to specify an "equals" condition for `WHERE` clauses (e.g., `workflowId = 'someId'`).
*   `import type { BaseServerTool } from '@/lib/copilot/tools/server/base-tool'`: Imports the `BaseServerTool` type. This type defines the expected structure for a server-side tool in the application's framework (likely a "Copilot" or AI assistant system), ensuring that tools conform to a standard interface. The `type` keyword indicates that it's only imported for type-checking and won't be present in the compiled JavaScript.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a utility function `createLogger` to create a logger instance. This is used for logging information, warnings, and errors during the tool's execution, which is crucial for debugging and monitoring.

---

#### **Interfaces**

These interfaces define the structure of data objects used throughout the file, ensuring type safety and clarity.

```typescript
interface GetWorkflowConsoleArgs {
  workflowId: string
  limit?: number
  includeDetails?: boolean
}
```

*   This interface defines the expected arguments for the `get_workflow_console` server tool.
    *   `workflowId: string`: The unique identifier of the workflow whose execution logs are to be fetched. This is a mandatory string.
    *   `limit?: number`: An optional number specifying the maximum number of recent workflow executions to retrieve. If not provided, a default might be used.
    *   `includeDetails?: boolean`: An optional boolean flag. If `true`, it indicates that the output should include detailed information about individual block executions within each workflow run.

```typescript
interface BlockExecution {
  id: string
  blockId: string
  blockName: string
  blockType: string
  startedAt: string
  endedAt: string
  durationMs: number
  status: 'success' | 'error' | 'skipped'
  errorMessage?: string
  inputData: any
  outputData: any
  cost?: {
    total: number
    input: number
    output: number
    model?: string
    tokens?: { total: number; prompt: number; completion: number }
  }
}
```

*   This interface describes a single execution of a "block" (a step or component) within a workflow.
    *   `id: string`: The unique ID of this specific block execution.
    *   `blockId: string`: The ID of the block template or definition.
    *   `blockName: string`: The human-readable name of the block.
    *   `blockType: string`: The type or category of the block (e.g., 'LLM', 'HTTP_Request', 'Conditional').
    *   `startedAt: string`: Timestamp when the block started execution (ISO 8601 format).
    *   `endedAt: string`: Timestamp when the block finished execution (ISO 8601 format).
    *   `durationMs: number`: The duration of the block's execution in milliseconds.
    *   `status: 'success' | 'error' | 'skipped'`: The outcome of the block's execution.
    *   `errorMessage?: string`: An optional error message if the block failed.
    *   `inputData: any`: The data provided as input to the block. `any` is used because the structure can vary widely.
    *   `outputData: any`: The data produced as output by the block. `any` is used for flexibility.
    *   `cost?: {...}`: An optional object detailing the cost associated with this block execution, often relevant for AI model interactions.
        *   `total`, `input`, `output`: Breakdown of monetary cost.
        *   `model`: The specific AI model used, if applicable.
        *   `tokens`: Further breakdown of token usage for AI models.

```typescript
interface ExecutionEntry {
  id: string
  executionId: string
  level: string
  trigger: string
  startedAt: string
  endedAt: string | null
  durationMs: number | null
  totalCost: number | null
  totalTokens: number | null
  blockExecutions: BlockExecution[]
  output?: any
  errorMessage?: string
  errorBlock?: {
    blockId?: string
    blockName?: string
    blockType?: string
  }
}
```

*   This interface represents a single complete execution of a workflow, formatted for the console output. It aggregates information from the raw log data.
    *   `id: string`: The unique ID of this workflow execution log entry.
    *   `executionId: string`: The overall execution ID for this workflow run (could be the same as `id` or a different identifier).
    *   `level: string`: The logging level (e.g., 'info', 'debug').
    *   `trigger: string`: How the workflow was initiated (e.g., 'manual', 'API', 'schedule').
    *   `startedAt: string`: Timestamp when the workflow execution began (ISO 8601 format).
    *   `endedAt: string | null`: Timestamp when the workflow execution ended, or `null` if still running or ended with an unknown time.
    *   `durationMs: number | null`: Total duration of the workflow execution in milliseconds, or `null`.
    *   `totalCost: number | null`: The total monetary cost associated with this workflow run, or `null`.
    *   `totalTokens: number | null`: The total number of tokens used (e.g., for AI models) in this workflow run, or `null`.
    *   `blockExecutions: BlockExecution[]`: An array of `BlockExecution` objects, detailing each step within the workflow (included only if `includeDetails` is true).
    *   `output?: any`: The final output produced by the workflow, if any.
    *   `errorMessage?: string`: An optional summary error message if the workflow failed.
    *   `errorBlock?: {...}`: An optional object providing context about which block caused the error, if identifiable.
        *   `blockId?`, `blockName?`, `blockType?`: Details of the block that failed.

---

#### **Helper Functions**

These functions encapsulate specific logic for parsing, normalizing, and extracting data from the raw workflow execution logs.

```typescript
function extractBlockExecutionsFromTraceSpans(traceSpans: any[]): BlockExecution[] {
  const blockExecutions: BlockExecution[] = []

  function processSpan(span: any) {
    if (span?.blockId) {
      blockExecutions.push({
        id: span.id,
        blockId: span.blockId,
        blockName: span.name || '',
        blockType: span.type,
        startedAt: span.startTime,
        endedAt: span.endTime,
        durationMs: span.duration || 0,
        status: span.status || 'success',
        errorMessage: span.output?.error || undefined,
        inputData: span.input || {},
        outputData: span.output || {},
        cost: span.cost || undefined,
      })
    }
    if (span?.children && Array.isArray(span.children)) {
      span.children.forEach(processSpan)
    }
  }

  traceSpans.forEach(processSpan)
  return blockExecutions
}
```

*   **`extractBlockExecutionsFromTraceSpans(traceSpans: any[]): BlockExecution[]`**: This function takes an array of `traceSpans` (which can be nested, representing a tree structure of operations) and flattens them into a list of `BlockExecution` objects.
    *   `const blockExecutions: BlockExecution[] = []`: Initializes an empty array to store the extracted `BlockExecution` objects. This array will be populated by the recursive `processSpan` function.
    *   `function processSpan(span: any)`: This is a nested helper function designed to process a single `span` and its children recursively.
        *   `if (span?.blockId)`: Checks if the current `span` object has a `blockId` property. This is the key indicator that this span represents an actual executable block within the workflow.
            *   `blockExecutions.push({...})`: If it's a block, an object conforming to the `BlockExecution` interface is created and pushed into the `blockExecutions` array.
                *   `id: span.id`, `blockId: span.blockId`, `blockName: span.name || ''`, `blockType: span.type`, `startedAt: span.startTime`, `endedAt: span.endTime`: These fields are directly mapped from the `span` object. `blockName` defaults to an empty string if `span.name` is missing.
                *   `durationMs: span.duration || 0`: Maps `span.duration`, defaulting to `0` if it's missing.
                *   `status: span.status || 'success'`: Maps `span.status`, defaulting to `'success'` if not explicitly set (optimistic assumption).
                *   `errorMessage: span.output?.error || undefined`: Extracts an error message from `span.output.error`, or sets it to `undefined` if not present.
                *   `inputData: span.input || {}`, `outputData: span.output || {}`: Maps input and output data, defaulting to an empty object if not present.
                *   `cost: span.cost || undefined`: Maps cost information, or sets it to `undefined`.
        *   `if (span?.children && Array.isArray(span.children))`: Checks if the current `span` has a `children` property and if it's an array. This indicates nested operations.
            *   `span.children.forEach(processSpan)`: If children exist, the `processSpan` function is called recursively for each child span, effectively traversing the tree structure.
    *   `traceSpans.forEach(processSpan)`: The main `traceSpans` array (the top-level spans) is iterated, initiating the recursive processing.
    *   `return blockExecutions`: Returns the populated array of `BlockExecution` objects.

```typescript
function normalizeErrorMessage(errorValue: unknown): string | undefined {
  if (!errorValue) return undefined
  if (typeof errorValue === 'string') return errorValue
  if (errorValue instanceof Error) return errorValue.message
  if (typeof errorValue === 'object') {
    try {
      return JSON.stringify(errorValue)
    } catch {}
  }
  try {
    return String(errorValue)
  } catch {
    return undefined
  }
}
```

*   **`normalizeErrorMessage(errorValue: unknown): string | undefined`**: This utility function attempts to convert various forms of error values into a consistent string format, or `undefined` if it cannot be normalized.
    *   `if (!errorValue) return undefined`: If `errorValue` is `null`, `undefined`, `0`, `false`, or an empty string, it's considered no error, and `undefined` is returned.
    *   `if (typeof errorValue === 'string') return errorValue`: If the error is already a string, it's returned directly.
    *   `if (errorValue instanceof Error) return errorValue.message`: If `errorValue` is an instance of JavaScript's `Error` class, its `message` property is returned.
    *   `if (typeof errorValue === 'object')`: If it's a generic object (not an `Error` instance).
        *   `try { return JSON.stringify(errorValue) } catch {}`: It attempts to convert the object into a JSON string. If `JSON.stringify` fails (e.g., due to circular references), the `catch` block silently handles the error and the function proceeds.
    *   `try { return String(errorValue) } catch { return undefined }`: As a last resort, it attempts to convert the value to a string using `String()`. If even this fails (which is rare but possible with exotic types), `undefined` is returned. This ensures the function is very robust.

```typescript
function extractErrorFromExecutionData(executionData: any): ExecutionEntry['errorBlock'] & {
  message?: string
} {
  if (!executionData) return {}

  const errorDetails = executionData.errorDetails
  if (errorDetails) {
    const message = normalizeErrorMessage(errorDetails.error || errorDetails.message)
    if (message) {
      return {
        message,
        blockId: errorDetails.blockId,
        blockName: errorDetails.blockName,
        blockType: errorDetails.blockType,
      }
    }
  }

  const finalOutputError = normalizeErrorMessage(executionData.finalOutput?.error)
  if (finalOutputError) {
    return {
      message: finalOutputError,
      blockName: 'Workflow',
    }
  }

  const genericError = normalizeErrorMessage(executionData.error)
  if (genericError) {
    return {
      message: genericError,
      blockName: 'Workflow',
    }
  }

  return {}
}
```

*   **`extractErrorFromExecutionData(executionData: any): ...`**: This function tries to find and format an error message and its associated block details from the top-level `executionData` object. It prioritizes more specific error locations.
    *   `if (!executionData) return {}`: If no `executionData` is provided, return an empty object.
    *   `const errorDetails = executionData.errorDetails`: Tries to get error details from a specific `errorDetails` property within `executionData`. This is often the most precise source of error information.
    *   `if (errorDetails)`: If `errorDetails` exist:
        *   `const message = normalizeErrorMessage(errorDetails.error || errorDetails.message)`: Normalizes the error, prioritizing `errorDetails.error` over `errorDetails.message`.
        *   `if (message)`: If a valid message is extracted, return it along with `blockId`, `blockName`, and `blockType` from `errorDetails`.
    *   `const finalOutputError = normalizeErrorMessage(executionData.finalOutput?.error)`: If `errorDetails` didn't yield an error, it checks `executionData.finalOutput.error`.
    *   `if (finalOutputError)`: If an error is found here, return it, attributing it to the 'Workflow' itself.
    *   `const genericError = normalizeErrorMessage(executionData.error)`: Finally, it checks for a generic `error` property directly on `executionData`.
    *   `if (genericError)`: If an error is found, return it, also attributing it to the 'Workflow'.
    *   `return {}`: If no error is found after checking all these locations, an empty object is returned.

```typescript
function extractErrorFromTraceSpans(traceSpans: any[]): ExecutionEntry['errorBlock'] & {
  message?: string
} {
  if (!Array.isArray(traceSpans) || traceSpans.length === 0) return {}

  const queue = [...traceSpans]
  while (queue.length > 0) {
    const span = queue.shift() // Breadth-first search (BFS)

    if (!span || typeof span !== 'object') continue

    const message =
      normalizeErrorMessage(span.output?.error) ||
      normalizeErrorMessage(span.error) ||
      normalizeErrorMessage(span.output?.message) ||
      normalizeErrorMessage(span.message)

    const status = span.status
    if (status === 'error' || message) {
      return {
        message,
        blockId: span.blockId,
        blockName: span.blockName || span.name || (span.blockId ? undefined : 'Workflow'),
        blockType: span.blockType || span.type,
      }
    }

    if (Array.isArray(span.children)) {
      queue.push(...span.children)
    }
  }

  return {}
}
```

*   **`extractErrorFromTraceSpans(traceSpans: any[]): ...`**: This function performs a breadth-first search (BFS) through the `traceSpans` tree to find the *first* span that indicates an error.
    *   `if (!Array.isArray(traceSpans) || traceSpans.length === 0) return {}`: Handles cases where `traceSpans` is not an array or is empty.
    *   `const queue = [...traceSpans]`: Initializes a `queue` with the top-level `traceSpans` to perform a BFS. Using `shift()` later means elements are processed in order of insertion.
    *   `while (queue.length > 0)`: Loops as long as there are spans in the queue to process.
        *   `const span = queue.shift()`: Removes the first `span` from the queue for processing.
        *   `if (!span || typeof span !== 'object') continue`: Skips invalid or non-object spans.
        *   `const message = ...`: This line attempts to find an error message within the current `span` by checking several common locations in a prioritized order: `span.output?.error`, `span.error`, `span.output?.message`, `span.message`. Each attempt uses `normalizeErrorMessage`.
        *   `const status = span.status`: Gets the status of the current span.
        *   `if (status === 'error' || message)`: If the span's `status` is explicitly 'error' OR if an `message` was successfully extracted, then this span is identified as the source of an error.
            *   `return {...}`: Returns the found `message` and attempts to derive the `blockId`, `blockName`, and `blockType` from the span.
                *   `blockName`: Prioritizes `span.blockName`, then `span.name`. If `blockId` exists but no name, it's `undefined`. If no `blockId` or name, it defaults to 'Workflow'. This heuristic tries to provide the best context.
        *   `if (Array.isArray(span.children))`: If the current span has children.
            *   `queue.push(...span.children)`: Adds all children to the end of the queue for later processing (this is what makes it a Breadth-First Search).
    *   `return {}`: If the loop completes without finding any error, an empty object is returned.

```typescript
function deriveExecutionErrorSummary(params: {
  blockExecutions: BlockExecution[]
  traceSpans: any[]
  executionData: any
}): { message?: string; block?: ExecutionEntry['errorBlock'] } {
  const { blockExecutions, traceSpans, executionData } = params

  const blockError = blockExecutions.find((block) => block.status === 'error' && block.errorMessage)
  if (blockError) {
    return {
      message: blockError.errorMessage,
      block: {
        blockId: blockError.blockId,
        blockName: blockError.blockName,
        blockType: blockError.blockType,
      },
    }
  }

  const executionDataError = extractErrorFromExecutionData(executionData)
  if (executionDataError.message) {
    return {
      message: executionDataError.message,
      block: {
        blockId: executionDataError.blockId,
        blockName:
          executionDataError.blockName || (executionDataError.blockId ? undefined : 'Workflow'),
        blockType: executionDataError.blockType,
      },
    }
  }

  const traceError = extractErrorFromTraceSpans(traceSpans)
  if (traceError.message) {
    return {
      message: traceError.message,
      block: {
        blockId: traceError.blockId,
        blockName: traceError.blockName || (traceError.blockId ? undefined : 'Workflow'),
        blockType: traceError.blockType,
      },
    }
  }

  return {}
}
```

*   **`deriveExecutionErrorSummary(params: {...}): {...}`**: This function acts as a central error aggregator, taking different sources of error information and returning the most relevant one, following a specific hierarchy of precedence.
    *   `const { blockExecutions, traceSpans, executionData } = params`: Destructures the input parameters for easier access.
    *   `const blockError = blockExecutions.find(...)`: **Priority 1: Check `blockExecutions` directly.** It searches the pre-extracted `blockExecutions` array for any block that has an 'error' status and a non-empty `errorMessage`. This is often the most direct and specific error.
    *   `if (blockError)`: If a `blockError` is found, its `errorMessage` and relevant `block` details are returned immediately.
    *   `const executionDataError = extractErrorFromExecutionData(executionData)`: **Priority 2: Check `executionData`.** If no block-specific error was found, it calls `extractErrorFromExecutionData` to look for errors in the top-level execution summary.
    *   `if (executionDataError.message)`: If an error is found from `executionData`, its message and block details (with 'Workflow' as a fallback for `blockName`) are returned.
    *   `const traceError = extractErrorFromTraceSpans(traceSpans)`: **Priority 3: Check `traceSpans`.** If still no error, it calls `extractErrorFromTraceSpans` to traverse the raw trace spans for an error.
    *   `if (traceError.message)`: If an error is found from `traceSpans`, its message and block details (with 'Workflow' as a fallback) are returned.
    *   `return {}`: If no error is found across any of these sources, an empty object is returned.

---

#### **Main Server Tool Export**

This is the core of the file, defining the `get_workflow_console` server tool.

```typescript
export const getWorkflowConsoleServerTool: BaseServerTool<GetWorkflowConsoleArgs, any> = {
  name: 'get_workflow_console',
  async execute(rawArgs: GetWorkflowConsoleArgs): Promise<any> {
    const logger = createLogger('GetWorkflowConsoleServerTool')
    const {
      workflowId,
      limit = 3,
      includeDetails = true,
    } = rawArgs || ({} as GetWorkflowConsoleArgs)

    if (!workflowId || typeof workflowId !== 'string') {
      throw new Error('workflowId is required')
    }

    logger.info('Fetching workflow console logs', { workflowId, limit, includeDetails })

    const executionLogs = await db
      .select({
        id: workflowExecutionLogs.id,
        executionId: workflowExecutionLogs.executionId,
        level: workflowExecutionLogs.level,
        trigger: workflowExecutionLogs.trigger,
        startedAt: workflowExecutionLogs.startedAt,
        endedAt: workflowExecutionLogs.endedAt,
        totalDurationMs: workflowExecutionLogs.totalDurationMs,
        executionData: workflowExecutionLogs.executionData,
        cost: workflowExecutionLogs.cost,
      })
      .from(workflowExecutionLogs)
      .where(eq(workflowExecutionLogs.workflowId, workflowId))
      .orderBy(desc(workflowExecutionLogs.startedAt))
      .limit(limit)

    const formattedEntries: ExecutionEntry[] = executionLogs.map((log) => {
      const executionData = log.executionData as any
      const traceSpans = executionData?.traceSpans || []
      const blockExecutions = includeDetails ? extractBlockExecutionsFromTraceSpans(traceSpans) : []

      let finalOutput: any
      if (blockExecutions.length > 0) {
        const sortedBlocks = [...blockExecutions].sort(
          (a, b) => new Date(b.endedAt).getTime() - new Date(a.endedAt).getTime()
        )
        const outputBlock = sortedBlocks.find(
          (block) =>
            block.status === 'success' &&
            block.outputData &&
            Object.keys(block.outputData).length > 0
        )
        if (outputBlock) finalOutput = outputBlock.outputData
      }

      const { message: errorMessage, block: errorBlock } = deriveExecutionErrorSummary({
        blockExecutions,
        traceSpans,
        executionData,
      })

      return {
        id: log.id,
        executionId: log.executionId,
        level: log.level,
        trigger: log.trigger,
        startedAt: log.startedAt.toISOString(),
        endedAt: log.endedAt?.toISOString() || null,
        durationMs: log.totalDurationMs,
        totalCost: (log.cost as any)?.total ?? null,
        totalTokens: (log.cost as any)?.tokens?.total ?? null,
        blockExecutions,
        output: finalOutput,
        errorMessage: errorMessage,
        errorBlock: errorBlock,
      }
    })

    const resultSize = JSON.stringify(formattedEntries).length
    logger.info('Workflow console result prepared', {
      entryCount: formattedEntries.length,
      resultSizeKB: Math.round(resultSize / 1024),
      hasBlockDetails: includeDetails,
    })

    return {
      entries: formattedEntries,
      totalEntries: formattedEntries.length,
      workflowId,
      retrievedAt: new Date().toISOString(),
      hasBlockDetails: includeDetails,
    }
  },
}
```

*   **`export const getWorkflowConsoleServerTool: BaseServerTool<GetWorkflowConsoleArgs, any> = { ... }`**: This line exports the server tool.
    *   `export const getWorkflowConsoleServerTool`: Declares and exports a constant variable.
    *   `: BaseServerTool<GetWorkflowConsoleArgs, any>`: This is the type annotation. It specifies that this object conforms to the `BaseServerTool` interface, accepting `GetWorkflowConsoleArgs` as input and returning `any` as output (though in practice, it returns a specific object structure).
    *   `name: 'get_workflow_console'`: This property gives the tool a unique name. This name is how the tool would be invoked by the AI agent or other systems.

*   **`async execute(rawArgs: GetWorkflowConsoleArgs): Promise<any> { ... }`**: This is the core method that gets executed when the tool is called. It's an `async` function because it performs database operations.
    *   `const logger = createLogger('GetWorkflowConsoleServerTool')`: Creates a logger instance specifically for this tool, which helps in tracing its operations.
    *   `const { workflowId, limit = 3, includeDetails = true, } = rawArgs || ({} as GetWorkflowConsoleArgs)`: Destructures the input `rawArgs`.
        *   `workflowId`: Directly extracted.
        *   `limit = 3`: If `limit` is not provided in `rawArgs`, it defaults to `3`.
        *   `includeDetails = true`: If `includeDetails` is not provided, it defaults to `true`.
        *   `rawArgs || ({} as GetWorkflowConsoleArgs)`: This is a defensive measure. If `rawArgs` itself is `null` or `undefined`, it defaults to an empty object, preventing errors during destructuring.
    *   `if (!workflowId || typeof workflowId !== 'string') { throw new Error('workflowId is required') }`: Input validation: ensures `workflowId` is present and is a string. If not, it throws an error, indicating invalid input.
    *   `logger.info('Fetching workflow console logs', { workflowId, limit, includeDetails })`: Logs an informational message indicating that the tool is starting to fetch logs, including the parameters used.
    *   **Database Query (`await db.select(...)`)**:
        *   `await db.select({...})`: Initiates a Drizzle ORM `SELECT` query.
        *   `id: workflowExecutionLogs.id, ... cost: workflowExecutionLogs.cost,`: Specifies which columns to select from the `workflowExecutionLogs` table. The selected columns directly map to properties that will be needed for `ExecutionEntry`.
        *   `.from(workflowExecutionLogs)`: Specifies the table to query.
        *   `.where(eq(workflowExecutionLogs.workflowId, workflowId))`: Filters the results to only include logs for the specified `workflowId`.
        *   `.orderBy(desc(workflowExecutionLogs.startedAt))`: Sorts the results by `startedAt` date in descending order, meaning the most recent executions come first.
        *   `.limit(limit)`: Limits the number of results returned to the `limit` value (defaulting to 3).
        *   `const executionLogs = ...`: Stores the result of the database query.
    *   **Data Formatting (`executionLogs.map((log) => { ... })`)**:
        *   `const formattedEntries: ExecutionEntry[] = executionLogs.map((log) => { ... })`: Iterates over each raw `log` entry fetched from the database and transforms it into the structured `ExecutionEntry` format.
            *   `const executionData = log.executionData as any`: Extracts the `executionData` column, which is often a JSON blob containing nested details, and casts it to `any` for flexibility.
            *   `const traceSpans = executionData?.traceSpans || []`: Extracts `traceSpans` from `executionData`, defaulting to an empty array if not found.
            *   `const blockExecutions = includeDetails ? extractBlockExecutionsFromTraceSpans(traceSpans) : []`: Conditionally calls `extractBlockExecutionsFromTraceSpans` only if `includeDetails` is `true`. Otherwise, `blockExecutions` remains an empty array.
            *   **`let finalOutput: any; ...`**: Logic to determine the workflow's final output.
                *   `if (blockExecutions.length > 0)`: Only proceeds if block details are available.
                *   `const sortedBlocks = [...blockExecutions].sort(...)`: Creates a shallow copy of `blockExecutions` and sorts them in reverse chronological order based on `endedAt` (most recent first).
                *   `const outputBlock = sortedBlocks.find(...)`: Finds the *first* (most recent due to sorting) block that completed `success`fully and has non-empty `outputData`.
                *   `if (outputBlock) finalOutput = outputBlock.outputData`: If such a block is found, its `outputData` is set as the `finalOutput` for the workflow.
            *   `const { message: errorMessage, block: errorBlock } = deriveExecutionErrorSummary(...)`: Calls the `deriveExecutionErrorSummary` helper function to get the most relevant error message and its associated block context.
            *   **`return { ... }`**: Constructs and returns a single `ExecutionEntry` object for the current `log`.
                *   `id`, `executionId`, `level`, `trigger`: Directly mapped from `log`.
                *   `startedAt: log.startedAt.toISOString()`: Formats `startedAt` to an ISO string.
                *   `endedAt: log.endedAt?.toISOString() || null`: Formats `endedAt` to an ISO string if present, otherwise `null`.
                *   `durationMs: log.totalDurationMs`: Maps total duration.
                *   `totalCost: (log.cost as any)?.total ?? null`: Extracts total cost from the `cost` object, defaulting to `null`. The `??` (nullish coalescing operator) handles `null` or `undefined`.
                *   `totalTokens: (log.cost as any)?.tokens?.total ?? null`: Extracts total tokens from the nested `cost.tokens` object, defaulting to `null`.
                *   `blockExecutions`: Includes the processed `blockExecutions`.
                *   `output: finalOutput`: Includes the derived `finalOutput`.
                *   `errorMessage: errorMessage`, `errorBlock: errorBlock`: Includes the error summary.
    *   **Logging Results (`logger.info(...)`)**:
        *   `const resultSize = JSON.stringify(formattedEntries).length`: Calculates the approximate size of the JSON representation of the results.
        *   `logger.info('Workflow console result prepared', { ... })`: Logs summary information about the prepared results, including count, size, and whether block details were included. This is useful for monitoring performance and data transfer.
    *   **Return Value (`return { ... }`)**:
        *   `return { entries: formattedEntries, ... }`: Returns an object containing the formatted execution entries, their total count, the requested `workflowId`, the timestamp of retrieval, and the `hasBlockDetails` flag. This structured output is what the consuming system (e.g., AI agent) will receive.