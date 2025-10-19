```typescript
import type { Edge } from 'reactflow'
import type { BlockLog, NormalizedBlockOutput } from '@/executor/types'
import type { DeploymentStatus } from '@/stores/workflows/registry/types'
import type { Loop, Parallel, WorkflowState } from '@/stores/workflows/workflow/types'

export type { WorkflowState, Loop, Parallel, DeploymentStatus }
export type WorkflowEdge = Edge
export type { NormalizedBlockOutput, BlockLog }

export interface PricingInfo {
  input: number
  output: number
  cachedInput?: number
  updatedAt: string
}

export interface TokenUsage {
  prompt: number
  completion: number
  total: number
}

export interface CostBreakdown {
  input: number
  output: number
  total: number
  tokens: TokenUsage
  model: string
  pricing: PricingInfo
}

export interface ToolCall {
  name: string
  duration: number
  startTime: string
  endTime: string
  status: 'success' | 'error'
  input?: Record<string, unknown>
  output?: Record<string, unknown>
  error?: string
}

export type BlockInputData = Record<string, any>
export type BlockOutputData = NormalizedBlockOutput | null

export interface ExecutionEnvironment {
  variables: Record<string, string>
  workflowId: string
  executionId: string
  userId: string
  workspaceId: string
}

export interface ExecutionTrigger {
  type: 'api' | 'webhook' | 'schedule' | 'manual' | 'chat'
  source: string
  data?: Record<string, unknown>
  timestamp: string
}

export interface ExecutionStatus {
  status: 'running' | 'completed' | 'failed' | 'cancelled'
  startedAt: string
  endedAt?: string
  durationMs?: number
}

export interface WorkflowExecutionSnapshot {
  id: string
  workflowId: string
  stateHash: string
  stateData: WorkflowState
  createdAt: string
}

export type WorkflowExecutionSnapshotInsert = Omit<WorkflowExecutionSnapshot, 'createdAt'>
export type WorkflowExecutionSnapshotSelect = WorkflowExecutionSnapshot

export interface WorkflowExecutionLog {
  id: string
  workflowId: string
  executionId: string
  stateSnapshotId: string
  level: 'info' | 'error'
  trigger: ExecutionTrigger['type']
  startedAt: string
  endedAt: string
  totalDurationMs: number
  files?: Array<{
    id: string
    name: string
    size: number
    type: string
    url: string
    key: string
    uploadedAt: string
    expiresAt: string
    storageProvider?: 's3' | 'blob' | 'local'
    bucketName?: string
  }>
  // Execution details
  executionData: {
    environment?: ExecutionEnvironment
    trigger?: ExecutionTrigger
    traceSpans?: TraceSpan[]
    errorDetails?: {
      blockId: string
      blockName: string
      error: string
      stackTrace?: string
    }
  }
  // Top-level cost information
  cost?: {
    input?: number
    output?: number
    total?: number
    tokens?: { prompt?: number; completion?: number; total?: number }
    models?: Record<
      string,
      {
        input?: number
        output?: number
        total?: number
        tokens?: { prompt?: number; completion?: number; total?: number }
      }
    >
  }
  duration?: string
  createdAt: string
}

export type WorkflowExecutionLogInsert = Omit<WorkflowExecutionLog, 'id' | 'createdAt'>
export type WorkflowExecutionLogSelect = WorkflowExecutionLog

export interface TokenInfo {
  input?: number
  output?: number
  total?: number
  prompt?: number
  completion?: number
}

export interface ProviderTiming {
  duration: number
  startTime: string
  endTime: string
  segments: Array<{
    type: string
    name?: string
    startTime: string | number
    endTime: string | number
    duration: number
  }>
}

export interface TraceSpan {
  id: string
  name: string
  type: string
  duration: number
  startTime: string
  endTime: string
  children?: TraceSpan[]
  toolCalls?: ToolCall[]
  status?: 'success' | 'error'
  tokens?: number | TokenInfo
  relativeStartMs?: number
  blockId?: string
  input?: Record<string, unknown>
  output?: Record<string, unknown>
  model?: string
  cost?: {
    input?: number
    output?: number
    total?: number
  }
  providerTiming?: ProviderTiming
}

export interface WorkflowExecutionSummary {
  id: string
  workflowId: string
  workflowName: string
  executionId: string
  trigger: ExecutionTrigger['type']
  status: ExecutionStatus['status']
  startedAt: string
  endedAt: string
  durationMs: number

  costSummary: {
    total: number
    inputCost: number
    outputCost: number
    tokens: number
  }
  stateSnapshotId: string
  errorSummary?: {
    blockId: string
    blockName: string
    message: string
  }
}

export interface WorkflowExecutionDetail extends WorkflowExecutionSummary {
  environment: ExecutionEnvironment
  triggerData: ExecutionTrigger
  blockExecutions: BlockExecutionSummary[]
  traceSpans: TraceSpan[]
  workflowState: WorkflowState
}

export interface BlockExecutionSummary {
  id: string
  blockId: string
  blockName: string
  blockType: string
  startedAt: string
  endedAt: string
  durationMs: number
  status: 'success' | 'error' | 'skipped'
  errorMessage?: string
  cost?: CostBreakdown
  inputSummary: {
    parameterCount: number
    hasComplexData: boolean
  }
  outputSummary: {
    hasOutput: boolean
    outputType: string
    hasError: boolean
  }
}

export interface PaginatedResponse<T> {
  data: T[]
  pagination: {
    page: number
    pageSize: number
    total: number
    totalPages: number
    hasNext: boolean
    hasPrevious: boolean
  }
}

export type WorkflowExecutionsResponse = PaginatedResponse<WorkflowExecutionSummary>
export type BlockExecutionsResponse = PaginatedResponse<BlockExecutionSummary>

export interface WorkflowExecutionFilters {
  workflowIds?: string[]
  folderIds?: string[]
  triggers?: ExecutionTrigger['type'][]
  status?: ExecutionStatus['status'][]
  startDate?: string
  endDate?: string
  search?: string
  minDuration?: number
  maxDuration?: number
  minCost?: number
  maxCost?: number
  hasErrors?: boolean
}

export interface PaginationParams {
  page: number
  pageSize: number
  sortBy?: 'startedAt' | 'durationMs' | 'totalCost' | 'blockCount'
  sortOrder?: 'asc' | 'desc'
}

export interface LogsQueryParams extends WorkflowExecutionFilters, PaginationParams {
  includeBlockSummary?: boolean
  includeWorkflowState?: boolean
}

export interface LogsError {
  code: 'EXECUTION_NOT_FOUND' | 'SNAPSHOT_NOT_FOUND' | 'INVALID_WORKFLOW_STATE' | 'STORAGE_ERROR'
  message: string
  details?: Record<string, unknown>
}

export interface ValidationError {
  field: string
  message: string
  value: unknown
}

export class LogsServiceError extends Error {
  public code: LogsError['code']
  public details?: Record<string, unknown>

  constructor(message: string, code: LogsError['code'], details?: Record<string, unknown>) {
    super(message)
    this.name = 'LogsServiceError'
    this.code = code
    this.details = details
  }
}

export interface DatabaseOperationResult<T> {
  success: boolean
  data?: T
  error?: LogsServiceError
}

export interface BatchInsertResult<T> {
  inserted: T[]
  failed: Array<{
    item: T
    error: string
  }>
  totalAttempted: number
  totalSucceeded: number
  totalFailed: number
}

export interface SnapshotService {
  createSnapshot(workflowId: string, state: WorkflowState): Promise<WorkflowExecutionSnapshot>
  getSnapshot(id: string): Promise<WorkflowExecutionSnapshot | null>
  getSnapshotByHash(workflowId: string, hash: string): Promise<WorkflowExecutionSnapshot | null>
  computeStateHash(state: WorkflowState): string
  cleanupOrphanedSnapshots(olderThanDays: number): Promise<number>
}

export interface SnapshotCreationResult {
  snapshot: WorkflowExecutionSnapshot
  isNew: boolean
}

export interface ExecutionLoggerService {
  startWorkflowExecution(params: {
    workflowId: string
    executionId: string
    trigger: ExecutionTrigger
    environment: ExecutionEnvironment
    workflowState: WorkflowState
  }): Promise<{
    workflowLog: WorkflowExecutionLog
    snapshot: WorkflowExecutionSnapshot
  }>

  completeWorkflowExecution(params: {
    executionId: string
    endedAt: string
    totalDurationMs: number

    costSummary: {
      totalCost: number
      totalInputCost: number
      totalOutputCost: number
      totalTokens: number
    }
    finalOutput: BlockOutputData
    traceSpans?: TraceSpan[]
  }): Promise<WorkflowExecutionLog>
}
```

