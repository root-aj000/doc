Okay, let's break down this TypeScript code, piece by piece. This code defines a Zustand store to manage a queue of operations, handling retries, timeouts, and offline mode.  It's designed to ensure that operations are processed reliably, even in the face of network issues.

**1. Purpose of this file:**

This file implements a client-side operation queue using Zustand, a state management library.  Its primary purpose is to:

*   **Manage asynchronous operations:**  It holds a queue of operations that need to be performed, particularly those that interact with a backend server.
*   **Ensure reliability:** It handles retries with exponential backoff, timeouts, and offline scenarios to make sure operations are eventually processed or handled gracefully.
*   **Maintain data consistency:**  By queuing operations, it helps prevent race conditions and ensures that updates are applied in the correct order.
*   **Optimize performance:** The queue implementation coalesces operations, dropping redundant updates to the same variable or subblock.
*   **Provide a centralized point of control:**  The store provides methods to add, confirm, fail, cancel, and process operations, making it easier to manage the overall operation flow.
*   **Graceful Failure:** Implements an offline mode that can be triggered when operations fail too many times.

**2. Code Explanation:**

```typescript
import { create } from 'zustand'
import { createLogger } from '@/lib/logs/console/logger'

const logger = createLogger('OperationQueue')
```

*   **`import { create } from 'zustand'`:** Imports the `create` function from the `zustand` library.  `zustand` is a minimalist state management solution for React.  The `create` function is used to define the store.
*   **`import { createLogger } from '@/lib/logs/console/logger'`:**  Imports a custom logger function from a local module.  This is likely used for debugging and monitoring the operation queue.  The `@` alias suggests a configured path in the build/toolchain setup (e.g. webpack, vite).
*   **`const logger = createLogger('OperationQueue')`:** Creates a logger instance specifically for this module, tagging logs with "OperationQueue" for easier filtering and debugging.

```typescript
export interface QueuedOperation {
  id: string
  operation: {
    operation: string
    target: string
    payload: any
  }
  workflowId: string
  timestamp: number
  retryCount: number
  status: 'pending' | 'processing' | 'confirmed' | 'failed'
  userId: string
  immediate?: boolean // Flag for immediate processing (skips debouncing)
}
```

*   **`export interface QueuedOperation`:** Defines the structure of a single operation within the queue.
    *   **`id: string`:** A unique identifier for the operation. This is crucial for tracking and managing operations.
    *   **`operation: { operation: string; target: string; payload: any }`:**  Describes the operation to be performed.
        *   **`operation: string`:**  The type of operation (e.g., 'create', 'update', 'delete', 'subblock-update', 'variable-update').
        *   **`target: string`:**  The target of the operation (e.g., 'block', 'variable', 'subblock').
        *   **`payload: any`:**  The data associated with the operation. This could be anything from a complete object to a simple value.  The use of `any` is generally discouraged in TypeScript in favor of more specific types but is appropriate here since the structure of the payload will vary widely based on the `operation` and `target`.
    *   **`workflowId: string`**:  The ID of the workflow this operation belongs to.
    *   **`timestamp: number`:**  The time the operation was added to the queue (in milliseconds since the Unix epoch).
    *   **`retryCount: number`:**  The number of times this operation has been retried.
    *   **`status: 'pending' | 'processing' | 'confirmed' | 'failed'`:**  The current status of the operation.
        *   `pending`:  Waiting to be processed.
        *   `processing`: Currently being executed.
        *   `confirmed`:  Successfully completed.
        *   `failed`:  Failed to complete (possibly after retries).
    *   **`userId: string`**: The ID of the user that initiated the operation.
    *   **`immediate?: boolean`**:  An optional flag. If true, this operation should be processed immediately, bypassing any debouncing or delay mechanisms (though none is implemented in this code, this flag allows for future use).  The `?` makes this property optional.

```typescript
interface OperationQueueState {
  operations: QueuedOperation[]
  isProcessing: boolean
  hasOperationError: boolean

  addToQueue: (operation: Omit<QueuedOperation, 'timestamp' | 'retryCount' | 'status'>) => void
  confirmOperation: (operationId: string) => void
  failOperation: (operationId: string, retryable?: boolean) => void
  handleOperationTimeout: (operationId: string) => void
  processNextOperation: () => void
  cancelOperationsForBlock: (blockId: string) => void
  cancelOperationsForVariable: (variableId: string) => void
  cancelOperationsForWorkflow: (workflowId: string) => void

  triggerOfflineMode: () => void
  clearError: () => void
}
```

