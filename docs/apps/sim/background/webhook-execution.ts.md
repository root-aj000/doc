This TypeScript file is a critical backend service responsible for **processing and executing workflows triggered by incoming webhooks**. It acts as the central orchestrator, handling everything from initial payload reception to final workflow execution, logging, and state management.

It's designed to be robust, incorporating features like idempotency to prevent duplicate executions, usage limit checks, secure handling of environment variables, and specialized processing for different webhook providers (like Airtable and WhatsApp).

---

## Simplified Complex Logic

At its core, this file implements a sophisticated pipeline for handling webhook events. Here's a simplified breakdown of the key logical steps:

1.  **Receive Webhook & Initialize:**
    *   An incoming webhook payload (body, headers, path, etc.) is received.
    *   A unique `executionId` and a short `requestId` for logging are generated.

2.  **Idempotency Check:**
    *   Before any heavy processing, an idempotency key is created (based on `webhookId` and headers).
    *   This key is used to ensure that if the same webhook is sent multiple times (e.g., due to network retries), the workflow only executes once. If it's a duplicate, the previously recorded result is returned.

3.  **Internal Execution (`executeWebhookJobInternal`):**
    *   This is where the actual workflow execution logic lives.
    *   **Start Logging:** A `LoggingSession` is initialized to track all events during this execution.
    *   **Usage Limits:** The system first checks if the user associated with the workflow has exceeded their usage limits. If so, execution is aborted.
    *   **Load Workflow Definition:** The workflow's structure (blocks, edges, loops, parallels) is loaded from the database. It can load either the "live" (latest saved) or "deployed" state.
    *   **Environment Variables:** Personal and workspace-level encrypted environment variables are fetched from the database, decrypted, and made available to the workflow.
    *   **Prepare Workflow State:** The raw workflow definition is transformed and "serialized" into a format that the `Executor` (the workflow engine) can understand. This involves merging sub-block states and organizing block configurations.
    *   **Input Formatting & Attachment Processing:**
        *   The raw webhook `body` and `headers` are formatted into a structured `input` object that the workflow expects.
        *   **Special Provider Handling:**
            *   **Airtable:** If the provider is Airtable, a highly specific logic path is taken to fetch and process Airtable change payloads. The workflow only executes if actual changes are detected.
            *   **WhatsApp:** If WhatsApp, and no messages are found in the payload, execution is skipped.
        *   **File/Attachment Handling:** For standard webhooks, the `input` is scanned for fields marked as `file` or `file[]` (based on the trigger's schema or generic input format). Any identified files are then processed, uploaded to execution storage, and replaced with references.
    *   **Execute Workflow:**
        *   An `Executor` instance is created, configured with the serialized workflow, current block states, decrypted environment variables, and the prepared webhook `input`.
        *   The `Executor` then runs the workflow logic, traversing blocks, executing actions, and managing state transitions.
    *   **Post-Execution:**
        *   **Update Stats:** If the workflow executed successfully, counters for webhook triggers and user activity are updated in the database.
        *   **Complete Logging:** The `LoggingSession` is completed, capturing all logs, outputs, and performance metrics (trace spans) from the workflow execution.
        *   **Return Result:** A summary of the execution (success status, output, duration) is returned.
    *   **Error Handling:** A comprehensive `catch` block ensures that any errors during execution are logged and the `LoggingSession` is completed with an error state.

4.  **Trigger.dev Task Integration:**
    *   The entire process is wrapped in a `task` definition from the `@trigger.dev/sdk`. This means `executeWebhookJob` is designed to be run as a background job within the Trigger.dev ecosystem, allowing for retries and asynchronous execution.

---

## Detailed Line-by-Line Explanation

```typescript
import { db } from '@sim/db'
import { userStats, webhook, workflow as workflowTable } from '@sim/db/schema'
import { task } from '@trigger.dev/sdk'
import { eq, sql } from 'drizzle-orm'
import { v4 as uuidv4 } from 'uuid'
import { checkServerSideUsageLimits } from '@/lib/billing'
import { getPersonalAndWorkspaceEnv } from '@/lib/environment/utils'
import { processExecutionFiles } from '@/lib/execution/files'
import { IdempotencyService, webhookIdempotency } from '@/lib/idempotency'
import { createLogger } from '@/lib/logs/console/logger'
import { LoggingSession } from '@/lib/logs/execution/logging-session'
import { buildTraceSpans } from '@/lib/logs/execution/trace-spans/trace-spans'
import { decryptSecret } from '@/lib/utils'
import { WebhookAttachmentProcessor } from '@/lib/webhooks/attachment-processor'
import { fetchAndProcessAirtablePayloads, formatWebhookInput } from '@/lib/webhooks/utils'
import {
  loadDeployedWorkflowState,
  loadWorkflowFromNormalizedTables,
} from '@/lib/workflows/db-helpers'
import { updateWorkflowRunCounts } from '@/lib/workflows/utils'
import { Executor } from '@/executor'
import type { ExecutionResult } from '@/executor/types'
import { Serializer } from '@/serializer'
import { mergeSubblockState } from '@/stores/workflows/server-utils'
import { getTrigger } from '@/triggers'
```
**Imports:** These lines import various modules and utilities crucial for the webhook execution process. They are categorized for clarity:
*   `@sim/db`:
    *   `db`: Imports the Drizzle ORM database client. This is the main interface for interacting with the application's database.
    *   `userStats`, `webhook`, `workflow as workflowTable`: Imports specific table schemas from the database for `userStats` (to track user activity), `webhook` (to retrieve webhook configurations), and `workflow` (aliased as `workflowTable` to avoid naming conflicts, for workflow definitions).
*   `@trigger.dev/sdk`:
    *   `task`: Imports the `task` decorator from the Trigger.dev SDK. This is used to define a background job or task that can be executed by the Trigger.dev runtime.
*   `drizzle-orm`:
    *   `eq`, `sql`: Imports Drizzle ORM functions for creating equality conditions (`eq`) in queries and for writing raw SQL expressions (`sql`), often used for increments or date functions.
*   `uuid`:
    *   `v4 as uuidv4`: Imports the `v4` function from the `uuid` library, aliased as `uuidv4`, to generate universally unique identifiers.
*   `@/lib/billing`:
    *   `checkServerSideUsageLimits`: Imports a utility function to verify if a user has exceeded their platform usage limits.
*   `@/lib/environment/utils`:
    *   `getPersonalAndWorkspaceEnv`: Imports a function to retrieve encrypted environment variables specific to a user and their workspace.
*   `@/lib/execution/files`:
    *   `processExecutionFiles`: Imports a utility for processing files associated with an execution, likely for uploading them to storage.
*   `@/lib/idempotency`:
    *   `IdempotencyService`, `webhookIdempotency`: Imports services related to idempotency, a mechanism to ensure that an operation performed multiple times has the same effect as performing it once. `IdempotencyService` likely provides key generation, and `webhookIdempotency` is a specific wrapper for webhooks.
*   `@/lib/logs/console/logger`:
    *   `createLogger`: Imports a factory function to create a logger instance for structured logging.
*   `@/lib/logs/execution/logging-session`:
    *   `LoggingSession`: Imports a class for managing the lifecycle of an execution-specific logging session.
*   `@/lib/logs/execution/trace-spans/trace-spans`:
    *   `buildTraceSpans`: Imports a function to convert raw execution logs into a structured format suitable for distributed tracing (spans).
*   `@/lib/utils`:
    *   `decryptSecret`: Imports a utility function to decrypt sensitive information (secrets).
*   `@/lib/webhooks/attachment-processor`:
    *   `WebhookAttachmentProcessor`: Imports a service dedicated to processing attachments received via webhooks.
*   `@/lib/webhooks/utils`:
    *   `fetchAndProcessAirtablePayloads`, `formatWebhookInput`: Imports utilities specifically for handling Airtable webhook data and for general webhook input formatting.
*   `@/lib/workflows/db-helpers`:
    *   `loadDeployedWorkflowState`, `loadWorkflowFromNormalizedTables`: Imports helper functions to load workflow definitions from the database, either in their "deployed" (production) state or "live" (latest saved) state.
*   `@/lib/workflows/utils`:
    *   `updateWorkflowRunCounts`: Imports a utility to update counters related to workflow executions.
*   `@/executor`:
    *   `Executor`: Imports the core workflow execution engine class.
    *   `type ExecutionResult`: Imports the type definition for the result returned by the `Executor`.
*   `@/serializer`:
    *   `Serializer`: Imports a class responsible for transforming the workflow definition into an executable format.
*   `@/stores/workflows/server-utils`:
    *   `mergeSubblockState`: Imports a utility to combine different parts of a workflow's block state.
*   `@/triggers`:
    *   `getTrigger`: Imports a function to retrieve the configuration for a specific trigger block.

```typescript
const logger = createLogger('TriggerWebhookExecution')
```
**Logger Initialization:**
*   `const logger = createLogger('TriggerWebhookExecution')`: Initializes a logger instance with the name 'TriggerWebhookExecution'. This logger will be used throughout the file to record events, debug information, warnings, and errors.

```typescript
/**
 * Process trigger outputs based on their schema definitions
 * Finds outputs marked as 'file' or 'file[]' and uploads them to execution storage
 */
async function processTriggerFileOutputs(
  input: any,
  triggerOutputs: Record<string, any>,
  context: {
    workspaceId: string
    workflowId: string
    executionId: string
    requestId: string
  },
  path = ''
): Promise<any> {
  if (!input || typeof input !== 'object') {
    return input
  }

  const processed: any = Array.isArray(input) ? [] : {}

  for (const [key, value] of Object.entries(input)) {
    const currentPath = path ? `${path}.${key}` : key
    const outputDef = triggerOutputs[key]
    const val: any = value

    // If this field is marked as file or file[], process it
    if (outputDef?.type === 'file[]' && Array.isArray(val)) {
      try {
        processed[key] = await WebhookAttachmentProcessor.processAttachments(val as any, context)
      } catch (error) {
        processed[key] = []
      }
    } else if (outputDef?.type === 'file' && val) {
      try {
        const [processedFile] = await WebhookAttachmentProcessor.processAttachments(
          [val as any],
          context
        )
        processed[key] = processedFile
      } catch (error) {
        logger.error(`[${context.requestId}] Error processing ${currentPath}:`, error)
        processed[key] = val
      }
    } else if (outputDef && typeof outputDef === 'object' && !outputDef.type) {
      // Nested object in schema - recurse with the nested schema
      processed[key] = await processTriggerFileOutputs(val, outputDef, context, currentPath)
    } else {
      // Not a file output - keep as is
      processed[key] = val
    }
  }

  return processed
}
```
**`processTriggerFileOutputs` Function:**
*   This asynchronous function recursively processes the input data from a webhook, specifically looking for fields that are defined as `file` or `file[]` types in the trigger's schema. If found, these files are processed (e.g., uploaded to storage).
*   `input: any`: The data received from the webhook, potentially containing files.
*   `triggerOutputs: Record<string, any>`: The schema definition for the trigger's outputs, which specifies the types of each field.
*   `context: { ... }`: An object providing contextual information for file processing, including `workspaceId`, `workflowId`, `executionId`, and `requestId`.
*   `path = ''`: An optional parameter to track the current path within the nested input object for logging purposes.
*   `if (!input || typeof input !== 'object') { return input }`: Base case for recursion: if the input is not an object or is null/undefined, return it directly.
*   `const processed: any = Array.isArray(input) ? [] : {}`: Initializes `processed` as either an array or an object, matching the type of the `input`, to store the processed data.
*   `for (const [key, value] of Object.entries(input))`: Iterates over each key-value pair in the `input` object.
    *   `const currentPath = path ? `${path}.${key}` : key`: Constructs the full path for the current key, useful for detailed error logging.
    *   `const outputDef = triggerOutputs[key]`: Retrieves the schema definition for the current `key` from `triggerOutputs`.
    *   `const val: any = value`: A type assertion for convenience.
    *   `if (outputDef?.type === 'file[]' && Array.isArray(val))`: Checks if the schema defines this field as an array of files (`file[]`) and if the actual value is an array.
        *   `try { processed[key] = await WebhookAttachmentProcessor.processAttachments(val as any, context) }`: If it's `file[]`, it calls `WebhookAttachmentProcessor.processAttachments` to handle the array of files. The result (e.g., URLs to uploaded files) is stored.
        *   `catch (error) { processed[key] = [] }`: If processing fails, the field is set to an empty array to prevent further errors.
    *   `else if (outputDef?.type === 'file' && val)`: Checks if the schema defines this field as a single file (`file`) and if a value exists.
        *   `try { const [processedFile] = await WebhookAttachmentProcessor.processAttachments([val as any], context) processed[key] = processedFile }`: If it's a single file, it's wrapped in an array and passed to `processAttachments`. The first (and only) processed file is then assigned.
        *   `catch (error) { logger.error(...); processed[key] = val }`: Logs any error during single file processing and keeps the original value in case of failure.
    *   `else if (outputDef && typeof outputDef === 'object' && !outputDef.type)`: Checks if `outputDef` exists, is an object, and *doesn't* have a `type` property. This indicates a nested object in the schema.
        *   `processed[key] = await processTriggerFileOutputs(val, outputDef, context, currentPath)`: Recursively calls `processTriggerFileOutputs` for the nested object.
    *   `else { processed[key] = val }`: If the field is not a file type or a nested object, its value is kept as is.
*   `return processed`: Returns the processed input object.

```typescript
export type WebhookExecutionPayload = {
  webhookId: string
  workflowId: string
  userId: string
  provider: string
  body: any
  headers: Record<string, string>
  path: string
  blockId?: string
  testMode?: boolean
  executionTarget?: 'deployed' | 'live'
  credentialId?: string
}
```
**`WebhookExecutionPayload` Type:**
*   This type defines the structure of the payload that initiates a webhook workflow execution.
*   `webhookId: string`: The ID of the specific webhook configuration that triggered this execution.
*   `workflowId: string`: The ID of the workflow to be executed.
*   `userId: string`: The ID of the user who owns the workflow.
*   `provider: string`: The name of the webhook provider (e.g., 'airtable', 'slack', 'generic').
*   `body: any`: The raw body of the incoming HTTP request.
*   `headers: Record<string, string>`: The HTTP headers of the incoming request.
*   `path: string`: The path part of the incoming webhook URL.
*   `blockId?: string`: (Optional) The ID of the specific block within the workflow that triggered the execution (e.g., a webhook trigger block).
*   `testMode?: boolean`: (Optional) A flag indicating if this is a test execution.
*   `executionTarget?: 'deployed' | 'live'`: (Optional) Specifies whether to load the 'deployed' (production) version of the workflow or the 'live' (latest saved) version.
*   `credentialId?: string`: (Optional) The ID of any associated credential, if required by the webhook provider.

```typescript
export async function executeWebhookJob(payload: WebhookExecutionPayload) {
  const executionId = uuidv4()
  const requestId = executionId.slice(0, 8)

  logger.info(`[${requestId}] Starting webhook execution`, {
    webhookId: payload.webhookId,
    workflowId: payload.workflowId,
    provider: payload.provider,
    userId: payload.userId,
    executionId,
  })

  const idempotencyKey = IdempotencyService.createWebhookIdempotencyKey(
    payload.webhookId,
    payload.headers
  )

  const runOperation = async () => {
    return await executeWebhookJobInternal(payload, executionId, requestId)
  }

  return await webhookIdempotency.executeWithIdempotency(
    payload.provider,
    idempotencyKey,
    runOperation
  )
}
```
**`executeWebhookJob` Function:**
*   This is the entry point for starting a webhook execution, acting as an idempotency wrapper around the core logic.
*   `payload: WebhookExecutionPayload`: The input payload defining the webhook event.
*   `const executionId = uuidv4()`: Generates a unique ID for this specific execution.
*   `const requestId = executionId.slice(0, 8)`: Creates a shorter `requestId` (first 8 characters of `executionId`) for concise logging.
*   `logger.info(...)`: Logs the start of the webhook execution with key details.
*   `const idempotencyKey = IdempotencyService.createWebhookIdempotencyKey(...)`: Generates an idempotency key using the webhook ID and request headers. This key uniquely identifies a specific incoming webhook event.
*   `const runOperation = async () => { ... }`: Defines an asynchronous function `runOperation` that encapsulates the actual workflow execution logic. This function will be passed to the idempotency service.
    *   `return await executeWebhookJobInternal(payload, executionId, requestId)`: Calls the main internal execution function with the payload and generated IDs.
*   `return await webhookIdempotency.executeWithIdempotency(...)`: This is the core idempotency mechanism. It attempts to execute `runOperation`. If an execution with the same `idempotencyKey` has already completed, it will return the cached result instead of running `runOperation` again. This prevents duplicate workflow runs for the same webhook event.

```typescript
async function executeWebhookJobInternal(
  payload: WebhookExecutionPayload,
  executionId: string,
  requestId: string
) {
  const loggingSession = new LoggingSession(payload.workflowId, executionId, 'webhook', requestId)

  try {
    const usageCheck = await checkServerSideUsageLimits(payload.userId)
    if (usageCheck.isExceeded) {
      logger.warn(...)
      throw new Error(...)
    }

    // Load workflow state based on execution target
    const workflowData =
      payload.executionTarget === 'live'
        ? await loadWorkflowFromNormalizedTables(payload.workflowId)
        : await loadDeployedWorkflowState(payload.workflowId)
    if (!workflowData) {
      throw new Error(`Workflow ${payload.workflowId} has no live normalized state`)
    }

    const { blocks, edges, loops, parallels } = workflowData

    const wfRows = await db
      .select({ workspaceId: workflowTable.workspaceId })
      .from(workflowTable)
      .where(eq(workflowTable.id, payload.workflowId))
      .limit(1)
    const workspaceId = wfRows[0]?.workspaceId || undefined

    const { personalEncrypted, workspaceEncrypted } = await getPersonalAndWorkspaceEnv(
      payload.userId,
      workspaceId
    )
    const mergedEncrypted = { ...personalEncrypted, ...workspaceEncrypted }
    const decryptedPairs = await Promise.all(
      Object.entries(mergedEncrypted).map(async ([key, encrypted]) => {
        const { decrypted } = await decryptSecret(encrypted)
        return [key, decrypted] as const
      })
    )
    const decryptedEnvVars: Record<string, string> = Object.fromEntries(decryptedPairs)

    // Start logging session
    await loggingSession.safeStart({
      userId: payload.userId,
      workspaceId: workspaceId || '',
      variables: decryptedEnvVars,
      triggerData: {
        isTest: payload.testMode === true,
        executionTarget: payload.executionTarget || 'deployed',
      },
    })

    // Merge subblock states (matching workflow-execution pattern)
    const mergedStates = mergeSubblockState(blocks, {})

    // Process block states for execution
    const processedBlockStates = Object.entries(mergedStates).reduce(
      (acc, [blockId, blockState]) => {
        acc[blockId] = Object.entries(blockState.subBlocks).reduce(
          (subAcc, [key, subBlock]) => {
            subAcc[key] = subBlock.value
            return subAcc
          },
          {} as Record<string, any>
        )
        return acc
      },
      {} as Record<string, Record<string, any>>
    )

    // Handle workflow variables (for now, use empty object since we don't have workflow metadata)
    const workflowVariables = {}

    // Create serialized workflow
    const serializer = new Serializer()
    const serializedWorkflow = serializer.serializeWorkflow(
      mergedStates,
      edges,
      loops || {},
      parallels || {},
      true // Enable validation during execution
    )

    // Handle special Airtable case
    if (payload.provider === 'airtable') {
      logger.info(`[${requestId}] Processing Airtable webhook via fetchAndProcessAirtablePayloads`)

      // Load the actual webhook record from database to get providerConfig
      const [webhookRecord] = await db
        .select()
        .from(webhook)
        .where(eq(webhook.id, payload.webhookId))
        .limit(1)

      if (!webhookRecord) {
        throw new Error(`Webhook record not found: ${payload.webhookId}`)
      }

      const webhookData = {
        id: payload.webhookId,
        provider: payload.provider,
        providerConfig: webhookRecord.providerConfig,
      }

      // Create a mock workflow object for Airtable processing
      const mockWorkflow = {
        id: payload.workflowId,
        userId: payload.userId,
      }

      // Get the processed Airtable input
      const airtableInput = await fetchAndProcessAirtablePayloads(
        webhookData,
        mockWorkflow,
        requestId
      )

      // If we got input (changes), execute the workflow like other providers
      if (airtableInput) {
        logger.info(`[${requestId}] Executing workflow with Airtable changes`)

        // Create executor and execute (same as standard webhook flow)
        const executor = new Executor({
          workflow: serializedWorkflow,
          currentBlockStates: processedBlockStates,
          envVarValues: decryptedEnvVars,
          workflowInput: airtableInput,
          workflowVariables,
          contextExtensions: {
            executionId,
            workspaceId: workspaceId || '',
            isDeployedContext: !payload.testMode,
          },
        })

        // Set up logging on the executor
        loggingSession.setupExecutor(executor)

        // Execute the workflow
        const result = await executor.execute(payload.workflowId, payload.blockId)

        // Check if we got a StreamingExecution result
        const executionResult =
          'stream' in result && 'execution' in result ? result.execution : result

        logger.info(`[${requestId}] Airtable webhook execution completed`, {
          success: executionResult.success,
          workflowId: payload.workflowId,
        })

        // Update workflow run counts on success
        if (executionResult.success) {
          await updateWorkflowRunCounts(payload.workflowId)

          // Track execution in user stats
          await db
            .update(userStats)
            .set({
              totalWebhookTriggers: sql`total_webhook_triggers + 1`,
              lastActive: sql`now()`,
            })
            .where(eq(userStats.userId, payload.userId))
        }

        // Build trace spans and complete logging session
        const { traceSpans, totalDuration } = buildTraceSpans(executionResult)

        await loggingSession.safeComplete({
          endedAt: new Date().toISOString(),
          totalDurationMs: totalDuration || 0,
          finalOutput: executionResult.output || {},
          traceSpans: traceSpans as any,
          workflowInput: airtableInput,
        })

        return {
          success: executionResult.success,
          workflowId: payload.workflowId,
          executionId,
          output: executionResult.output,
          executedAt: new Date().toISOString(),
          provider: payload.provider,
        }
      }
      // No changes to process
      logger.info(`[${requestId}] No Airtable changes to process`)

      await loggingSession.safeComplete({
        endedAt: new Date().toISOString(),
        totalDurationMs: 0,
        finalOutput: { message: 'No Airtable changes to process' },
        traceSpans: [],
      })

      return {
        success: true,
        workflowId: payload.workflowId,
        executionId,
        output: { message: 'No Airtable changes to process' },
        executedAt: new Date().toISOString(),
      }
    }

    // Format input for standard webhooks
    // Load the actual webhook to get providerConfig (needed for Teams credentialId)
    const webhookRows = await db
      .select()
      .from(webhook)
      .where(eq(webhook.id, payload.webhookId))
      .limit(1)

    const actualWebhook =
      webhookRows.length > 0
        ? webhookRows[0]
        : {
            provider: payload.provider,
            blockId: payload.blockId,
            providerConfig: {},
          }

    const mockWorkflow = {
      id: payload.workflowId,
      userId: payload.userId,
    }
    const mockRequest = {
      headers: new Map(Object.entries(payload.headers)),
    } as any

    const input = await formatWebhookInput(actualWebhook, mockWorkflow, payload.body, mockRequest)

    if (!input && payload.provider === 'whatsapp') {
      logger.info(`[${requestId}] No messages in WhatsApp payload, skipping execution`)
      await loggingSession.safeComplete({
        endedAt: new Date().toISOString(),
        totalDurationMs: 0,
        finalOutput: { message: 'No messages in WhatsApp payload' },
        traceSpans: [],
      })
      return {
        success: true,
        workflowId: payload.workflowId,
        executionId,
        output: { message: 'No messages in WhatsApp payload' },
        executedAt: new Date().toISOString(),
      }
    }

    // Process trigger file outputs based on schema
    if (input && payload.blockId && blocks[payload.blockId]) {
      try {
        const triggerBlock = blocks[payload.blockId]
        const triggerId = triggerBlock?.subBlocks?.triggerId?.value

        if (triggerId && typeof triggerId === 'string') {
          const triggerConfig = getTrigger(triggerId)

          if (triggerConfig?.outputs) {
            logger.debug(`[${requestId}] Processing trigger ${triggerId} file outputs`)
            const processedInput = await processTriggerFileOutputs(input, triggerConfig.outputs, {
              workspaceId: workspaceId || '',
              workflowId: payload.workflowId,
              executionId,
              requestId,
            })
            Object.assign(input, processedInput) // Merge processed files back into input
          }
        }
      } catch (error) {
        logger.error(`[${requestId}] Error processing trigger file outputs:`, error)
        // Continue without processing attachments rather than failing execution
      }
    }

    // Process generic webhook files based on inputFormat
    if (input && payload.provider === 'generic' && payload.blockId && blocks[payload.blockId]) {
      try {
        const triggerBlock = blocks[payload.blockId]

        if (triggerBlock?.subBlocks?.inputFormat?.value) {
          const inputFormat = triggerBlock.subBlocks.inputFormat.value as unknown as Array<{
            name: string
            type: 'string' | 'number' | 'boolean' | 'object' | 'array' | 'files'
          }>
          logger.debug(`[${requestId}] Processing generic webhook files from inputFormat`)

          const fileFields = inputFormat.filter((field) => field.type === 'files')

          if (fileFields.length > 0 && typeof input === 'object' && input !== null) {
            const executionContext = {
              workspaceId: workspaceId || '',
              workflowId: payload.workflowId,
              executionId,
            }

            for (const fileField of fileFields) {
              const fieldValue = input[fileField.name]

              if (fieldValue && typeof fieldValue === 'object') {
                const uploadedFiles = await processExecutionFiles(
                  fieldValue,
                  executionContext,
                  requestId
                )

                if (uploadedFiles.length > 0) {
                  input[fileField.name] = uploadedFiles
                  logger.info(
                    `[${requestId}] Successfully processed ${uploadedFiles.length} file(s) for field: ${fileField.name}`
                  )
                }
              }
            }
          }
        }
      } catch (error) {
        logger.error(`[${requestId}] Error processing generic webhook files:`, error)
        // Continue without processing files rather than failing execution
      }
    }

    // Create executor and execute
    const executor = new Executor({
      workflow: serializedWorkflow,
      currentBlockStates: processedBlockStates,
      envVarValues: decryptedEnvVars,
      workflowInput: input || {},
      workflowVariables,
      contextExtensions: {
        executionId,
        workspaceId: workspaceId || '',
        isDeployedContext: !payload.testMode,
      },
    })

    // Set up logging on the executor
    loggingSession.setupExecutor(executor)

    logger.info(`[${requestId}] Executing workflow for ${payload.provider} webhook`)

    // Execute the workflow
    const result = await executor.execute(payload.workflowId, payload.blockId)

    // Check if we got a StreamingExecution result
    const executionResult = 'stream' in result && 'execution' in result ? result.execution : result

    logger.info(`[${requestId}] Webhook execution completed`, {
      success: executionResult.success,
      workflowId: payload.workflowId,
      provider: payload.provider,
    })

    // Update workflow run counts on success
    if (executionResult.success) {
      await updateWorkflowRunCounts(payload.workflowId)

      // Track execution in user stats (skip in test mode)
      if (!payload.testMode) {
        await db
          .update(userStats)
          .set({
            totalWebhookTriggers: sql`total_webhook_triggers + 1`,
            lastActive: sql`now()`,
          })
          .where(eq(userStats.userId, payload.userId))
      }
    }

    // Build trace spans and complete logging session
    const { traceSpans, totalDuration } = buildTraceSpans(executionResult)

    await loggingSession.safeComplete({
      endedAt: new Date().toISOString(),
      totalDurationMs: totalDuration || 0,
      finalOutput: executionResult.output || {},
      traceSpans: traceSpans as any,
      workflowInput: input,
    })

    return {
      success: executionResult.success,
      workflowId: payload.workflowId,
      executionId,
      output: executionResult.output,
      executedAt: new Date().toISOString(),
      provider: payload.provider,
    }
  } catch (error: any) {
    logger.error(`[${requestId}] Webhook execution failed`, {
      error: error.message,
      stack: error.stack,
      workflowId: payload.workflowId,
      provider: payload.provider,
    })

    // Complete logging session with error (matching workflow-execution pattern)
    try {
      const executionResult = (error?.executionResult as ExecutionResult | undefined) || {
        success: false,
        output: {},
        logs: [],
      }
      const { traceSpans } = buildTraceSpans(executionResult)

      await loggingSession.safeCompleteWithError({
        endedAt: new Date().toISOString(),
        totalDurationMs: 0,
        error: {
          message: error.message || 'Webhook execution failed',
          stackTrace: error.stack,
        },
        traceSpans,
      })
    } catch (loggingError) {
      logger.error(`[${requestId}] Failed to complete logging session`, loggingError)
    }

    throw error
  }
}
```
**`executeWebhookJobInternal` Function:**
*   This is the core logic function where the webhook-triggered workflow execution actually happens.
*   `loggingSession = new LoggingSession(...)`: Creates a new logging session for this specific workflow run, associating it with the workflow, execution, and request IDs.
*   `try { ... } catch (error: any) { ... }`: The entire execution logic is wrapped in a try-catch block to gracefully handle errors and ensure proper logging completion even on failure.
*   `const usageCheck = await checkServerSideUsageLimits(payload.userId)`: Checks if the user associated with this workflow has exceeded their allowed usage limits.
*   `if (usageCheck.isExceeded) { ... throw new Error(...) }`: If limits are exceeded, a warning is logged, and an error is thrown, aborting the execution.
*   `const workflowData = payload.executionTarget === 'live' ? ... : ...`: Loads the workflow definition (`blocks`, `edges`, `loops`, `parallels`) from the database. It decides whether to load the 'live' (latest saved) or 'deployed' (production-ready) version based on `payload.executionTarget`.
*   `if (!workflowData) { ... throw new Error(...) }`: Throws an error if the workflow data cannot be found.
*   `const { blocks, edges, loops, parallels } = workflowData`: Destructures the loaded workflow data.
*   `const wfRows = await db.select(...).from(workflowTable).where(...).limit(1)`: Queries the `workflowTable` to fetch the `workspaceId` associated with the workflow.
*   `const workspaceId = wfRows[0]?.workspaceId || undefined`: Extracts the `workspaceId`.
*   `const { personalEncrypted, workspaceEncrypted } = await getPersonalAndWorkspaceEnv(...)`: Retrieves encrypted environment variables belonging to the user and their workspace.
*   `const mergedEncrypted = { ...personalEncrypted, ...workspaceEncrypted }`: Combines personal and workspace encrypted environment variables.
*   `const decryptedPairs = await Promise.all(...)`: Iterates through the merged encrypted variables, decrypting each one using `decryptSecret`.
*   `const decryptedEnvVars: Record<string, string> = Object.fromEntries(decryptedPairs)`: Converts the array of `[key, decryptedValue]` pairs into an object, making decrypted environment variables easily accessible.
*   `await loggingSession.safeStart(...)`: Starts the logging session, passing in user, workspace, decrypted variables, and trigger-specific data (like `isTest` mode).
*   `const mergedStates = mergeSubblockState(blocks, {})`: Merges the states of sub-blocks within the workflow definition. This is a common step in preparing workflow data for execution.
*   `const processedBlockStates = Object.entries(mergedStates).reduce(...)`: Transforms the `mergedStates` into a simpler format (`processedBlockStates`) that the `Executor` can consume, extracting just the `value` from each sub-block.
*   `const workflowVariables = {}`: Initializes an empty object for workflow variables. This might be a placeholder for future functionality.
*   `const serializer = new Serializer()`: Creates an instance of the `Serializer`.
*   `const serializedWorkflow = serializer.serializeWorkflow(...)`: Serializes the workflow definition (blocks, edges, loops, parallels) into a structured, executable format for the `Executor`. `true` enables validation.
*   `if (payload.provider === 'airtable') { ... }`: **Airtable Specific Logic.** This block handles webhooks coming from Airtable, which often require special processing.
    *   `logger.info(...)`: Logs that Airtable processing is starting.
    *   `const [webhookRecord] = await db.select().from(webhook).where(eq(webhook.id, payload.webhookId)).limit(1)`: Fetches the specific webhook record from the database to get its `providerConfig`.
    *   `if (!webhookRecord) { ... throw new Error(...) }`: Throws an error if the webhook record is not found.
    *   `const webhookData = { ... }`: Creates a simplified `webhookData` object.
    *   `const mockWorkflow = { ... }`: Creates a minimal workflow object needed for Airtable processing.
    *   `const airtableInput = await fetchAndProcessAirtablePayloads(...)`: Calls a specialized utility to fetch and process Airtable payloads, determining if there are relevant changes.
    *   `if (airtableInput) { ... }`: If `airtableInput` (meaning changes were detected) exists, the workflow proceeds with execution.
        *   `logger.info(...)`: Logs that Airtable workflow execution is beginning.
        *   `const executor = new Executor(...)`: Instantiates the `Executor` with the serialized workflow, block states, environment variables, and the `airtableInput`. `contextExtensions` provides additional context.
        *   `loggingSession.setupExecutor(executor)`: Integrates the executor with the logging session so executor-level logs are captured.
        *   `const result = await executor.execute(...)`: Runs the workflow.
        *   `const executionResult = 'stream' in result && 'execution' in result ? result.execution : result`: Handles potential differences in `Executor` result types (e.g., streaming vs. direct).
        *   `logger.info(...)`: Logs the completion of Airtable execution.
        *   `if (executionResult.success) { await updateWorkflowRunCounts(...) await db.update(userStats).set(...).where(...) }`: If successful, updates the workflow run count and increments `totalWebhookTriggers` for the user.
        *   `const { traceSpans, totalDuration } = buildTraceSpans(executionResult)`: Processes the execution result to generate trace spans for detailed logging.
        *   `await loggingSession.safeComplete(...)`: Completes the logging session with the final output and trace spans.
        *   `return { ... }`: Returns a structured result for the Airtable execution.
    *   `else { ... logger.info(...) await loggingSession.safeComplete(...) return { ... } }`: If `airtableInput` is empty (no changes detected), logs that no changes were processed, completes the logging session with a message, and returns a successful but empty result.
*   `const webhookRows = await db.select().from(webhook).where(eq(webhook.id, payload.webhookId)).limit(1)`: For non-Airtable webhooks, fetches the `webhook` record to get `providerConfig`.
*   `const actualWebhook = webhookRows.length > 0 ? webhookRows[0] : { ... }`: Uses the fetched webhook or creates a default mock if not found.
*   `const mockWorkflow = { ... }`: Creates a minimal workflow object for input formatting.
*   `const mockRequest = { ... }`: Creates a mock request object with headers.
*   `const input = await formatWebhookInput(actualWebhook, mockWorkflow, payload.body, mockRequest)`: Formats the raw webhook `body` and `headers` into a structured `input` object expected by the workflow.
*   `if (!input && payload.provider === 'whatsapp') { ... }`: **WhatsApp Specific Logic.** If the provider is WhatsApp and `input` is empty (meaning no relevant messages), logs that execution is skipped, completes logging, and returns.
*   `if (input && payload.blockId && blocks[payload.blockId]) { ... }`: **Process Trigger File Outputs.** This block processes files based on the *trigger's specific schema*.
    *   `const triggerBlock = blocks[payload.blockId]`: Gets the trigger block definition.
    *   `const triggerId = triggerBlock?.subBlocks?.triggerId?.value`: Extracts the `triggerId`.
    *   `if (triggerId && typeof triggerId === 'string') { ... }`: If a `triggerId` exists:
        *   `const triggerConfig = getTrigger(triggerId)`: Retrieves the configuration for that trigger.
        *   `if (triggerConfig?.outputs) { ... const processedInput = await processTriggerFileOutputs(...) Object.assign(input, processedInput) }`: If the trigger has `outputs` (schema), it calls `processTriggerFileOutputs` to handle any files defined in that schema and merges the processed files back into the `input`.
    *   `catch (error) { ... logger.error(...) }`: Logs errors during file output processing but continues execution, meaning file processing is not critical for the entire workflow to run.
*   `if (input && payload.provider === 'generic' && payload.blockId && blocks[payload.blockId]) { ... }`: **Process Generic Webhook Files.** This block handles file uploads specifically for generic webhooks where the file fields are defined in the `inputFormat` of the trigger block itself.
    *   `const inputFormat = triggerBlock.subBlocks.inputFormat.value as unknown as Array<{ ... }> `: Retrieves the `inputFormat` definition from the trigger block.
    *   `const fileFields = inputFormat.filter((field) => field.type === 'files')`: Filters `inputFormat` to find fields explicitly typed as 'files'.
    *   `if (fileFields.length > 0 && typeof input === 'object' && input !== null) { ... }`: If file fields are found and `input` is an object:
        *   `const executionContext = { ... }`: Creates an execution context for `processExecutionFiles`.
        *   `for (const fileField of fileFields) { ... const uploadedFiles = await processExecutionFiles(...) if (uploadedFiles.length > 0) { input[fileField.name] = uploadedFiles } }`: Iterates through each file field, calls `processExecutionFiles` to handle the actual file uploads, and updates the `input` with the results.
    *   `catch (error) { ... logger.error(...) }`: Logs errors but continues execution.
*   `const executor = new Executor(...)`: Instantiates the `Executor` again for standard webhooks (similar to the Airtable block).
*   `loggingSession.setupExecutor(executor)`: Connects the executor to the logging session.
*   `logger.info(...)`: Logs that standard workflow execution is starting.
*   `const result = await executor.execute(payload.workflowId, payload.blockId)`: Executes the workflow.
*   `const executionResult = 'stream' in result && 'execution' in result ? result.execution : result`: Handles potential streaming results.
*   `logger.info(...)`: Logs the completion of standard webhook execution.
*   `if (executionResult.success) { await updateWorkflowRunCounts(...) if (!payload.testMode) { await db.update(userStats).set(...).where(...) } }`: On successful execution, updates workflow run counts and user stats (skipping user stats in `testMode`).
*   `const { traceSpans, totalDuration } = buildTraceSpans(executionResult)`: Builds trace spans from the execution result.
*   `await loggingSession.safeComplete(...)`: Completes the logging session with the final output and trace spans.
*   `return { ... }`: Returns a structured execution result.
*   `catch (error: any) { ... }`: **Error Handling Block.**
    *   `logger.error(...)`: Logs the error details, including message and stack trace.
    *   `try { ... await loggingSession.safeCompleteWithError(...) } catch (loggingError) { ... }`: Attempts to complete the logging session with an error. This nested try-catch ensures that logging errors themselves don't prevent the primary error from being thrown. It also attempts to build trace spans even from a failed execution result if available.
    *   `throw error`: Re-throws the original error to propagate it up the call stack, indicating that the job failed.

```typescript
export const webhookExecution = task({
  id: 'webhook-execution',
  retry: {
    maxAttempts: 1,
  },
  run: async (payload: WebhookExecutionPayload) => executeWebhookJob(payload),
})
```
**`webhookExecution` Task Definition:**
*   `export const webhookExecution = task({ ... })`: This line defines `webhookExecution` as a background task using the `@trigger.dev/sdk`.
*   `id: 'webhook-execution'`: Assigns a unique identifier to this task.
*   `retry: { maxAttempts: 1 }`: Configures the task to retry only once if it fails.
*   `run: async (payload: WebhookExecutionPayload) => executeWebhookJob(payload)`: Specifies the function to be executed when this task runs. It calls `executeWebhookJob` (the idempotency wrapper) with the provided `payload`. This setup allows the webhook processing logic to run reliably as an asynchronous background job within the Trigger.dev platform.