### Purpose of this file

This TypeScript file defines a comprehensive set of interfaces, types, and a class related to workflow executions and their associated logs. It serves as a central source of truth for data structures used within a system that manages and tracks the execution of workflows, including details about triggers, environments, costs, errors, and snapshots. It focuses on structuring the data associated with logging and execution details for a workflow system. This allows clear definitions and sharing of these data structures across different parts of the application, promoting type safety and code maintainability.

### Simplification of Complex Logic

The file simplifies complex logic by:

1.  **Data Modeling**: Defining explicit interfaces for various aspects of workflow execution (e.g., `ExecutionEnvironment`, `ExecutionTrigger`, `CostBreakdown`). This provides a clear structure and allows developers to reason about these concepts more easily.
2.  **Type Aliases**: Using type aliases (e.g., `BlockInputData`, `WorkflowEdge`) to provide meaningful names for complex types, which enhances readability.
3.  **Specialized Interfaces**: Breaking down complex data into smaller, more manageable interfaces (e.g., `WorkflowExecutionDetail` extends `WorkflowExecutionSummary`, rather than having one giant interface).
4.  **Omit Types**: Leveraging `Omit` to create new types derived from existing ones, reducing redundancy and ensuring consistency (e.g. `WorkflowExecutionSnapshotInsert`).