*   **`interface OperationQueueState`:** Defines the structure of the Zustand store's state.
    *   **`operations: QueuedOperation[]`:** The array holding all queued operations.
    *   **`isProcessing: boolean`:**  A flag indicating whether an operation is currently being processed.  This helps prevent multiple operations from running concurrently.
    *   **`hasOperationError: boolean`:** A flag to indicate if there was an error in any operation. This is used to trigger an offline mode when fatal errors are encountered.
    *   **`addToQueue: (operation: Omit<QueuedOperation, 'timestamp' | 'retryCount' | 'status'>) => void`:**  A function to add a new operation to the queue. It uses `Omit` to require all properties of `QueuedOperation` except `timestamp`, `retryCount`, and `status`, which are automatically managed by the store.
    *   **`confirmOperation: (operationId: string) => void`:**  A function to mark an operation as successfully completed, removing it from the queue.
    *   **`failOperation: (operationId: string, retryable?: boolean) => void`:** A function to mark an operation as failed. It handles retries if `retryable` is true (default).
    *   **`handleOperationTimeout: (operationId: string) => void`:** A function called when an operation takes too long to complete, triggering a retry.
    *   **`processNextOperation: () => void`:** A function that starts processing the next pending operation in the queue.
    *   **`cancelOperationsForBlock: (blockId: string) => void`:** Cancels all operations related to a specific block, identified by `blockId`.
    *   **`cancelOperationsForVariable: (variableId: string) => void`:** Cancels all operations related to a specific variable, identified by `variableId`.
    *  **`cancelOperationsForWorkflow: (workflowId: string) => void`:** Cancels all operations related to a specific workflow, identified by `workflowId`.
    *   **`triggerOfflineMode: () => void`:** A function to set the store into an offline mode, typically when operations fail repeatedly.
    *   **`clearError: () => void`:** A function to clear the `hasOperationError` flag, typically when the application comes back online.

```typescript
const retryTimeouts = new Map<string, NodeJS.Timeout>()
const operationTimeouts = new Map<string, NodeJS.Timeout>()
```

*   **`const retryTimeouts = new Map<string, NodeJS.Timeout>()`:**  A `Map` to store `NodeJS.Timeout` objects associated with retry attempts for specific operations.  The key is the `operationId`. This allows the code to cancel pending retries if necessary.
*   **`const operationTimeouts = new Map<string, NodeJS.Timeout>()`:**  A `Map` to store `NodeJS.Timeout` objects associated with operation timeouts.  The key is the `operationId`. This is used to detect operations that are taking too long and trigger a retry or failure.

```typescript
let emitWorkflowOperation:
  | ((operation: string, target: string, payload: any, operationId?: string) => void)
  | null = null
let emitSubblockUpdate:
  | ((blockId: string, subblockId: string, value: any, operationId?: string) => void)
  | null = null
let emitVariableUpdate:
  | ((variableId: string, field: string, value: any, operationId?: string) => void)
  | null = null
```

*   These variables declare function types that will be assigned with functions to actually perform the operations. They're initialized to `null` and later set via `registerEmitFunctions`.  This pattern is likely used for decoupling the operation queue from the specific implementation of the operations.
    *   `emitWorkflowOperation`: A function to emit a general workflow operation.
    *   `emitSubblockUpdate`: A function to emit a subblock update operation.
    *   `emitVariableUpdate`: A function to emit a variable update operation.

```typescript
export function registerEmitFunctions(
  workflowEmit: (operation: string, target: string, payload: any, operationId?: string) => void,
  subblockEmit: (blockId: string, subblockId: string, value: any, operationId?: string) => void,
  variableEmit: (variableId: string, field: string, value: any, operationId?: string) => void,
  workflowId: string | null
) {
  emitWorkflowOperation = workflowEmit
  emitSubblockUpdate = subblockEmit
  emitVariableUpdate = variableEmit
  currentRegisteredWorkflowId = workflowId
}
```

