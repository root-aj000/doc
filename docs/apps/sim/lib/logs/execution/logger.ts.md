This TypeScript file defines an `ExecutionLogger` service, a central component responsible for recording and managing the lifecycle of workflow executions within a system. It tracks when workflows start, when they complete, their outcomes (success/failure), associated costs, and updates user/organization statistics accordingly. It also extracts important artifacts like attached files from the execution data.

Essentially, this file acts as the primary logging and billing interface for every workflow run, ensuring that all relevant data is persistently stored and that usage is correctly accounted for.

---

## Detailed Explanation

Let's break down this file piece by piece.

### Imports

The file begins by importing various modules and types from internal and external libraries. These imports provide the necessary tools for database interaction, unique ID generation, logging, event emission, billing, and type definitions.

```typescript
import { db } from '@sim/db'
```

*   **`db`**: This imports the Drizzle ORM database client instance. Drizzle is a TypeScript ORM (Object-Relational Mapper) that allows interacting with a database using type-safe TypeScript code instead of raw SQL queries. `@sim/db` suggests a common database module for the application.

```typescript
import {
  member,
  organization,
  userStats,
  user as userTable,
  workflow,
  workflowExecutionLogs,
} from '@sim/db/schema'
```

*   These are Drizzle schema definitions, representing different tables in the database:
    *   **`member`**: Likely defines the structure for records linking users to organizations.
    *   **`organization`**: Defines the structure for organization records, potentially including billing limits.
    *   **`userStats`**: Defines the structure for user statistics, crucial for tracking usage and costs.
    *   **`user as userTable`**: Defines the user table. It's aliased to `userTable` to avoid conflicts with other `user` variables or types in the file.
    *   **`workflow`**: Defines the structure for workflow definitions.
    *   **`workflowExecutionLogs`**: This is the primary table where workflow execution details are stored.

```typescript
import { eq, sql } from 'drizzle-orm'
```

*   **`eq`**: A Drizzle ORM function used to create an "equals" condition in database queries (e.g., `WHERE id = '...'`).
*   **`sql`**: A Drizzle ORM function that allows embedding raw SQL expressions into queries, which is useful for operations like `SUM` or incrementing values atomically.

```typescript
import { v4 as uuidv4 } from 'uuid'
```

*   **`uuidv4`**: Imports the `v4` function from the `uuid` library, used to generate universally unique identifiers (UUIDs), which are often used for primary keys like `executionId`.

```typescript
import { getHighestPrioritySubscription } from '@/lib/billing/core/subscription'
import { checkUsageStatus, maybeSendUsageThresholdEmail } from '@/lib/billing/core/usage'
import { checkAndBillOverageThreshold } from '@/lib/billing/threshold-billing'
import { isBillingEnabled } from '@/lib/environment'
```

*   These imports are all related to the application's billing system:
    *   **`getHighestPrioritySubscription`**: Retrieves the most relevant subscription plan for a given user.
    *   **`checkUsageStatus`**: Checks a user's current usage against their subscription limits.
    *   **`maybeSendUsageThresholdEmail`**: A function that determines if a usage threshold has been crossed and, if so, sends a notification email.
    *   **`checkAndBillOverageThreshold`**: Handles incremental billing for users who exceed their plan limits.
    *   **`isBillingEnabled`**: A boolean flag from the environment configuration, indicating whether billing features are active.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