### Explanation of each line of code

```typescript
import type { Edge } from 'reactflow'
import type { BlockLog, NormalizedBlockOutput } from '@/executor/types'
import type { DeploymentStatus } from '@/stores/workflows/registry/types'
import type { Loop, Parallel, WorkflowState } from '@/stores/workflows/workflow/types'
```

*   **Imports:** These lines import type definitions from other modules.  `reactflow` provides `Edge` for visual workflow representation.  `@/executor/types` contributes `BlockLog` and `NormalizedBlockOutput` for block execution details. `@/stores/workflows/registry/types` provides `DeploymentStatus`, likely related to workflow deployment.  `@/stores/workflows/workflow/types` gives `Loop`, `Parallel`, and `WorkflowState` representing workflow structure and state.  Using `type` in the import statement indicates that these are type-only imports, meaning they're not used as values but only for type checking.

```typescript
export type { WorkflowState, Loop, Parallel, DeploymentStatus }
export type WorkflowEdge = Edge
export type { NormalizedBlockOutput, BlockLog }
```

*   **Re-exports:** These lines re-export the imported types, making them available for use in other modules that import this file.  This is a convenience for developers, as they can import all necessary workflow-related types from a single location. `WorkflowEdge` is defined as type alias of `Edge` from `reactflow`.

```typescript
export interface PricingInfo {
  input: number
  output: number
  cachedInput?: number
  updatedAt: string
}
```

*   **`PricingInfo` Interface:** Defines the structure for pricing information, including `input` cost, `output` cost, optional `cachedInput` cost, and the timestamp (`updatedAt`) of when the pricing information was last updated.

```typescript
export interface TokenUsage {
  prompt: number
  completion: number
  total: number
}
```

*   **`TokenUsage` Interface:**  Specifies the structure for token usage data, including `prompt` tokens, `completion` tokens, and the `total` tokens used.  This is likely relevant in the context of large language models (LLMs).