*   **`export function registerEmitFunctions(...)`:** A function to register the "emit" functions and the `workflowId`.  This is how the operation queue gets the functions it needs to actually *perform* the operations.
    *   It takes three function arguments (`workflowEmit`, `subblockEmit`, `variableEmit`) and `workflowId` which will be assigned to the corresponding `emit...` variables.
    *   This allows you to inject the specific implementations of these functions from elsewhere in your application. The workflowId is very important as this restricts the queue to a single workflow

```typescript
let currentRegisteredWorkflowId: string | null = null
```

*   A variable to hold the currently registered workflow ID. Only operations related to this workflow will be processed.

```typescript
export const useOperationQueueStore = create<OperationQueueState>((set, get) => ({
  operations: [],
  isProcessing: false,
  hasOperationError: false,

  addToQueue: (operation) => {
    // Immediate coalescing without client-side debouncing:
    // For subblock updates, keep only latest pending op for the same blockId+subblockId
    if (
      operation.operation.operation === 'subblock-update' &&
      operation.operation.target === 'subblock'
    ) {
      const { blockId, subblockId } = operation.operation.payload
      set((state) => ({
        operations: [
          ...state.operations.filter(
            (op) =>
              !(
                op.status === 'pending' &&
                op.operation.operation === 'subblock-update' &&
                op.operation.target === 'subblock' &&
                op.operation.payload?.blockId === blockId &&
                op.operation.payload?.subblockId === subblockId
              )
          ),
        ],
      }))
    }

    // For variable updates, keep only latest pending op for same variableId+field
    if (
      operation.operation.operation === 'variable-update' &&
      operation.operation.target === 'variable'
    ) {
      const { variableId, field } = operation.operation.payload
      set((state) => ({
        operations: [
          ...state.operations.filter(
            (op) =>
              !(
                op.status === 'pending' &&
                op.operation.operation === 'variable-update' &&
                op.operation.target === 'variable' &&
                op.operation.payload?.variableId === variableId &&
                op.operation.payload?.field === field
              )
          ),
        ],
      }))
    }

    // Handle remaining logic
    const state = get()

    // Check for duplicate operation ID
    const existingOp = state.operations.find((op) => op.id === operation.id)
    if (existingOp) {
      logger.debug('Skipping duplicate operation ID', {
        operationId: operation.id,
        existingStatus: existingOp.status,
      })
      return
    }

    // Enhanced duplicate content check - especially important for block operations
    const duplicateContent = state.operations.find(
      (op) =>
        op.operation.operation === operation.operation.operation &&
        op.operation.target === operation.operation.target &&
        op.workflowId === operation.workflowId &&
        // For block operations, check the block ID specifically
        ((operation.operation.target === 'block' &&
          op.operation.payload?.id === operation.operation.payload?.id) ||
          // For other operations, fall back to full payload comparison
          (operation.operation.target !== 'block' &&
            JSON.stringify(op.operation.payload) === JSON.stringify(operation.operation.payload)))
    )

    if (duplicateContent) {
      logger.debug('Skipping duplicate operation content', {
        operationId: operation.id,
        existingOperationId: duplicateContent.id,
        operation: operation.operation.operation,
        target: operation.operation.target,
        existingStatus: duplicateContent.status,
        payload:
          operation.operation.target === 'block'
            ? { id: operation.operation.payload?.id }
            : operation.operation.payload,
      })
      return
    }

    const queuedOp: QueuedOperation = {
      ...operation,
      timestamp: Date.now(),
      retryCount: 0,
      status: 'pending',
    }

    logger.debug('Adding operation to queue', {
      operationId: queuedOp.id,
      operation: queuedOp.operation,
    })

    set((state) => ({
      operations: [...state.operations, queuedOp],
    }))

    // Start processing if not already processing
    get().processNextOperation()
  },

  confirmOperation: (operationId) => {
    const state = get()
    const operation = state.operations.find((op) => op.id === operationId)
    const newOperations = state.operations.filter((op) => op.id !== operationId)

    const retryTimeout = retryTimeouts.get(operationId)
    if (retryTimeout) {
      clearTimeout(retryTimeout)
      retryTimeouts.delete(operationId)
    }

    const operationTimeout = operationTimeouts.get(operationId)
    if (operationTimeout) {
      clearTimeout(operationTimeout)
      operationTimeouts.delete(operationId)
    }

    logger.debug('Removing operation from queue', {
      operationId,
      remainingOps: newOperations.length,
    })

    set({ operations: newOperations, isProcessing: false })

    // Process next operation in queue
    get().processNextOperation()
  },

  failOperation: (operationId: string, retryable = true) => {
    const state = get()
    const operation = state.operations.find((op) => op.id === operationId)
    if (!operation) {
      logger.warn('Attempted to fail operation that does not exist in queue', { operationId })
      return
    }

    const operationTimeout = operationTimeouts.get(operationId)
    if (operationTimeout) {
      clearTimeout(operationTimeout)
      operationTimeouts.delete(operationId)
    }

    if (!retryable) {
      logger.debug('Operation marked as non-retryable, removing from queue', { operationId })

      set((state) => ({
        operations: state.operations.filter((op) => op.id !== operationId),
        isProcessing: false,
      }))

      get().processNextOperation()
      return
    }

    // More aggressive retry for subblock/variable updates, less aggressive for structural ops
    const isSubblockOrVariable =
      (operation.operation.operation === 'subblock-update' &&
        operation.operation.target === 'subblock') ||
      (operation.operation.operation === 'variable-update' &&
        operation.operation.target === 'variable')

    const maxRetries = isSubblockOrVariable ? 5 : 3 // 5 retries for text, 3 for structural

    if (operation.retryCount < maxRetries) {
      const newRetryCount = operation.retryCount + 1
      // Faster retries for subblock/variable, exponential for structural
      const delay = isSubblockOrVariable
        ? Math.min(1000 * newRetryCount, 3000) // 1s, 2s, 3s, 3s, 3s (cap at 3s)
        : 2 ** newRetryCount * 1000 // 2s, 4s, 8s (exponential for structural)

      logger.warn(
        `Operation failed, retrying in ${delay}ms (attempt ${newRetryCount}/${maxRetries})`,
        {
          operationId,
          retryCount: newRetryCount,
          operation: operation.operation.operation,
        }
      )

      // Update retry count and mark as pending for retry
      set((state) => ({
        operations: state.operations.map((op) =>
          op.id === operationId
            ? { ...op, retryCount: newRetryCount, status: 'pending' as const }
            : op
        ),
        isProcessing: false, // Allow processing to continue
      }))

      // Schedule retry
      const timeout = setTimeout(() => {
        retryTimeouts.delete(operationId)
        get().processNextOperation()
      }, delay)

      retryTimeouts.set(operationId, timeout)
    } else {
      // Always trigger offline mode when we can't persist - never silently drop data
      logger.error('Operation failed after max retries, triggering offline mode', {
        operationId,
        operation: operation.operation.operation,
        retryCount: operation.retryCount,
      })
      get().triggerOfflineMode()
    }
  },

  handleOperationTimeout: (operationId: string) => {
    const state = get()
    const operation = state.operations.find((op) => op.id === operationId)
    if (!operation) {
      logger.debug('Ignoring timeout for operation not in queue', { operationId })
      return
    }

    logger.warn('Operation timeout detected - treating as failure to trigger retries', {
      operationId,
    })

    get().failOperation(operationId)
  },

  processNextOperation: () => {
    const state = get()

    // Don't process if already processing
    if (state.isProcessing) {
      return
    }

    const nextOperation = currentRegisteredWorkflowId
      ? state.operations.find(
          (op) => op.status === 'pending' && op.workflowId === currentRegisteredWorkflowId
        )
      : state.operations.find((op) => op.status === 'pending')
    if (!nextOperation) {
      return
    }

    if (currentRegisteredWorkflowId && nextOperation.workflowId !== currentRegisteredWorkflowId) {
      return
    }

    // Mark as processing
    set((state) => ({
      operations: state.operations.map((op) =>
        op.id === nextOperation.id ? { ...op, status: 'processing' as const } : op
      ),
      isProcessing: true,
    }))

    logger.debug('Processing operation sequentially', {
      operationId: nextOperation.id,
      operation: nextOperation.operation,
      retryCount: nextOperation.retryCount,
    })

    // Emit the operation
    const { operation: op, target, payload } = nextOperation.operation
    if (op === 'subblock-update' && target === 'subblock') {
      if (emitSubblockUpdate) {
        emitSubblockUpdate(payload.blockId, payload.subblockId, payload.value, nextOperation.id)
      }
    } else if (op === 'variable-update' && target === 'variable') {
      if (emitVariableUpdate) {
        emitVariableUpdate(payload.variableId, payload.field, payload.value, nextOperation.id)
      }
    } else {
      if (emitWorkflowOperation) {
        emitWorkflowOperation(op, target, payload, nextOperation.id)
      }
    }

    // Create operation timeout - longer for subblock/variable updates to handle reconnects
    const isSubblockOrVariable =
      (nextOperation.operation.operation === 'subblock-update' &&
        nextOperation.operation.target === 'subblock') ||
      (nextOperation.operation.operation === 'variable-update' &&
        nextOperation.operation.target === 'variable')
    const timeoutDuration = isSubblockOrVariable ? 15000 : 5000 // 15s for text edits, 5s for structural ops

    const timeoutId = setTimeout(() => {
      logger.warn(`Operation timeout - no server response after ${timeoutDuration}ms`, {
        operationId: nextOperation.id,
        operation: nextOperation.operation.operation,
      })
      operationTimeouts.delete(nextOperation.id)
      get().handleOperationTimeout(nextOperation.id)
    }, timeoutDuration)

    operationTimeouts.set(nextOperation.id, timeoutId)
  },

  cancelOperationsForBlock: (blockId: string) => {
    logger.debug('Canceling all operations for block', { blockId })

    // No debounced timeouts to cancel (moved to server-side)

    // Find and cancel operation timeouts for operations related to this block
    const state = get()
    const operationsToCancel = state.operations.filter(
      (op) =>
        (op.operation.target === 'block' && op.operation.payload?.id === blockId) ||
        (op.operation.target === 'subblock' && op.operation.payload?.blockId === blockId)
    )

    // Cancel timeouts for these operations
    operationsToCancel.forEach((op) => {
      const operationTimeout = operationTimeouts.get(op.id)
      if (operationTimeout) {
        clearTimeout(operationTimeout)
        operationTimeouts.delete(op.id)
      }

      const retryTimeout = retryTimeouts.get(op.id)
      if (retryTimeout) {
        clearTimeout(retryTimeout)
        retryTimeouts.delete(op.id)
      }
    })

    // Remove all operations for this block (both pending and processing)
    const newOperations = state.operations.filter(
      (op) =>
        !(
          (op.operation.target === 'block' && op.operation.payload?.id === blockId) ||
          (op.operation.target === 'subblock' && op.operation.payload?.blockId === blockId)
        )
    )

    set({
      operations: newOperations,
      isProcessing: false, // Reset processing state in case we removed the current operation
    })

    logger.debug('Cancelled operations for block', {
      blockId,
      cancelledOperations: operationsToCancel.length,
    })

    // Process next operation if there are any remaining
    get().processNextOperation()
  },

  cancelOperationsForVariable: (variableId: string) => {
    logger.debug('Canceling all operations for variable', { variableId })

    // No debounced timeouts to cancel (moved to server-side)

    // Find and cancel operation timeouts for operations related to this variable
    const state = get()
    const operationsToCancel = state.operations.filter(
      (op) =>
        (op.operation.target === 'variable' && op.operation.payload?.variableId === variableId) ||
        (op.operation.target === 'variable' &&
          op.operation.payload?.sourceVariableId === variableId)
    )

    // Cancel timeouts for these operations
    operationsToCancel.forEach((op) => {
      const operationTimeout = operationTimeouts.get(op.id)
      if (operationTimeout) {
        clearTimeout(operationTimeout)
        operationTimeouts.delete(op.id)
      }

      const retryTimeout = retryTimeouts.get(op.id)
      if (retryTimeout) {
        clearTimeout(retryTimeout)
        retryTimeouts.delete(op.id)
      }
    })

    // Remove all operations for this variable (both pending and processing)
    const newOperations = state.operations.filter(
      (op) =>
        !(
          (op.operation.target === 'variable' && op.operation.payload?.variableId === variableId) ||
          (op.operation.target === 'variable' &&
            op.operation.payload?.sourceVariableId === variableId)
        )
    )

    set({
      operations: newOperations,
      isProcessing: false, // Reset processing state in case we removed the current operation
    })

    logger.debug('Cancelled operations for variable', {
      variableId,
      cancelledOperations: operationsToCancel.length,
    })

    // Process next operation if there are any remaining
    get().processNextOperation()
  },

  cancelOperationsForWorkflow: (workflowId: string) => {
    const state = get()
    retryTimeouts.forEach((timeout, opId) => {
      const op = state.operations.find((o) => o.id === opId)
      if (op && op.workflowId === workflowId) {
        clearTimeout(timeout)
        retryTimeouts.delete(opId)
      }
    })
    operationTimeouts.forEach((timeout, opId) => {
      const op = state.operations.find((o) => o.id === opId)
      if (op && op.workflowId === workflowId) {
        clearTimeout(timeout)
        operationTimeouts.delete(opId)
      }
    })
    set((s) => ({
      operations: s.operations.filter((op) => op.workflowId !== workflowId),
      isProcessing: false,
    }))
  },

  triggerOfflineMode: () => {
    logger.error('Operation failed after retries - triggering offline mode')

    retryTimeouts.forEach((timeout) => clearTimeout(timeout))
    retryTimeouts.clear()
    operationTimeouts.forEach((timeout) => clearTimeout(timeout))
    operationTimeouts.clear()

    set({
      operations: [],
      isProcessing: false,
      hasOperationError: true,
    })
  },

  clearError: () => {
    set({ hasOperationError: false })
  },
}))
```