import { emitWorkflowExecutionCompleted } from '@/lib/logs/events'
import { snapshotService } from '@/lib/logs/execution/snapshot/service'
```

*   **`createLogger`**: A utility to create a logger instance, likely for console and potentially other log destinations.
*   **`emitWorkflowExecutionCompleted`**: A function to emit an event when a workflow execution finishes. This allows other parts of the system to react to completed executions.
*   **`snapshotService`**: A service specifically designed to manage state snapshots of workflows. Snapshots capture the workflow's state at a particular moment, useful for debugging or replay.

```typescript
import type {
  BlockOutputData,
  ExecutionEnvironment,
  ExecutionTrigger,
  ExecutionLoggerService as IExecutionLoggerService,
  TraceSpan,
  WorkflowExecutionLog,
  WorkflowExecutionSnapshot,
  WorkflowState,
} from '@/lib/logs/types'
```

*   These are TypeScript `type` imports, defining the shapes of various data structures used throughout the logger:
    *   **`BlockOutputData`**: The data produced by a workflow block.
    *   **`ExecutionEnvironment`**: Details about the environment in which the workflow ran.
    *   **`ExecutionTrigger`**: Information about what initiated the workflow (e.g., manual, API, schedule).
    *   **`ExecutionLoggerService as IExecutionLoggerService`**: An interface that this `ExecutionLogger` class is expected to implement, ensuring it adheres to a specific contract.
    *   **`TraceSpan`**: Represents a segment of a workflow's execution, useful for tracing and debugging.
    *   **`WorkflowExecutionLog`**: The comprehensive log record for a workflow execution.
    *   **`WorkflowExecutionSnapshot`**: A snapshot of the workflow's state.
    *   **`WorkflowState`**: The current state of the workflow.

### `ToolCall` Interface

```typescript
export interface ToolCall {
  name: string
  duration: number // in milliseconds
  startTime: string // ISO timestamp
  endTime: string // ISO timestamp
  status: 'success' | 'error'
  input?: Record<string, any>
  output?: Record<string, any>
  error?: string
}
```

*   This interface defines the structure of a `ToolCall` object.
*   **Purpose**: `ToolCall` objects are likely nested within `TraceSpan`s and represent individual operations or "tools" invoked during a workflow execution. They provide granular details about each step.
*   **Properties**:
    *   `name`: The name of the tool or operation.
    *   `duration`: How long the tool ran, in milliseconds.
    *   `startTime`, `endTime`: ISO 8601 formatted timestamps marking the beginning and end of the call.
    *   `status`: Indicates if the tool call `success`fully completed or resulted in an `error`.
    *   `input` (optional): The data provided as input to the tool.
    *   `output` (optional): The data produced as output by the tool.
    *   `error` (optional): An error message if the tool call failed.

### Logger Initialization

```typescript
const logger = createLogger('ExecutionLogger')
```

*   This line initializes a logger instance specifically for the `ExecutionLogger` module, giving it the name `'ExecutionLogger'`. This helps in filtering and identifying logs originating from this service.

### `ExecutionLogger` Class

This is the core class of the file, implementing the `IExecutionLoggerService` interface.

```typescript
export class ExecutionLogger implements IExecutionLoggerService {
```

*   **`export class ExecutionLogger`**: Declares a class named `ExecutionLogger` and makes it available for import in other files.
*   **`implements IExecutionLoggerService`**: This clause ensures that the `ExecutionLogger` class provides all the methods and properties defined in the `IExecutionLoggerService` interface. This is a crucial TypeScript feature for enforcing contracts and maintaining code consistency.

#### `startWorkflowExecution` Method

This asynchronous method is responsible for initiating a new workflow execution log record in the database and creating an initial state snapshot.

```typescript
  async startWorkflowExecution(params: {
    workflowId: string
    executionId: string
    trigger: ExecutionTrigger
    environment: ExecutionEnvironment
    workflowState: WorkflowState
  }): Promise<{
    workflowLog: WorkflowExecutionLog
    snapshot: WorkflowExecutionSnapshot
  }> {
```

*   **`async startWorkflowExecution(params: {...})`**: Defines an asynchronous method that accepts a single `params` object containing several key pieces of information about the workflow execution.
*   **Parameters**:
    *   `workflowId`: The ID of the workflow definition being executed.
    *   `executionId`: A unique ID for this specific run of the workflow.
    *   `trigger`: An object describing what initiated the execution (e.g., `type: 'manual'`, `type: 'api'`).
    *   `environment`: Details about the execution environment.
    *   `workflowState`: The initial state of the workflow at the very beginning of its execution.
*   **Return Type**: The method promises to return an object containing the newly created `workflowLog` record and the `snapshot` of the initial state.

```typescript
    const { workflowId, executionId, trigger, environment, workflowState } = params
```

*   **Destructuring `params`**: This line uses object destructuring to extract the individual properties from the `params` object, making them easier to reference throughout the method.

```typescript
    logger.debug(`Starting workflow execution ${executionId} for workflow ${workflowId}`)
```

*   **Logging**: Records a debug message indicating the start of a workflow execution, including its unique `executionId` and the `workflowId` it belongs to.

```typescript
    const snapshotResult = await snapshotService.createSnapshotWithDeduplication(
      workflowId,
      workflowState
    )
```

*   **Create Snapshot**:
    *   Calls `snapshotService.createSnapshotWithDeduplication` to create a record of the workflow's initial `workflowState`.
    *   `WithDeduplication` implies that if an identical `workflowState` has been recorded previously for this `workflowId`, a new snapshot might not be created; instead, the existing one might be referenced to save storage space and avoid redundancy.
    *   The `snapshotResult` object will contain the `snapshot` data, including its ID.

```typescript
    const startTime = new Date()
```

*   **Record Start Time**: Captures the current date and time as the `startTime` for this execution.

```typescript
    const [workflowLog] = await db
      .insert(workflowExecutionLogs)
      .values({
        id: uuidv4(),
        workflowId,
        executionId,
        stateSnapshotId: snapshotResult.snapshot.id,
        level: 'info',
        trigger: trigger.type,
        startedAt: startTime,
        endedAt: null,
        totalDurationMs: null,
        executionData: {
          environment,
          trigger,
        },
      })
      .returning()
```

*   **Insert Log Record**:
    *   `await db.insert(workflowExecutionLogs)`: Initiates an insert operation into the `workflowExecutionLogs` table using Drizzle ORM.
    *   `.values({...})`: Specifies the data to be inserted:
        *   `id: uuidv4()`: Generates a new unique ID for this log entry.
        *   `workflowId`, `executionId`: The IDs passed in the parameters.
        *   `stateSnapshotId`: The ID of the initial state snapshot created earlier.
        *   `level: 'info'`: Sets the initial log level to 'info' (it might be updated to 'error' later if the execution fails).
        *   `trigger: trigger.type`: Stores the type of trigger.
        *   `startedAt: startTime`: The recorded start time.
        *   `endedAt: null`, `totalDurationMs: null`: These are initially null and will be populated when the workflow completes.
        *   `executionData: { environment, trigger }`: Stores additional JSON data related to the execution, including the environment and the full trigger object.
    *   `.returning()`: This Drizzle clause tells the database to return the newly inserted row's data. Since we're inserting a single row, destructuring `[workflowLog]` extracts the first (and only) element from the returned array.

```typescript
    logger.debug(`Created workflow log ${workflowLog.id} for execution ${executionId}`)
```

*   **Logging**: Records a debug message confirming the creation of the workflow log entry, including its ID.

```typescript
    return {
      workflowLog: {
        id: workflowLog.id,
        workflowId: workflowLog.workflowId,
        executionId: workflowLog.executionId,
        stateSnapshotId: workflowLog.stateSnapshotId,
        level: workflowLog.level as 'info' | 'error',
        trigger: workflowLog.trigger as ExecutionTrigger['type'],
        startedAt: workflowLog.startedAt.toISOString(),
        endedAt: workflowLog.endedAt?.toISOString() || workflowLog.startedAt.toISOString(),
        totalDurationMs: workflowLog.totalDurationMs || 0,
        executionData: workflowLog.executionData as WorkflowExecutionLog['executionData'],
        createdAt: workflowLog.createdAt.toISOString(),
      },
      snapshot: snapshotResult.snapshot,
    }
```

*   **Return Value**: Returns an object containing the formatted `workflowLog` and the `snapshot`.
    *   Each property of `workflowLog` is explicitly mapped from the database record.
    *   **Type Assertions (`as ...`)**: Used because Drizzle's generic return types might be broad (`string | null` for `level` or `trigger`), but at this point, we know they conform to the more specific types.
    *   **Date Conversion**: `startedAt.toISOString()` and `endedAt?.toISOString()` convert `Date` objects from the database into ISO 8601 string format, which is a common practice for APIs and logging. The `|| workflowLog.startedAt.toISOString()` provides a fallback for `endedAt` if it's still null.
    *   **Fallback Values**: `totalDurationMs || 0` ensures a default value of 0 if `totalDurationMs` is null.

#### `completeWorkflowExecution` Method

This is the most complex method in the class, responsible for updating a workflow's log record upon completion, determining its final status, recording costs, and handling billing-related notifications and updates.

```typescript
  async completeWorkflowExecution(params: {
    executionId: string
    endedAt: string
    totalDurationMs: number
    costSummary: { /* ... cost details ... */ }
    finalOutput: BlockOutputData
    traceSpans?: TraceSpan[]
    workflowInput?: any
  }): Promise<WorkflowExecutionLog> {
```

*   **`async completeWorkflowExecution(params: {...})`**: An asynchronous method that takes a `params` object with details about the completed execution.
*   **Parameters**:
    *   `executionId`: The ID of the execution being completed.
    *   `endedAt`: An ISO timestamp string indicating when the execution finished.
    *   `totalDurationMs`: The total duration of the execution in milliseconds.
    *   `costSummary`: A detailed object containing all cost-related metrics (total cost, token counts, model-specific costs, etc.). This is critical for billing.
    *   `finalOutput`: The data produced by the workflow at its conclusion.
    *   `traceSpans` (optional): An array of `TraceSpan` objects, providing a detailed trace of execution steps. These are crucial for debugging and error detection.
    *   `workflowInput` (optional): The initial input provided to the workflow.
*   **Return Type**: Promises to return the fully updated `WorkflowExecutionLog` record.

```typescript
    const {
      executionId,
      endedAt,
      totalDurationMs,
      costSummary,
      finalOutput,
      traceSpans,
      workflowInput,
    } = params
```

*   **Destructuring `params`**: Extracts properties for easier access.

```typescript
    logger.debug(`Completing workflow execution ${executionId}`)
```

*   **Logging**: Records a debug message indicating the start of the completion process for a specific `executionId`.

```typescript
    // Determine if workflow failed by checking trace spans for errors
    const hasErrors = traceSpans?.some((span: any) => {
      const checkSpanForErrors = (s: any): boolean => {
        if (s.status === 'error') return true
        if (s.children && Array.isArray(s.children)) {
          return s.children.some(checkSpanForErrors)
        }
        return false
      }
      return checkSpanForErrors(span)
    })
```

*   **Error Detection Logic**:
    *   `traceSpans?.some(...)`: This checks if any of the top-level `traceSpans` indicate an error. The `?` (optional chaining) handles cases where `traceSpans` might be undefined. `some()` returns `true` if at least one element satisfies the condition.
    *   `checkSpanForErrors(s: any)`: This is a recursive helper function:
        *   It first checks if the current `span` (`s`) itself has a `status` of `'error'`.
        *   If not, and if the span has `children` (indicating nested operations) which are an array, it recursively calls `checkSpanForErrors` on each child.
        *   It returns `true` as soon as an error is found anywhere in the span's hierarchy.
    *   **Purpose**: This logic allows the system to determine if the workflow execution failed at any point, even in deeply nested operations, by inspecting the trace data.

```typescript
    const level = hasErrors ? 'error' : 'info'
```

*   **Set Log Level**: Based on whether errors were detected, the `level` for the log record is set to `'error'` or `'info'`.

```typescript
    // Extract files from trace spans, final output, and workflow input
    const executionFiles = this.extractFilesFromExecution(traceSpans, finalOutput, workflowInput)
```

*   **Extract Files**: Calls a private helper method (`this.extractFilesFromExecution`) to scan the `traceSpans`, `finalOutput`, and `workflowInput` for any references to files. This is important for auditing and later retrieval of artifacts produced or consumed by the workflow.

```typescript
    const [updatedLog] = await db
      .update(workflowExecutionLogs)
      .set({
        level,
        endedAt: new Date(endedAt),
        totalDurationMs,
        files: executionFiles.length > 0 ? executionFiles : null,
        executionData: {
          traceSpans,
          finalOutput,
          tokenBreakdown: {
            prompt: costSummary.totalPromptTokens,
            completion: costSummary.totalCompletionTokens,
            total: costSummary.totalTokens,
          },
          models: costSummary.models,
        },
        cost: {
          total: costSummary.totalCost,
          input: costSummary.totalInputCost,
          output: costSummary.totalOutputCost,
          tokens: {
            prompt: costSummary.totalPromptTokens,
            completion: costSummary.totalCompletionTokens,
            total: costSummary.totalTokens,
          },
          models: costSummary.models,
        },
      })
      .where(eq(workflowExecutionLogs.executionId, executionId))
      .returning()
```

*   **Update Log Record**:
    *   `await db.update(workflowExecutionLogs)`: Initiates an update operation on the `workflowExecutionLogs` table.
    *   `.set({...})`: Specifies the fields to be updated:
        *   `level`: The determined log level (`'info'` or `'error'`).
        *   `endedAt`: Converts the `endedAt` string into a `Date` object for storage.
        *   `totalDurationMs`: The total execution duration.
        *   `files`: Stores the extracted `executionFiles` array, or `null` if no files were found.
        *   `executionData`: An object containing structured data:
            *   `traceSpans`, `finalOutput`: The raw data from the parameters.
            *   `tokenBreakdown`, `models`: Detailed token and model cost information from the `costSummary`.
        *   `cost`: Another structured object specifically for cost details, mirroring parts of `costSummary`. This separation might be for easier querying or specific data structures in the database.
    *   `.where(eq(workflowExecutionLogs.executionId, executionId))`: Crucially, this specifies that only the log record matching the current `executionId` should be updated. `eq` is a Drizzle function for equality comparison.
    *   `.returning()`: Returns the updated row. `[updatedLog]` destructures the first (and only) element.

```typescript
    if (!updatedLog) {
      throw new Error(`Workflow log not found for execution ${executionId}`)
    }
```

*   **Error Handling**: If `updatedLog` is null (meaning no record was found or updated), it throws an error. This should ideally not happen if `startWorkflowExecution` successfully created the log.

---

### Billing and Usage Logic (The Complex Part)

This block handles updating user/organization billing statistics and sending usage threshold notifications.

```typescript
    try {
      const [wf] = await db.select().from(workflow).where(eq(workflow.id, updatedLog.workflowId))
      if (wf) {
        const [usr] = await db
          .select({ id: userTable.id, email: userTable.email, name: userTable.name })
          .from(userTable)
          .where(eq(userTable.id, wf.userId))
          .limit(1)
```

*   **Fetch Workflow and User**:
    *   It first retrieves the `workflow` record using `updatedLog.workflowId`.
    *   If the workflow is found (`if (wf)`), it then fetches the associated `user` record (`usr`) using `wf.userId`, selecting only the `id`, `email`, and `name`. This information is crucial for billing and notifications.

```typescript
        if (usr?.email) {
          const sub = await getHighestPrioritySubscription(usr.id)
          const costDelta = costSummary.totalCost
          const planName = sub?.plan || 'Free'
          const scope: 'user' | 'organization' =
            sub && (sub.plan === 'team' || sub.plan === 'enterprise') ? 'organization' : 'user'
```

*   **Check User Email & Subscription**:
    *   `if (usr?.email)`: Proceeds only if the user has an email address (necessary for notifications).
    *   `getHighestPrioritySubscription(usr.id)`: Retrieves the user's active subscription.
    *   `costDelta`: Stores the total cost of the current workflow execution.
    *   `planName`: Determines the user's plan (e.g., 'Free', 'team', 'enterprise').
    *   `scope`: Defines whether billing should be tracked at the `user` level or `organization` level, based on the subscription plan.

#### User-Level Billing (`scope === 'user'`)

```typescript
          if (scope === 'user') {
            const before = await checkUsageStatus(usr.id)

            await this.updateUserStats(
              updatedLog.workflowId,
              costSummary,
              updatedLog.trigger as ExecutionTrigger['type']
            )

            const limit = before.usageData.limit
            const percentBefore = before.usageData.percentUsed
            const percentAfter =
              limit > 0 ? Math.min(100, percentBefore + (costDelta / limit) * 100) : percentBefore
            const currentUsageAfter = before.usageData.currentUsage + costDelta

            await maybeSendUsageThresholdEmail({
              scope: 'user',
              userId: usr.id,
              userEmail: usr.email,
              userName: usr.name || undefined,
              planName,
              percentBefore,
              percentAfter,
              currentUsageAfter,
              limit,
            })
          }
```

*   **`checkUsageStatus(usr.id)`**: Fetches the user's current usage data and limits *before* applying the current execution's cost.
*   **`this.updateUserStats(...)`**: Calls a private method to update the user's `userStats` database record with the `costSummary` and trigger type. This is where the actual cost is applied to the user's balance.
*   **Calculate Usage Percentages**:
    *   `limit`: The user's usage limit.
    *   `percentBefore`: Usage percentage before this execution.
    *   `percentAfter`: Calculates the new usage percentage after adding `costDelta`. It includes a check for `limit > 0` to prevent division by zero and `Math.min(100, ...)` to cap the percentage at 100.
    *   `currentUsageAfter`: The total current usage after this execution.
*   **`maybeSendUsageThresholdEmail(...)`**: Calls the function to potentially send a usage notification email to the user if a configured threshold has been crossed (e.g., 50%, 80%, 100% of their limit). It passes all necessary user and usage details.

#### Organization-Level Billing (`else if (sub?.referenceId)`)

```typescript
          else if (sub?.referenceId) {
            let orgLimit = 0
            const orgRows = await db
              .select({ orgUsageLimit: organization.orgUsageLimit })
              .from(organization)
              .where(eq(organization.id, sub.referenceId))
              .limit(1)
            const { getPlanPricing } = await import('@/lib/billing/core/billing')
            const { basePrice } = getPlanPricing(sub.plan)
            const minimum = (sub.seats || 1) * basePrice
            if (orgRows.length > 0 && orgRows[0].orgUsageLimit) {
              const configured = Number.parseFloat(orgRows[0].orgUsageLimit)
              orgLimit = Math.max(configured, minimum)
            } else {
              orgLimit = minimum
            }

            const [{ sum: orgUsageBefore }] = await db
              .select({ sum: sql`COALESCE(SUM(${userStats.currentPeriodCost}), 0)` })
              .from(member)
              .leftJoin(userStats, eq(member.userId, userStats.userId))
              .where(eq(member.organizationId, sub.referenceId))
              .limit(1)
            const orgUsageBeforeNum = Number.parseFloat(
              (orgUsageBefore as any)?.toString?.() || '0'
            )

            await this.updateUserStats(
              updatedLog.workflowId,
              costSummary,
              updatedLog.trigger as ExecutionTrigger['type']
            )

            const percentBefore =
              orgLimit > 0 ? Math.min(100, (orgUsageBeforeNum / orgLimit) * 100) : 0
            const percentAfter =
              orgLimit > 0
                ? Math.min(100, percentBefore + (costDelta / orgLimit) * 100)
                : percentBefore
            const currentUsageAfter = orgUsageBeforeNum + costDelta

            await maybeSendUsageThresholdEmail({
              scope: 'organization',
              organizationId: sub.referenceId,
              planName,
              percentBefore,
              percentAfter,
              currentUsageAfter,
              limit: orgLimit,
            })
          }
```

*   **`else if (sub?.referenceId)`**: This block executes if the user is part of an organization (indicated by `sub.referenceId`, which likely refers to an `organization.id`).
*   **Determine Organization Limit**:
    *   `orgRows`: Fetches the `orgUsageLimit` from the `organization` table.
    *   `import('@/lib/billing/core/billing')`: Dynamically imports the billing utility.
    *   `getPlanPricing(sub.plan)`, `basePrice`: Gets pricing details for the organization's plan.
    *   `minimum = (sub.seats || 1) * basePrice`: Calculates a minimum usage limit based on the number of seats and the base price per seat.
    *   `orgLimit = Math.max(configured, minimum)`: The final organization limit is the greater of the configured limit (if any) or the calculated minimum.
*   **Calculate Organization Usage Before**:
    *   `db.select({ sum: sql`COALESCE(SUM(${userStats.currentPeriodCost}), 0)` })`: This is a Drizzle query to calculate the *total current period cost for all users within the organization*.
        *   `COALESCE(SUM(...), 0)`: Ensures that if there's no usage data, the sum defaults to 0.
        *   `from(member).leftJoin(userStats, eq(member.userId, userStats.userId))`: Joins the `member` table (to link users to organizations) with the `userStats` table (to get each user's cost).
        *   `where(eq(member.organizationId, sub.referenceId))`: Filters to only include members of the current organization.
    *   `orgUsageBeforeNum`: Parses the result into a number.
*   **`this.updateUserStats(...)`**: **Important**: Even for organization-scoped billing, the *individual user's* stats (`userStats` record) are updated first. This ensures per-user tracking.
*   **Calculate Organization Usage Percentages**: Similar to user-level calculations, but using `orgUsageBeforeNum` and `orgLimit`.
*   **`maybeSendUsageThresholdEmail(...)`**: Sends a usage notification email, but scoped to the `organization`, including the `organizationId`.

```typescript
        } else {
          await this.updateUserStats(
            updatedLog.workflowId,
            costSummary,
            updatedLog.trigger as ExecutionTrigger['type']
          )
        }
      } else {
        await this.updateUserStats(
          updatedLog.workflowId,
          costSummary,
          updatedLog.trigger as ExecutionTrigger['type']
        )
      }
    } catch (e) {
      try {
        await this.updateUserStats(
          updatedLog.workflowId,
          costSummary,
          updatedLog.trigger as ExecutionTrigger['type']
        )
      } catch {}
      logger.warn('Usage threshold notification check failed (non-fatal)', { error: e })
    }
```

*   **Fallback `updateUserStats`**:
    *   The `else` blocks (`if (usr?.email)` and `if (wf)`) ensure that `this.updateUserStats` is called *even if* no user email is found or if the workflow record isn't found. This prioritizes recording the cost against the user's stats, even if notification logic can't proceed.
*   **`catch (e)` block**:
    *   Handles any errors that occur during the complex billing/notification process.
    *   **Crucially**: It attempts to call `this.updateUserStats` *again* within the `catch` block. This emphasizes that updating the user's raw usage stats is paramount, even if the subsequent notification or plan-specific logic fails. The nested `try {} catch {}` around this retry prevents that internal retry from causing a further error in the outer `catch`.
    *   `logger.warn(...)`: Logs a warning, indicating that the failure is "non-fatal" – meaning the core workflow execution log update should still proceed, but there was an issue with usage notifications.

---

```typescript
    logger.debug(`Completed workflow execution ${executionId}`)
```

*   **Logging**: Records a debug message indicating the successful completion of the workflow execution processing.

```typescript
    const completedLog: WorkflowExecutionLog = {
      id: updatedLog.id,
      workflowId: updatedLog.workflowId,
      executionId: updatedLog.executionId,
      stateSnapshotId: updatedLog.stateSnapshotId,
      level: updatedLog.level as 'info' | 'error',
      trigger: updatedLog.trigger as ExecutionTrigger['type'],
      startedAt: updatedLog.startedAt.toISOString(),
      endedAt: updatedLog.endedAt?.toISOString() || endedAt,
      totalDurationMs: updatedLog.totalDurationMs || totalDurationMs,
      executionData: updatedLog.executionData as WorkflowExecutionLog['executionData'],
      cost: updatedLog.cost as any,
      createdAt: updatedLog.createdAt.toISOString(),
    }
```

*   **Return Formatted Log**: Creates a final `WorkflowExecutionLog` object to be returned, similar to `startWorkflowExecution`. It explicitly maps all properties from `updatedLog`, applies type assertions, and converts `Date` objects to ISO strings. It uses `endedAt` from parameters as a fallback if the database `endedAt` is null.

```typescript
    emitWorkflowExecutionCompleted(completedLog).catch((error) => {
      logger.error('Failed to emit workflow execution completed event', {
        error,
        executionId,
      })
    })
```

*   **Emit Event**: Calls `emitWorkflowExecutionCompleted` with the `completedLog` object. This allows other parts of the application (e.g., UI updates, analytics, external integrations) to react to the workflow's completion.
*   `.catch((error) => ...)`: Asynchronously emitting events can sometimes fail. This ensures that any errors during event emission are logged but do not prevent the method from returning the `completedLog`.

```typescript
    return completedLog
  }
```

*   Returns the fully processed and formatted `completedLog`.

#### `getWorkflowExecution` Method

This method allows retrieving a specific workflow execution log record by its `executionId`.

```typescript
  async getWorkflowExecution(executionId: string): Promise<WorkflowExecutionLog | null> {
    const [workflowLog] = await db
      .select()
      .from(workflowExecutionLogs)
      .where(eq(workflowExecutionLogs.executionId, executionId))
      .limit(1)
```

*   **`async getWorkflowExecution(executionId: string)`**: An asynchronous method that takes the `executionId` as a string.
*   **Return Type**: Promises to return a `WorkflowExecutionLog` object if found, or `null` otherwise.
*   **Database Query**:
    *   `db.select().from(workflowExecutionLogs)`: Selects all columns from the `workflowExecutionLogs` table.
    *   `where(eq(workflowExecutionLogs.executionId, executionId))`: Filters the results to find the log entry with the matching `executionId`.
    *   `.limit(1)`: Ensures only one record is returned (as `executionId` should be unique).
    *   `[workflowLog]`: Destructures the first element from the array of results.

```typescript
    if (!workflowLog) return null
```

*   **Not Found**: If `workflowLog` is null (no record was found), the method returns `null`.

```typescript
    return {
      id: workflowLog.id,
      workflowId: workflowLog.workflowId,
      executionId: workflowLog.executionId,
      stateSnapshotId: workflowLog.stateSnapshotId,
      level: workflowLog.level as 'info' | 'error',
      trigger: workflowLog.trigger as ExecutionTrigger['type'],
      startedAt: workflowLog.startedAt.toISOString(),
      endedAt: workflowLog.endedAt?.toISOString() || workflowLog.startedAt.toISOString(),
      totalDurationMs: workflowLog.totalDurationMs || 0,
      executionData: workflowLog.executionData as WorkflowExecutionLog['executionData'],
      cost: workflowLog.cost as any,
      createdAt: workflowLog.createdAt.toISOString(),
    }
  }
```

*   **Return Formatted Log**: If a record is found, it's mapped to the `WorkflowExecutionLog` interface, with date objects converted to ISO strings and default values provided for `endedAt` and `totalDurationMs` if they are null. This ensures consistent data format when retrieving logs.

#### `private updateUserStats` Method

This private helper method encapsulates the logic for updating a user's cumulative statistics, particularly their total tokens used and costs. It's marked `private` because it's an internal utility for `ExecutionLogger`.

```typescript
  private async updateUserStats(
    workflowId: string,
    costSummary: { /* ... cost details ... */ },
    trigger: ExecutionTrigger['type']
  ): Promise<void> {
```

*   **`private async updateUserStats(...)`**: An asynchronous, private method.
*   **Parameters**:
    *   `workflowId`: The ID of the workflow that generated the cost.
    *   `costSummary`: The detailed cost information for the execution.
    *   `trigger`: The type of event that triggered the workflow execution.
*   **Return Type**: `Promise<void>` indicates it doesn't return a value, only performs an operation.

```typescript
    if (!isBillingEnabled) {
      logger.debug('Billing is disabled, skipping user stats cost update')
      return
    }
```

*   **Billing Enabled Check**: If the global `isBillingEnabled` flag is false, the method logs a debug message and exits early, skipping any billing-related updates.

```typescript
    if (costSummary.totalCost <= 0) {
      logger.debug('No cost to update in user stats')
      return
    }
```

*   **No Cost Check**: If the total cost for the execution is zero or negative, there's no need to update user stats, so it logs and exits.

```typescript
    try {
      // Get the workflow record to get the userId
      const [workflowRecord] = await db
        .select()
        .from(workflow)
        .where(eq(workflow.id, workflowId))
        .limit(1)

      if (!workflowRecord) {
        logger.error(`Workflow ${workflowId} not found for user stats update`)
        return
      }

      const userId = workflowRecord.userId
      const costToStore = costSummary.totalCost
```

*   **Fetch Workflow and User ID**:
    *   Retrieves the `workflow` record to extract the `userId` associated with it. This links the cost back to the individual user.
    *   If the `workflowRecord` isn't found, it logs an error and returns, as the `userId` cannot be determined.
    *   `costToStore`: Stores the `totalCost` for clarity.

```typescript
      const existing = await db.select().from(userStats).where(eq(userStats.userId, userId))
      if (existing.length === 0) {
        logger.error('User stats record not found - should be created during onboarding', {
          userId,
          trigger,
        })
        return
      }
```

*   **Check for Existing `userStats`**: Verifies that a `userStats` record already exists for the `userId`. Such a record is expected to be created during user onboarding. If not found, it logs an error and returns, as there's no record to update.

```typescript
      const updateFields: any = {
        totalTokensUsed: sql`total_tokens_used + ${costSummary.totalTokens}`,
        totalCost: sql`total_cost + ${costToStore}`,
        currentPeriodCost: sql`current_period_cost + ${costToStore}`,
        lastActive: new Date(),
      }
```

*   **Define Update Fields**:
    *   `updateFields`: An object that will contain the database fields to be updated.
    *   `sql` tagged template literal: This is a powerful Drizzle ORM feature. It allows defining an update operation that *increments* a column's value atomically using raw SQL, rather than fetching, adding, and then updating (which can lead to race conditions).
        *   `totalTokensUsed`: Incremented by the current execution's total tokens.
        *   `totalCost`: Incremented by the current execution's total cost (lifetime cost).
        *   `currentPeriodCost`: Incremented by the current execution's total cost (cost for the current billing period).
        *   `lastActive`: Updated to the current date, indicating recent user activity.

```typescript
      switch (trigger) {
        case 'manual':
          updateFields.totalManualExecutions = sql`total_manual_executions + 1`
          break
        case 'api':
          updateFields.totalApiCalls = sql`total_api_calls + 1`
          break
        case 'webhook':
          updateFields.totalWebhookTriggers = sql`total_webhook_triggers + 1`
          break
        case 'schedule':
          updateFields.totalScheduledExecutions = sql`total_scheduled_executions + 1`
          break
        case 'chat':
          updateFields.totalChatExecutions = sql`total_chat_executions + 1`
          break
      }
```

*   **Increment Trigger-Specific Counters**: A `switch` statement dynamically adds specific counter increments to `updateFields` based on the `trigger.type`. This allows tracking how many times a user's workflows are executed via different methods (manual, API, webhook, etc.). These also use Drizzle's `sql` for atomic increments.

```typescript
      await db.update(userStats).set(updateFields).where(eq(userStats.userId, userId))
```

*   **Perform Update**: Executes the Drizzle update query on the `userStats` table, applying all the defined `updateFields` for the specific `userId`.

```typescript
      logger.debug('Updated user stats record with cost data', {
        userId,
        trigger,
        addedCost: costToStore,
        addedTokens: costSummary.totalTokens,
      })
```

*   **Logging**: Records a debug message confirming the update, including relevant details.

```typescript
      // Check if user has hit overage threshold and bill incrementally
      await checkAndBillOverageThreshold(userId)
    } catch (error) {
      logger.error('Error updating user stats with cost information', {
        workflowId,
        error,
        costSummary,
      })
      // Don't throw - we want execution to continue even if user stats update fails
    }
  }
```

*   **`checkAndBillOverageThreshold(userId)`**: After updating the stats, this function is called to immediately check if the user has crossed any overage thresholds and trigger incremental billing if applicable.
*   **`catch (error)` block**: If any error occurs during the `updateUserStats` process, it's caught and logged as an error. The comment `// Don't throw` indicates that this operation is considered non-critical enough to halt the entire workflow execution process if it fails. The system prefers to complete the workflow and log the execution, even if user stats updates are temporarily impaired.

#### `private extractFilesFromExecution` Method

This private helper method recursively scans various parts of the workflow execution data to find and extract file references.

```typescript
  private extractFilesFromExecution(
    traceSpans?: any[],
    finalOutput?: any,
    workflowInput?: any
  ): any[] {
```

*   **`private extractFilesFromExecution(...)`**: An asynchronous, private method.
*   **Parameters**:
    *   `traceSpans` (optional): The array of `TraceSpan`s.
    *   `finalOutput` (optional): The final output of the workflow.
    *   `workflowInput` (optional): The initial input to the workflow.
*   **Return Type**: `any[]` (an array of any type), expected to be an array of file objects.

```typescript
    const files: any[] = []
    const seenFileIds = new Set<string>()
```

*   **Initialization**:
    *   `files`: An array to store the extracted file objects.
    *   `seenFileIds`: A `Set` used to keep track of file IDs that have already been added to the `files` array. This prevents duplicate file entries if a file is referenced multiple times.

```typescript
    // Helper function to extract files from any object
    const extractFilesFromObject = (obj: any, source: string) => {
      if (!obj || typeof obj !== 'object') return
```

*   **`extractFilesFromObject` Helper**:
    *   This nested function is the core of the extraction logic. It's designed to be recursive, traversing arbitrary JavaScript objects and arrays.
    *   `if (!obj || typeof obj !== 'object') return`: Base case for the recursion: if the current `obj` is null, not an object, or not an array, it cannot contain file references, so the function returns.

```typescript
      // Check if this object has files property
      if (Array.isArray(obj.files)) {
        for (const file of obj.files) {
          if (file?.name && file.key && file.id) {
            if (!seenFileIds.has(file.id)) {
              seenFileIds.add(file.id)
              files.push({ /* ... file properties ... */ })
            }
          }
        }
      }
```

*   **Check `obj.files` Array**: Looks for a property named `files` that is an array. If found, it iterates through each `file` in the array.
*   **File Validation**: `if (file?.name && file.key && file.id)` checks if the object looks like a valid file reference (having `name`, `key`, and `id`).
*   **Deduplication & Add**: If the `file.id` hasn't been seen before, it's added to `seenFileIds` and the file's relevant properties are pushed to the `files` array.

```typescript
      // Check if this object has attachments property (for Gmail and other tools)
      if (Array.isArray(obj.attachments)) {
        for (const file of obj.attachments) {
          if (file?.name && file.key && file.id) {
            if (!seenFileIds.has(file.id)) {
              seenFileIds.add(file.id)
              files.push({ /* ... file properties ... */ })
            }
          }
        }
      }
```

*   **Check `obj.attachments` Array**: Similar logic to `obj.files`, but specifically for an `attachments` property (common in integrations like Gmail).

```typescript
      // Check if this object itself is a file reference
      if (obj.name && obj.key && typeof obj.size === 'number') {
        if (!obj.id) {
          logger.warn(`File object missing ID, skipping: ${obj.name}`)
          return
        }

        if (!seenFileIds.has(obj.id)) {
          seenFileIds.add(obj.id)
          files.push({ /* ... file properties ... */ })
        }
      }
```

*   **Check if `obj` *is* a File**: This checks if the current object *itself* directly represents a file (e.g., `name`, `key`, `size` properties).
*   **ID Check**: It specifically warns if a file object is found without an `id`, indicating a potential data integrity issue.
*   **Deduplication & Add**: If unique, the file is added to `files`.

```typescript
      // Recursively check nested objects and arrays
      if (Array.isArray(obj)) {
        obj.forEach((item, index) => extractFilesFromObject(item, `${source}[${index}]`))
      } else if (typeof obj === 'object') {
        Object.entries(obj).forEach(([key, value]) => {
          extractFilesFromObject(value, `${source}.${key}`)
        })
      }
    }
```

*   **Recursive Traversal**: This is where the magic happens for deep inspection:
    *   `if (Array.isArray(obj))`: If `obj` is an array, it iterates over each item and recursively calls `extractFilesFromObject` on that item.
    *   `else if (typeof obj === 'object')`: If `obj` is a plain object, it iterates over its key-value pairs (`Object.entries`) and recursively calls `extractFilesFromObject` on each `value`.
    *   This ensures that even files deeply nested within complex `traceSpans`, `finalOutput`, or `workflowInput` are found.

```typescript
    // Extract files from trace spans
    if (traceSpans && Array.isArray(traceSpans)) {
      traceSpans.forEach((span, index) => {
        extractFilesFromObject(span, `trace_span_${index}`)
      })
    }

    // Extract files from final output
    if (finalOutput) {
      extractFilesFromObject(finalOutput, 'final_output')
    }

    // Extract files from workflow input
    if (workflowInput) {
      extractFilesFromObject(workflowInput, 'workflow_input')
    }
```

*   **Initiate Extraction**: These blocks make the initial calls to `extractFilesFromObject`, passing in the top-level `traceSpans`, `finalOutput`, and `workflowInput` to start the recursive scanning process.

```typescript
    logger.debug(`Extracted ${files.length} file(s) from execution`, {
      fileNames: files.map((f) => f.name),
    })

    return files
  }
}
```

*   **Logging & Return**: Logs how many files were found and returns the `files` array.

### Exporting the Logger Instance

```typescript
export const executionLogger = new ExecutionLogger()
```

*   This line creates a singleton instance of the `ExecutionLogger` class and exports it as `executionLogger`. This means that any other file importing `executionLogger` will receive the *same* instance, ensuring consistent state and avoiding multiple database connections or logger instances.

---

### Summary and Key Takeaways

1.  **Centralized Logging**: The `ExecutionLogger` is the single point of truth for logging workflow executions, ensuring consistency and rich data capture.
2.  **Database Persistence**: It uses Drizzle ORM to interact with a PostgreSQL (or similar) database, storing execution details, state snapshots, costs, and file references.
3.  **Lifecycle Management**: It handles both the start (`startWorkflowExecution`) and completion (`completeWorkflowExecution`) of a workflow, populating different parts of the log record at appropriate times.
4.  **Error Detection**: It can infer the success or failure of a workflow by recursively inspecting `traceSpans`.
5.  **Comprehensive Billing Integration**: A major part of `completeWorkflowExecution` is dedicated to:
    *   Calculating and applying execution costs to user/organization stats.
    *   Determining usage limits based on subscription plans.
    *   Sending proactive usage threshold notification emails.
    *   Triggering overage billing.
    *   It's designed with resilience, attempting to update core stats even if notification processes fail.
6.  **Data Enrichment**: It extracts valuable artifacts like files from various parts of the execution data, making them queryable later.
7.  **Atomic Database Updates**: Leverages Drizzle's `sql` utility to perform safe, atomic increments on counters in the `userStats` table, preventing race conditions.
8.  **Event-Driven Architecture**: Emits a `workflowExecutionCompleted` event, allowing other services to react in a decoupled manner.
9.  **Robustness**: Includes `try...catch` blocks and checks for `isBillingEnabled` to ensure the logging process is resilient and adapts to environment configurations.