```typescript
export interface CostBreakdown {
  input: number
  output: number
  total: number
  tokens: TokenUsage
  model: string
  pricing: PricingInfo
}
```

*   **`CostBreakdown` Interface:** Represents a detailed breakdown of costs, including `input` cost, `output` cost, `total` cost, `tokens` used (using the `TokenUsage` interface), the `model` used (e.g., a specific LLM), and detailed `pricing` information (using the `PricingInfo` interface).

```typescript
export interface ToolCall {
  name: string
  duration: number
  startTime: string
  endTime: string
  status: 'success' | 'error'
  input?: Record<string, unknown>
  output?: Record<string, unknown>
  error?: string
}
```

*   **`ToolCall` Interface:** Defines the structure for representing a call to an external tool or service within a workflow, including its `name`, `duration`, `startTime`, `endTime`, `status` (success or error), optional `input` and `output` data (as a generic record of key-value pairs where the value type is unknown), and an optional `error` message.

```typescript
export type BlockInputData = Record<string, any>
export type BlockOutputData = NormalizedBlockOutput | null
```

*   **`BlockInputData` Type Alias:** Defines the structure for data passed as input to a block. It's a `Record` (an object with string keys) where the value can be of any type (`any`).
*   **`BlockOutputData` Type Alias:** Defines the structure for data output by a block. It can be a `NormalizedBlockOutput` (imported from `@/executor/types`) or `null`.

```typescript
export interface ExecutionEnvironment {
  variables: Record<string, string>
  workflowId: string
  executionId: string
  userId: string
  workspaceId: string
}
```

*   **`ExecutionEnvironment` Interface:** Defines the execution environment for a workflow, including `variables` (as a record of string key-value pairs), `workflowId`, `executionId`, `userId`, and `workspaceId`.

```typescript
export interface ExecutionTrigger {
  type: 'api' | 'webhook' | 'schedule' | 'manual' | 'chat'
  source: string
  data?: Record<string, unknown>
  timestamp: string
}
```

*   **`ExecutionTrigger` Interface:** Defines how a workflow execution was triggered. It includes the `type` of trigger (e.g., 'api', 'webhook', 'schedule', 'manual', 'chat'), the `source` of the trigger, optional `data` associated with the trigger (as a generic record), and the `timestamp` of when the trigger occurred.

```typescript
export interface ExecutionStatus {
  status: 'running' | 'completed' | 'failed' | 'cancelled'
  startedAt: string
  endedAt?: string
  durationMs?: number
}
```

*   **`ExecutionStatus` Interface:**  Specifies the current status of a workflow execution, including `status` (running, completed, failed, or cancelled), `startedAt` timestamp, optional `endedAt` timestamp, and optional `durationMs` (duration in milliseconds).

```typescript
export interface WorkflowExecutionSnapshot {
  id: string
  workflowId: string
  stateHash: string
  stateData: WorkflowState
  createdAt: string
}
```

*   **`WorkflowExecutionSnapshot` Interface:** Represents a snapshot of the workflow's execution state at a specific point in time. It includes `id`, `workflowId`, `stateHash` (a hash of the workflow state), `stateData` (the actual `WorkflowState` object), and `createdAt` timestamp.

```typescript
export type WorkflowExecutionSnapshotInsert = Omit<WorkflowExecutionSnapshot, 'createdAt'>
export type WorkflowExecutionSnapshotSelect = WorkflowExecutionSnapshot
```

*   **`WorkflowExecutionSnapshotInsert` Type Alias:** Defines the structure for inserting a new workflow execution snapshot into a database. It uses the `Omit` utility type to exclude the `createdAt` field from the `WorkflowExecutionSnapshot` interface, as the database might automatically generate this value.
*   **`WorkflowExecutionSnapshotSelect` Type Alias:** Simply defines that the selection object for WorkflowExecutionSnapshot is the same as its base type.