*   **`export const useOperationQueueStore = create<OperationQueueState>((set, get) => { ... })`:**  This is the heart of the code. It creates the Zustand store.
    *   `create<OperationQueueState>`:  Tells Zustand that the store will manage state that conforms to the `OperationQueueState` interface.
    *   `(set, get) => ({ ... })`:  This is the store's definition. `set` is a function to update the state, and `get` is a function to access the current state.

    Let's break down the methods defined inside the `create` function:

    *   **`operations: [], isProcessing: false, hasOperationError: false`:**  Initial state values.

    *   **`addToQueue: (operation) => { ... }`:**
        *   **Coalescing:** Before adding the operation, it checks for duplicate operations to optimize performance, specifically for `subblock-update` and `variable-update` operations. For these operations, if a pending operation exists for the same target (blockId/subblockId or variableId/field), the old operation is replaced with the new one.
        *   **Duplicate ID Check:** It checks if an operation with the same `id` already exists in the queue. If so, it skips adding the new operation.
        *   **Enhanced Duplicate Content Check:**  It checks if an operation with the same content already exists in the queue (same operation type, target, workflowId and either the block ID if it's a block operation, or the stringified payload for all other operation types).  This prevents redundant operations.
        *   **Creating the Queued Operation:**  Creates a new `QueuedOperation` object, setting the `timestamp`, `retryCount`, and `status`.
        *   **Adding to the Queue:** Adds the new operation to the `operations` array using `set`.  The spread operator (`...state.operations`) ensures that the existing operations are preserved.
        *   **Starting Processing:**  Calls `get().processNextOperation()` to start processing the queue if it's not already running.

    *   **`confirmOperation: (operationId) => { ... }`:**
        *   **Finding and Removing:** Finds the operation with the given `operationId` and removes it from the `operations` array using `filter`.
        *   **Clearing Timeouts:** Clears both the retry timeout and the operation timeout associated with the confirmed operation, if they exist. This is important to prevent retries or timeouts from firing after the operation has been successfully confirmed.
        *   **Updating State:**  Updates the state by setting the new `operations` array.
        *   **Starting Next Operation:** Calls `get().processNextOperation()` to process the next operation in the queue.

    *   **`failOperation: (operationId, retryable = true) => { ... }`:**
        *   **Finding the Operation:** Finds the operation with the given `operationId`.
        *   **Clearing Timeout:** Clears the operation timeout associated with the failed operation.
        *   **Non-Retryable Handling:** If `retryable` is `false`, the operation is removed from the queue, and `processNextOperation` is called.
        *   **Retry Logic:**
            *   Checks if the `retryCount` is less than `maxRetries` (5 for subblock/variable updates, 3 for other operations).
            *   Calculates a `delay` for the retry. The delay is exponential for structural operations and capped for subblock/variable updates.
            *   Logs a warning message.
            *   Updates the operation's `retryCount` and `status` to 'pending'.
            *   Sets a timeout using `setTimeout` to call `processNextOperation` after the calculated delay.
            *   Stores the timeout ID in the `retryTimeouts` map.
        *   **Offline Mode:** If `retryCount` reaches `maxRetries`, it triggers `triggerOfflineMode`, indicating a persistent failure.

    *   **`handleOperationTimeout: (operationId) => { ... }`:**
        *   **Finds the Operation:** Find the operation in the queue
        *   **Treats as Failure:**  Called when an operation times out. It treats the timeout as a failure and calls `failOperation` to initiate a retry.

    *   **`processNextOperation: () => { ... }`:**
        *   **Check if Already Processing:** Checks if `isProcessing` is true. If so, it returns, preventing concurrent processing.
        *   **Finds the Next Operation:** Finds the next `pending` operation in the queue to process.
        *   **Workflow check:** If the workflow id is set, it only grabs operations that match the workflow ID
        *   **Mark as Processing:** Sets the `status` of the next operation to 'processing' and sets `isProcessing` to `true`.
        *   **Emit the Operation:**
            *   Extracts the `operation`, `target`, and `payload` from the operation.
            *   Calls the appropriate "emit" function (`emitSubblockUpdate`, `emitVariableUpdate`, or `emitWorkflowOperation`) based on the `operation` and `target`.  It passes the payload and the operation ID to the emit function.  The emit functions are responsible for actually executing the operation (e.g., sending a request to the server).
        *   **Set Operation Timeout:**
            *   Sets a timeout using `setTimeout` to call `handleOperationTimeout` after a specified duration.  The timeout duration is longer for subblock/variable updates (15 seconds) than for other operations (5 seconds).
            *   Stores the timeout ID in the `operationTimeouts` map.

    *   **`cancelOperationsForBlock: (blockId) => { ... }`:**
        *   **Cancels Timeouts:** Cancels any pending retry timeouts or operation timeouts for operations related to the given `blockId`.
        *   **Removes Operations:** Removes all operations related to the given `blockId` from the queue.
        *   **Resets Processing State:** Sets `isProcessing` to `false`.
        *   **Processes Next Operation:** Calls `get().processNextOperation()` to start processing the next operation in the queue.

    *   **`cancelOperationsForVariable: (variableId) => { ... }`:**
        *   **Cancels Timeouts:** Cancels any pending retry timeouts or operation timeouts for operations related to the given `variableId`.
        *   **Removes Operations:** Removes all operations related to the given `variableId` from the queue.
        *   **Resets Processing State:** Sets `isProcessing` to `false`.
        *   **Processes Next Operation:** Calls `get().processNextOperation()` to start processing the next operation in the queue.

     *   **`cancelOperationsForWorkflow: (workflowId) => { ... }`:**
        *   **Cancels Timeouts:** Cancels any pending retry timeouts or operation timeouts for operations related to the given `workflowId`.
        *   **Removes Operations:** Removes all operations related to the given `workflowId` from the queue.
        *   **Resets Processing State:** Sets `isProcessing` to `false`.

    *   **`triggerOfflineMode: () => { ... }`:**
        *   **Cancels Timeouts:** Cancels all pending retry timeouts and operation timeouts.
        *   **Clears the Queue:**  Empties the `operations` array.
        *   **Sets Error Flag:** Sets `hasOperationError` to `true`.

    *   **`clearError: () => { ... }`:**
        *   **Clears Error Flag:** Sets `hasOperationError` to `false`.

```typescript
export function useOperationQueue() {
  const store = useOperationQueueStore()

  return {
    queue: store.operations,
    isProcessing: store.isProcessing,
    hasOperationError: store.hasOperationError,
    addToQueue: store.addToQueue,
    confirmOperation: store.confirmOperation,
    failOperation: store.failOperation,
    processNextOperation: store.processNextOperation,
    cancelOperationsForBlock: store.cancelOperationsForBlock,
    cancelOperationsForVariable: store.cancelOperationsForVariable,
    triggerOfflineMode: store.triggerOfflineMode,
    clearError: store.clearError,
  }
}
```

*   