```typescript
export interface WorkflowExecutionLog {
  id: string
  workflowId: string
  executionId: string
  stateSnapshotId: string
  level: 'info' | 'error'
  trigger: ExecutionTrigger['type']
  startedAt: string
  endedAt: string
  totalDurationMs: number
  files?: Array<{
    id: string
    name: string
    size: number
    type: string
    url: string
    key: string
    uploadedAt: string
    expiresAt: string
    storageProvider?: 's3' | 'blob' | 'local'
    bucketName?: string
  }>
  // Execution details
  executionData: {
    environment?: ExecutionEnvironment
    trigger?: ExecutionTrigger
    traceSpans?: TraceSpan[]
    errorDetails?: {
      blockId: string
      blockName: string
      error: string
      stackTrace?: string
    }
  }
  // Top-level cost information
  cost?: {
    input?: number
    output?: number
    total?: number
    tokens?: { prompt?: number; completion?: number; total?: number }
    models?: Record<
      string,
      {
        input?: number
        output?: number
        total?: number
        tokens?: { prompt?: number; completion?: number; total?: number }
      }
    >
  }
  duration?: string
  createdAt: string
}
```

*   **`WorkflowExecutionLog` Interface:** This is a critical interface representing the log of a workflow execution. It includes:
    *   `id`: Unique identifier for the log entry.
    *   `workflowId`: The ID of the workflow.
    *   `executionId`: The ID of the specific execution of the workflow.
    *   `stateSnapshotId`: ID of the snapshot of the workflow state at the time of logging.
    *   `level`: Log level ('info' or 'error').
    *   `trigger`: The type of trigger that initiated the execution (using `ExecutionTrigger['type']`).
    *   `startedAt`: Start timestamp.
    *   `endedAt`: End timestamp.
    *   `totalDurationMs`: Total duration in milliseconds.
    *   `files`: (Optional) Array of files associated with the execution.
    *   `executionData`: Contains nested information about the execution:
        *   `environment`:  The execution environment (using the `ExecutionEnvironment` interface).
        *   `trigger`: The trigger that initiated the execution (using the `ExecutionTrigger` interface).
        *   `traceSpans`: An array of `TraceSpan` objects, providing detailed tracing information.
        *   `errorDetails`: (Optional) Information about errors that occurred during execution, including the `blockId`, `blockName`, `error` message, and optional `stackTrace`.
    *   `cost`:  (Optional) Cost information for the execution:
        *   `input`: Input cost.
        *   `output`: Output cost.
        *   `total`: Total cost.
        *   `tokens`: Token usage information.
        *   `models`: Cost breakdown per model.
    *   `duration`: String representation of the duration.
    *   `createdAt`: Creation timestamp of the log entry.

```typescript
export type WorkflowExecutionLogInsert = Omit<WorkflowExecutionLog, 'id' | 'createdAt'>
export type WorkflowExecutionLogSelect = WorkflowExecutionLog
```

*   **`WorkflowExecutionLogInsert` Type Alias:** Defines the structure for inserting a new workflow execution log into a database. It uses `Omit` to exclude `id` and `createdAt` fields, assuming the database handles these.
*   **`WorkflowExecutionLogSelect` Type Alias:** Represents the structure of a log retrieved from the database.

```typescript
export interface TokenInfo {
  input?: number
  output?: number
  total?: number
  prompt?: number
  completion?: number
}
```

*   **`TokenInfo` Interface:** Represents more granular token information, breaking down token usage into optional `input`, `output`, `prompt`, and `completion` counts.

```typescript
export interface ProviderTiming {
  duration: number
  startTime: string
  endTime: string
  segments: Array<{
    type: string
    name?: string
    startTime: string | number
    endTime: string | number
    duration: number
  }>
}
```

*   **`ProviderTiming` Interface:** Defines timing information for a specific provider or service used within the workflow. It includes `duration`, `startTime`, `endTime`, and an array of `segments` that further break down the timing into different parts of the provider's execution.

```typescript
export interface TraceSpan {
  id: string
  name: string
  type: string
  duration: number
  startTime: string
  endTime: string
  children?: TraceSpan[]
  toolCalls?: ToolCall[]
  status?: 'success' | 'error'
  tokens?: number | TokenInfo
  relativeStartMs?: number
  blockId?: string
  input?: Record<string, unknown>
  output?: Record<string, unknown>
  model?: string
  cost?: {
    input?: number
    output?: number
    total?: number
  }
  providerTiming?: ProviderTiming
}
```

*   **`TraceSpan` Interface:**  Represents a single span in a distributed trace, providing detailed information about a specific operation within the workflow. Key properties include:
    *   `id`: Unique identifier for the span.
    *   `name`: Name of the operation.
    *   `type`: Type of operation.
    *   `duration`: Duration of the operation.
    *   `startTime`: Start timestamp.
    *   `endTime`: End timestamp.
    *   `children`:  (Optional) Array of child `TraceSpan` objects, representing nested operations.
    *   `toolCalls`: (Optional) Array of `ToolCall` objects representing calls to external tools.
    *   `status`: (Optional) Status of the operation ('success' or 'error').
    *   `tokens`: (Optional) Token usage information.
    *   `relativeStartMs`: (Optional) Start time relative to the parent span.
    *   `blockId`: (Optional) ID of the block associated with the span.
    *   `input`: (Optional) Input data.
    *   `output`: (Optional) Output data.
    *   `model`: (Optional) The model used.
    *   `cost`: (Optional) Cost information.
    *   `providerTiming`: (Optional) Provider timing information.

```typescript
export interface WorkflowExecutionSummary {
  id: string
  workflowId: string
  workflowName: string
  executionId: string
  trigger: ExecutionTrigger['type']
  status: ExecutionStatus['status']
  startedAt: string
  endedAt: string
  durationMs: number

  costSummary: {
    total: number
    inputCost: number
    outputCost: number
    tokens: number
  }
  stateSnapshotId: string
  errorSummary?: {
    blockId: string
    blockName: string
    message: string
  }
}
```

*   **`WorkflowExecutionSummary` Interface:** Provides a high-level summary of a workflow execution. It includes:
    *   `id`: Unique ID of the summary.
    *   `workflowId`: ID of the workflow.
    *   `workflowName`: Name of the workflow.
    *   `executionId`: ID of the execution.
    *   `trigger`: Trigger type.
    *   `status`: Execution status.
    *   `startedAt`: Start timestamp.
    *   `endedAt`: End timestamp.
    *   `durationMs`: Duration in milliseconds.
    *   `costSummary`: Summary of costs: `total`, `inputCost`, `outputCost`, and `tokens`.
    *   `stateSnapshotId`: ID of the state snapshot.
    *   `errorSummary`: (Optional) Summary of any errors that occurred.

```typescript
export interface WorkflowExecutionDetail extends WorkflowExecutionSummary {
  environment: ExecutionEnvironment
  triggerData: ExecutionTrigger
  blockExecutions: BlockExecutionSummary[]
  traceSpans: TraceSpan[]
  workflowState: WorkflowState
}
```

*   **`WorkflowExecutionDetail` Interface:** Extends the `WorkflowExecutionSummary` to provide more detailed information about a workflow execution. It includes:
    *   `environment`: The execution environment.
    *   `triggerData`: The trigger data.
    *   `blockExecutions`: An array of `BlockExecutionSummary` objects, providing information about each block's execution.
    *   `traceSpans`: An array of `TraceSpan` objects.
    *   `workflowState`: The final workflow state.

```typescript
export interface BlockExecutionSummary {
  id: string
  blockId: string
  blockName: string
  blockType: string
  startedAt: string
  endedAt: string
  durationMs: number
  status: 'success' | 'error' | 'skipped'
  errorMessage?: string
  cost?: CostBreakdown
  inputSummary: {
    parameterCount: number
    hasComplexData: boolean
  }
  outputSummary: {
    hasOutput: boolean
    outputType: string
    hasError: boolean
  }
}
```

*   **`BlockExecutionSummary` Interface:**  Provides a summary of a single block's execution within a workflow. Includes:
    *   `id`: Unique ID of the summary.
    *   `blockId`: ID of the block.
    *   `blockName`: Name of the block.
    *   `blockType`: Type of the block.
    *   `startedAt`: Start timestamp.
    *   `endedAt`: End timestamp.
    *   `durationMs`: Duration in milliseconds.
    *   `status`: Execution status ('success', 'error', or 'skipped').
    *   `errorMessage`: (Optional) Error message if the block failed.
    *   `cost`: (Optional) Cost breakdown for the block.
    *   `inputSummary`: Summary of the block's input.
    *   `outputSummary`: Summary of the block's output.

```typescript
export interface PaginatedResponse<T> {
  data: T[]
  pagination: {
    page: number
    pageSize: number
    total: number
    totalPages: number
    hasNext: boolean
    hasPrevious: boolean
  }
}
```

*   **`PaginatedResponse` Interface:** Defines the structure for a paginated response from an API, including an array of `data` (of type `T`) and a `pagination` object with information about the current page, page size, total number of items, total number of pages, and flags indicating whether there are next and previous pages.

```typescript
export type WorkflowExecutionsResponse = PaginatedResponse<WorkflowExecutionSummary>
export type BlockExecutionsResponse = PaginatedResponse<BlockExecutionSummary>
```

*   **`WorkflowExecutionsResponse` Type Alias:**  Defines the type for a paginated response containing an array of `WorkflowExecutionSummary` objects.
*   **`BlockExecutionsResponse` Type Alias:** Defines the type for a paginated response containing an array of `BlockExecutionSummary` objects.

```typescript
export interface WorkflowExecutionFilters {
  workflowIds?: string[]
  folderIds?: string[]
  triggers?: ExecutionTrigger['type'][]
  status?: ExecutionStatus['status'][]
  startDate?: string
  endDate?: string
  search?: string
  minDuration?: number
  maxDuration?: number
  minCost?: number
  maxCost?: number
  hasErrors?: boolean
}
```

*   **`WorkflowExecutionFilters` Interface:** Defines the structure for filtering workflow executions based on various criteria, such as `workflowIds`, `folderIds`, `triggers`, `status`, `startDate`, `endDate`, a `search` string, `minDuration`, `maxDuration`, `minCost`, `maxCost`, and a flag indicating whether to filter by executions that have errors (`hasErrors`).

```typescript
export interface PaginationParams {
  page: number
  pageSize: number
  sortBy?: 'startedAt' | 'durationMs' | 'totalCost' | 'blockCount'
  sortOrder?: 'asc' | 'desc'
}
```

*   **`PaginationParams` Interface:** Defines the structure for pagination parameters, including the current `page`, `pageSize`, optional `sortBy` field, and optional `sortOrder` ('asc' or 'desc').

```typescript
export interface LogsQueryParams extends WorkflowExecutionFilters, PaginationParams {
  includeBlockSummary?: boolean
  includeWorkflowState?: boolean
}
```

*   **`LogsQueryParams` Interface:** Extends both `WorkflowExecutionFilters` and `PaginationParams` to define the structure for query parameters used when fetching workflow execution logs. It also includes optional flags to `includeBlockSummary` and `includeWorkflowState` in the response.

```typescript
export interface LogsError {
  code: 'EXECUTION_NOT_FOUND' | 'SNAPSHOT_NOT_FOUND' | 'INVALID_WORKFLOW_STATE' | 'STORAGE_ERROR'
  message: string
  details?: Record<string, unknown>
}
```

*   **`LogsError` Interface:** Defines the structure for an error related to workflow execution logs. Includes a `code` (specific error code from a union of possible errors), a `message`, and optional `details`.

```typescript
export interface ValidationError {
  field: string
  message: string
  value: unknown
}
```

*   **`ValidationError` Interface:** Defines the structure for a validation error, including the `field` that failed validation, a `message` describing the error, and the invalid `value`.

```typescript
export class LogsServiceError extends Error {
  public code: LogsError['code']
  public details?: Record<string, unknown>

  constructor(message: string, code: LogsError['code'], details?: Record<string, unknown>) {
    super(message)
    this.name = 'LogsServiceError'
    this.code = code
    this.details = details
  }
}
```

*   **`LogsServiceError` Class:** Defines a custom error class extending the built-in `Error` class. It includes a `code` (from the `LogsError` interface) and optional `details`.  This is a more structured way to handle errors within the logging service.

```typescript
export interface DatabaseOperationResult<T> {
  success: boolean
  data?: T
  error?: LogsServiceError
}
```

*   **`DatabaseOperationResult` Interface:** Defines the structure for the result of a database operation. It includes a `success` flag, optional `data` (of type `T`), and an optional `error` (of type `LogsServiceError`).

```typescript
export interface BatchInsertResult<T> {
  inserted: T[]
  failed: Array<{
    item: T
    error: string
  }>
  totalAttempted: number
  totalSucceeded: number
  totalFailed: number
}
```

*   **`BatchInsertResult` Interface:** Defines the structure for the result of a batch insert operation into a database. It includes:
    *   `inserted`: An array of items that were successfully inserted.
    *   `failed`: An array of objects containing the item that failed to insert and the associated error message.
    *   `totalAttempted`: The total number of items attempted to be inserted.
    *   `totalSucceeded`: The total number of items successfully inserted.
    *   `totalFailed`: The total number of items that failed to insert.

```typescript
export interface SnapshotService {
  createSnapshot(workflowId: string, state: WorkflowState): Promise<WorkflowExecutionSnapshot>
  getSnapshot(id: string): Promise<WorkflowExecutionSnapshot | null>
  getSnapshotByHash(workflowId: string, hash: string): Promise<WorkflowExecutionSnapshot | null>
  computeStateHash(state: WorkflowState): string
  cleanupOrphanedSnapshots(olderThanDays: number): Promise<number>
}
```

*   **`SnapshotService` Interface:** Defines the contract for a service that manages workflow execution snapshots.  It defines methods to:
    *   `createSnapshot`: Create a new snapshot of a workflow's state.
    *   `getSnapshot`: Retrieve a snapshot by its ID.
    *   `getSnapshotByHash`: Retrieve a snapshot by its hash.
    *   `computeStateHash`: Compute a hash of a workflow's state.
    *   `cleanupOrphanedSnapshots`: Remove snapshots that are no longer needed.

```typescript
export interface SnapshotCreationResult {
  snapshot: WorkflowExecutionSnapshot
  isNew: boolean
}
```

*   **`SnapshotCreationResult` Interface:** Defines the structure for the result of creating a workflow execution snapshot. It includes the `snapshot` object and a flag indicating whether the snapshot is new (`isNew`).

```typescript
export interface ExecutionLoggerService {
  startWorkflowExecution(params: {
    workflowId: string
    executionId: string
    trigger: ExecutionTrigger
    environment: ExecutionEnvironment
    workflowState: WorkflowState
  }): Promise<{
    workflowLog: WorkflowExecutionLog
    snapshot: WorkflowExecutionSnapshot
  }>

  completeWorkflowExecution(params: {
    executionId: string
    endedAt: string
    totalDurationMs: number

    costSummary: {
      totalCost: number
      totalInputCost: number
      totalOutputCost: number
      totalTokens: number
    }
    finalOutput: BlockOutputData
    traceSpans?: TraceSpan[]
  }): Promise<WorkflowExecutionLog>
}
```

*   **`ExecutionLoggerService` Interface:** Defines the contract for a service that logs workflow executions. It defines methods to:
    *   `startWorkflowExecution`: Start logging a new workflow execution.
    *   `completeWorkflowExecution`: Complete logging a workflow execution.

In summary, this file defines the data structures and service interfaces required for a robust workflow execution logging and management system.  The use of TypeScript interfaces and types ensures type safety and helps to maintain code quality and readability.
