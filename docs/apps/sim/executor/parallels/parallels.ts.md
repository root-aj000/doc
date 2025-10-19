This TypeScript file defines the `ParallelManager` class, a crucial component for handling the execution of "parallel" blocks within a larger workflow or system. Its primary role is to manage the distribution of items across multiple parallel executions, track their progress, and aggregate their results.

Imagine you have a list of tasks, and you want to process each task concurrently. A "parallel" block in this system allows you to define a sub-workflow that runs for each item in that list. The `ParallelManager` is the orchestrator that makes this happen, ensuring each item gets processed, and the system knows when all parallel executions are complete.

### Core Concepts

Before diving into the code, let's clarify a few terms used in this file:

*   **Parallel Block (`SerializedParallel`)**: A special type of block in the workflow that represents a loop or concurrent execution. It takes a list of "distribution items" (e.g., a list of users, files, or configurations) and runs its internal sub-workflow for each item.
*   **Virtual Block Instance**: When a parallel block runs, it doesn't just run once. For each "distribution item," it creates a "virtual" copy of its internal workflow. These are referred to as virtual block instances (e.g., `blockId_parallel_parallelId_iteration_0`, `blockId_parallel_parallelId_iteration_1`).
*   **Iteration**: Each run of the parallel block's sub-workflow for a single distribution item is called an iteration.
*   **`ExecutionContext`**: A central object that holds the current state of the entire workflow execution, including which blocks have run, which are active, parallel states, and results.
*   **`ParallelState`**: An object that specifically tracks the state of a single parallel block's execution (e.g., how many items, how many completed, results).

---

### File Breakdown

This file primarily exports an interface `ParallelState` and the `ParallelManager` class.

#### Imports

```typescript
import { createLogger } from '@/lib/logs/console/logger'
import { ParallelRoutingUtils } from '@/executor/parallels/utils'
import type { ExecutionContext, NormalizedBlockOutput } from '@/executor/types'
import type { SerializedBlock, SerializedParallel, SerializedWorkflow } from '@/serializer/types'
```

*   `createLogger`: Imports a utility to create a logger instance, used here for debugging and informational messages specific to parallel execution.
*   `ParallelRoutingUtils`: Imports a helper utility that deals with the specific routing logic required for parallel blocks, particularly for checking if all *necessary* virtual blocks have executed.
*   `ExecutionContext`, `NormalizedBlockOutput`: Type definitions for the overall execution context and the standardized output format of a block.
*   `SerializedBlock`, `SerializedParallel`, `SerializedWorkflow`: Type definitions for the serialized (JSON-representable) structures of a block, a parallel block, and the entire workflow. These are used to represent the workflow structure loaded from configuration.

```typescript
const logger = createLogger('ParallelManager')
```

*   This line initializes a logger instance specifically for the `ParallelManager` class. Log messages from this class will be prefixed with 'ParallelManager', making it easier to filter and understand logs during execution.

---

### `ParallelState` Interface

```typescript
export interface ParallelState {
  parallelCount: number
  distributionItems: any[] | Record<string, any> | null
  completedExecutions: number
  executionResults: Map<string, any>
  activeIterations: Set<number>
  currentIteration: number
}
```

This interface defines the structure for tracking the state of a *single* parallel block's execution. Each parallel block in the workflow will have its own `ParallelState` object managed by the system.

*   `parallelCount`: The total number of items to be processed in parallel. This is derived from the `distributionItems`.
*   `distributionItems`: The actual data items that need to be processed by the parallel block. This can be an array (for ordered items) or a record/object (for keyed items). It can be `null` if not yet initialized.
*   `completedExecutions`: A counter for how many individual iterations (virtual block instances) have fully completed their execution for this parallel block.
*   `executionResults`: A `Map` that stores the results from each completed iteration. The keys are typically strings representing the iteration (e.g., `iteration_0`), and values are the output from that iteration.
*   `activeIterations`: A `Set` of numbers representing the indices of iterations that are currently in progress. This helps track which iterations are still running.
*   `currentIteration`: A number indicating the current iteration being processed. While `activeIterations` tracks *all* currently running iterations, `currentIteration` might be used in some contexts to refer to the "next" or "primary" iteration being set up.

---

### `ParallelManager` Class

```typescript
/**
 * Manages parallel block execution and state.
 * Handles distribution of items across parallel executions and tracking completion.
 */
export class ParallelManager {
  constructor(private parallels: SerializedWorkflow['parallels'] = {}) {}

  // ... methods ...
}
```

This is the main class responsible for orchestrating parallel executions.

*   **Purpose**: To initialize, track, and finalize the execution of parallel blocks within a workflow. It handles the logic of distributing items, creating virtual block instances, managing their states, and aggregating results.
*   **Constructor**:
    ```typescript
    constructor(private parallels: SerializedWorkflow['parallels'] = {}) {}
    ```
    *   The constructor takes an optional argument `parallels`, which is expected to be a map of parallel block IDs to their `SerializedParallel` definitions. This essentially gives the manager a blueprint of all parallel blocks in the workflow.
    *   `private parallels`: This makes `parallels` a private member of the class, meaning it can only be accessed from within `ParallelManager` methods. It's automatically assigned by TypeScript due to the `private` keyword in the constructor. The default value `{}` ensures that if no parallels are provided, it initializes to an empty object.

#### Methods of `ParallelManager`

---

#### `initializeParallel`

```typescript
  /**
   * Initializes a parallel execution state.
   */
  initializeParallel(
    parallelId: string,
    distributionItems: any[] | Record<string, any>
  ): ParallelState {
    const parallelCount = Array.isArray(distributionItems)
      ? distributionItems.length
      : Object.keys(distributionItems).length

    return {
      parallelCount,
      distributionItems,
      completedExecutions: 0,
      executionResults: new Map(),
      activeIterations: new Set(),
      currentIteration: 1,
    }
  }
```

*   **Purpose**: To create and return an initial `ParallelState` object for a given parallel block. This is called when a parallel block is first encountered in the workflow execution.
*   **Parameters**:
    *   `parallelId`: The unique identifier of the parallel block.
    *   `distributionItems`: The actual data (an array or an object) that will be iterated over by the parallel block.
*   **Line-by-line explanation**:
    *   `const parallelCount = ...`: This line calculates the total number of iterations required.
        *   `Array.isArray(distributionItems) ? distributionItems.length : ...`: If `distributionItems` is an array, `parallelCount` is its length.
        *   `: Object.keys(distributionItems).length`: Otherwise (if it's an object/record), `parallelCount` is the number of keys in the object.
    *   `return { ... }`: Returns a new `ParallelState` object initialized with:
        *   `parallelCount`: The calculated count.
        *   `distributionItems`: The provided items.
        *   `completedExecutions: 0`: Initially, no iterations have completed.
        *   `executionResults: new Map()`: An empty `Map` to store results later.
        *   `activeIterations: new Set()`: An empty `Set` as no iterations are active yet.
        *   `currentIteration: 1`: Sets the initial iteration counter to 1 (or the first index, depending on usage).

---

#### `getIterationItem`

```typescript
  /**
   * Gets the current item for a specific parallel iteration.
   */
  getIterationItem(parallelState: ParallelState, iterationIndex: number): any {
    if (!parallelState.distributionItems) {
      return null
    }

    if (Array.isArray(parallelState.distributionItems)) {
      return parallelState.distributionItems[iterationIndex]
    }
    return Object.entries(parallelState.distributionItems)[iterationIndex]
  }
```

*   **Purpose**: To retrieve a specific item from the `distributionItems` based on its index for a given parallel iteration.
*   **Parameters**:
    *   `parallelState`: The current state object of the parallel block.
    *   `iterationIndex`: The 0-based index of the iteration for which to retrieve the item.
*   **Line-by-line explanation**:
    *   `if (!parallelState.distributionItems) { return null }`: If there are no distribution items (e.g., the state hasn't been fully initialized or items are missing), return `null`.
    *   `if (Array.isArray(parallelState.distributionItems)) { ... }`: If the `distributionItems` is an array:
        *   `return parallelState.distributionItems[iterationIndex]`: Directly return the item at the specified `iterationIndex`.
    *   `return Object.entries(parallelState.distributionItems)[iterationIndex]`: If `distributionItems` is an object, `Object.entries()` converts it into an array of `[key, value]` pairs. This line returns the `[key, value]` pair at the specified `iterationIndex`. This allows parallel processing of both arrays of values and objects with key-value pairs.

---

#### `areAllVirtualBlocksExecuted`

```typescript
  /**
   * Checks if all virtual blocks that SHOULD execute for a parallel have been executed.
   * This now respects conditional routing - only checks blocks that should execute.
   */
  areAllVirtualBlocksExecuted(
    parallelId: string,
    parallel: SerializedParallel,
    executedBlocks: Set<string>,
    parallelState: ParallelState,
    context: ExecutionContext
  ): boolean {
    const result = ParallelRoutingUtils.areAllRequiredVirtualBlocksExecuted(
      parallel,
      parallelState.parallelCount,
      executedBlocks,
      context
    )

    if (result) {
      logger.info(`All required virtual blocks completed for parallel ${parallelId}`)
    } else {
      logger.info(`Parallel ${parallelId} not complete - some blocks still need to execute`)
    }

    return result
  }
```

*   **Purpose**: To determine if all *necessary* virtual blocks associated with a parallel block have finished executing. This is crucial for knowing when a parallel block itself can be considered complete. It's "smart" because it takes conditional routing into account (i.e., if a virtual block was skipped due to a condition, it's not considered "required").
*   **Parameters**:
    *   `parallelId`: The ID of the parallel block.
    *   `parallel`: The `SerializedParallel` definition for the block.
    *   `executedBlocks`: A `Set` from the `ExecutionContext` containing the IDs of all blocks that have completed execution so far.
    *   `parallelState`: The current `ParallelState` object for this parallel block.
    *   `context`: The overall `ExecutionContext`.
*   **Line-by-line explanation**:
    *   `const result = ParallelRoutingUtils.areAllRequiredVirtualBlocksExecuted(...)`: This line delegates the actual complex logic to `ParallelRoutingUtils`. This utility function is responsible for analyzing the parallel block's structure, the number of iterations, the `executedBlocks` set, and the `ExecutionContext` to determine if all *relevant* virtual blocks have been executed.
    *   `if (result) { ... } else { ... }`: Logs an informational message indicating whether all required virtual blocks have completed or if there are still some pending.
    *   `return result`: Returns the boolean result from the utility function.

---

#### `processParallelIterations`

```typescript
  /**
   * Processes parallel iterations to check for completion and trigger re-execution.
   */
  async processParallelIterations(context: ExecutionContext): Promise<void> {
    if (!this.parallels || Object.keys(this.parallels).length === 0) {
      return
    }

    for (const [parallelId, parallel] of Object.entries(this.parallels)) {
      // Skip if this parallel has already been marked as completed
      if (context.completedLoops.has(parallelId)) {
        continue
      }

      // Check if the parallel block itself has been executed
      const parallelBlockExecuted = context.executedBlocks.has(parallelId)
      if (!parallelBlockExecuted) {
        continue
      }

      // Get the parallel state
      const parallelState = context.parallelExecutions?.get(parallelId)
      if (!parallelState || parallelState.currentIteration === 0) {
        continue
      }

      // Check if all virtual blocks have been executed
      const allVirtualBlocksExecuted = this.areAllVirtualBlocksExecuted(
        parallelId,
        parallel,
        context.executedBlocks,
        parallelState,
        context
      )

      if (allVirtualBlocksExecuted && !context.completedLoops.has(parallelId)) {
        // Check if the parallel block already has aggregated results stored
        const blockState = context.blockStates.get(parallelId)
        if (blockState?.output?.completed && blockState?.output?.results) {
          logger.info(
            `Parallel ${parallelId} already has aggregated results, marking as completed without re-execution`
          )
          // Just mark it as completed without re-execution
          context.completedLoops.add(parallelId)

          // Activate the parallel-end-source connections if not already done
          const parallelEndConnections =
            context.workflow?.connections.filter(
              (conn) => conn.source === parallelId && conn.sourceHandle === 'parallel-end-source'
            ) || []

          for (const conn of parallelEndConnections) {
            if (!context.activeExecutionPath.has(conn.target)) {
              context.activeExecutionPath.add(conn.target)
              logger.info(`Activated post-parallel path to ${conn.target}`)
            }
          }

          continue
        }

        logger.info(
          `All virtual blocks completed for parallel ${parallelId}, re-executing to aggregate results`
        )

        // Re-execute the parallel block to check completion and trigger end connections
        context.executedBlocks.delete(parallelId)
        context.activeExecutionPath.add(parallelId)

        // IMPORTANT: Remove child nodes from active execution path to prevent re-execution
        for (const nodeId of parallel.nodes) {
          context.activeExecutionPath.delete(nodeId)
        }
      }
    }
  }
```

*   **Purpose**: This is the core method that periodically checks the state of all parallel blocks in the workflow. It's responsible for identifying when a parallel block's iterations are complete and then triggering the necessary actions to finalize its execution, aggregate results, and allow the workflow to proceed.
*   **Parameters**:
    *   `context`: The overall `ExecutionContext`.
*   **Line-by-line explanation**:
    *   `if (!this.parallels || Object.keys(this.parallels).length === 0) { return }`: If there are no parallel blocks configured in the workflow, exit early.
    *   `for (const [parallelId, parallel] of Object.entries(this.parallels)) { ... }`: Loop through each defined parallel block in the workflow.
    *   `if (context.completedLoops.has(parallelId)) { continue }`: If this parallel block has already been marked as fully completed (its loop is done), skip it.
    *   `const parallelBlockExecuted = context.executedBlocks.has(parallelId)`: Checks if the parallel block *itself* (not its virtual instances) has been executed at least once. This is important because the parallel block itself is responsible for initiating its virtual instances.
    *   `if (!parallelBlockExecuted) { continue }`: If the parallel block hasn't run yet, there's nothing to process for its iterations, so skip.
    *   `const parallelState = context.parallelExecutions?.get(parallelId)`: Retrieve the `ParallelState` for the current `parallelId` from the `ExecutionContext`.
    *   `if (!parallelState || parallelState.currentIteration === 0) { continue }`: If no parallel state exists or it's in an uninitialized state (`currentIteration` is 0), skip.
    *   `const allVirtualBlocksExecuted = this.areAllVirtualBlocksExecuted(...)`: Calls the previously explained method to check if all *required* virtual blocks for this parallel have completed.
    *   `if (allVirtualBlocksExecuted && !context.completedLoops.has(parallelId)) { ... }`: If all virtual blocks are executed *and* the parallel hasn't been marked as completed yet, then it's time to finalize.
        *   `const blockState = context.blockStates.get(parallelId)`: Retrieves the state of the parallel block itself.
        *   `if (blockState?.output?.completed && blockState?.output?.results) { ... }`: This is an optimization. If the parallel block *already* has its aggregated results stored in its output (meaning it was probably re-executed previously, or results were pre-calculated), then:
            *   `logger.info(...)`: Logs that it's already aggregated.
            *   `context.completedLoops.add(parallelId)`: Mark the parallel block as completed in the `ExecutionContext`.
            *   `const parallelEndConnections = ...`: Finds all connections that originate from the "parallel-end-source" handle of this parallel block. These are the connections that should become active once the parallel block finishes.
            *   `for (const conn of parallelEndConnections) { ... }`: For each of these "end" connections:
                *   `if (!context.activeExecutionPath.has(conn.target)) { ... }`: If the target of this connection is not already active:
                    *   `context.activeExecutionPath.add(conn.target)`: Add the target node to the active execution path, allowing the workflow to proceed past the parallel block.
                    *   `logger.info(...)`: Log the activation.
            *   `continue`: Move to the next parallel block as this one is handled.
        *   `logger.info(...)`: If not already aggregated, logs that the virtual blocks are complete and it's time to re-execute for aggregation.
        *   `context.executedBlocks.delete(parallelId)`: **Crucial step**: Removes the parallel block ID from `executedBlocks`. This signals to the execution engine that the parallel block needs to be executed *again*. This re-execution will be responsible for aggregating the results from all virtual blocks and then marking the parallel block as truly `completed`.
        *   `context.activeExecutionPath.add(parallelId)`: Adds the parallel block ID back to the `activeExecutionPath`, ensuring the execution engine picks it up for re-execution.
        *   `for (const nodeId of parallel.nodes) { ... }`: **IMPORTANT**: Iterates through all child nodes (blocks inside the parallel sub-workflow).
            *   `context.activeExecutionPath.delete(nodeId)`: Removes these child nodes from the active execution path. This prevents them from being re-executed unnecessarily during the aggregation phase, ensuring only the parent parallel block runs to finalize.

---

#### `createVirtualBlockInstances`

```typescript
  /**
   * Creates virtual block instances for parallel execution.
   */
  createVirtualBlockInstances(
    block: SerializedBlock,
    parallelId: string,
    parallelState: ParallelState,
    executedBlocks: Set<string>,
    activeExecutionPath: Set<string>
  ): string[] {
    const virtualBlockIds: string[] = []

    for (let i = 0; i < parallelState.parallelCount; i++) {
      const virtualBlockId = `${block.id}_parallel_${parallelId}_iteration_${i}`

      // Skip if this virtual instance was already executed
      if (executedBlocks.has(virtualBlockId)) {
        continue
      }

      // Check if this virtual instance is in the active path
      if (!activeExecutionPath.has(virtualBlockId) && !activeExecutionPath.has(block.id)) {
        continue
      }

      virtualBlockIds.push(virtualBlockId)
    }

    return virtualBlockIds
  }
```

*   **Purpose**: Given a `SerializedBlock` that is part of a parallel workflow, this method generates the unique IDs for its "virtual instances" across all iterations that are either active or not yet executed.
*   **Parameters**:
    *   `block`: The original `SerializedBlock` definition (e.g., a "transform data" block *inside* a parallel loop).
    *   `parallelId`: The ID of the parent parallel block.
    *   `parallelState`: The `ParallelState` of the parent parallel block.
    *   `executedBlocks`: `Set` of executed blocks from the `ExecutionContext`.
    *   `activeExecutionPath`: `Set` of active blocks from the `ExecutionContext`.
*   **Line-by-line explanation**:
    *   `const virtualBlockIds: string[] = []`: Initializes an empty array to store the generated virtual block IDs.
    *   `for (let i = 0; i < parallelState.parallelCount; i++) { ... }`: Loops from `0` up to `parallelCount - 1`, covering each iteration.
    *   `const virtualBlockId = \`${block.id}_parallel_${parallelId}_iteration_${i}\``: Constructs the unique ID for a virtual instance. This ID combines the original block's ID, the parent parallel block's ID, and the iteration index. This naming convention is key to tracking individual virtual blocks.
    *   `if (executedBlocks.has(virtualBlockId)) { continue }`: If this specific virtual instance has already completed execution, skip it.
    *   `if (!activeExecutionPath.has(virtualBlockId) && !activeExecutionPath.has(block.id)) { continue }`: This is a crucial check for determining which virtual blocks should be considered.
        *   `!activeExecutionPath.has(virtualBlockId)`: Checks if the specific virtual instance itself is not currently marked as active.
        *   `!activeExecutionPath.has(block.id)`: Checks if the *original* block (the template) is not marked as active.
        *   If *neither* the virtual instance *nor* the original block are in the active path, it means this virtual instance is not currently part of the execution flow and should be skipped for now. This helps prevent generating virtual instances for paths not meant to run yet.
    *   `virtualBlockIds.push(virtualBlockId)`: If the virtual instance passes the checks, add its ID to the list.
    *   `return virtualBlockIds`: Returns the array of unique IDs for the virtual block instances that need to be considered for execution.

---

#### `setupIterationContext`

```typescript
  /**
   * Sets up iteration-specific context for a virtual block.
   */
  setupIterationContext(
    context: ExecutionContext,
    parallelInfo: { parallelId: string; iterationIndex: number }
  ): void {
    const parallelState = context.parallelExecutions?.get(parallelInfo.parallelId)
    if (parallelState?.distributionItems) {
      const currentItem = this.getIterationItem(parallelState, parallelInfo.iterationIndex)

      // Store the current item for this specific iteration
      const iterationKey = `${parallelInfo.parallelId}_iteration_${parallelInfo.iterationIndex}`
      context.loopItems.set(iterationKey, currentItem)
      context.loopItems.set(parallelInfo.parallelId, currentItem) // Backward compatibility
      context.loopIterations.set(parallelInfo.parallelId, parallelInfo.iterationIndex)

      logger.info(`Set up iteration context for ${iterationKey} with item:`, currentItem)
    }
  }
```

*   **Purpose**: When a virtual block instance is about to execute, this method ensures that the `ExecutionContext` is updated with the correct iteration-specific data (the "distribution item") for that particular iteration. This allows blocks within the parallel sub-workflow to access the item they are currently processing.
*   **Parameters**:
    *   `context`: The overall `ExecutionContext`.
    *   `parallelInfo`: An object containing `parallelId` (the parent parallel block's ID) and `iterationIndex` (the current iteration's index).
*   **Line-by-line explanation**:
    *   `const parallelState = context.parallelExecutions?.get(parallelInfo.parallelId)`: Retrieves the `ParallelState` for the relevant parallel block.
    *   `if (parallelState?.distributionItems) { ... }`: Ensures there's a valid parallel state and distribution items exist.
    *   `const currentItem = this.getIterationItem(parallelState, parallelInfo.iterationIndex)`: Uses the helper method to fetch the specific distribution item for the `iterationIndex`.
    *   `const iterationKey = \`${parallelInfo.parallelId}_iteration_${parallelInfo.iterationIndex}\``: Creates a unique key for this specific iteration's context.
    *   `context.loopItems.set(iterationKey, currentItem)`: Stores the `currentItem` in the `context.loopItems` Map, keyed by the `iterationKey`. Blocks inside the parallel can now retrieve this item using this key.
    *   `context.loopItems.set(parallelInfo.parallelId, currentItem)`: Stores the `currentItem` using just the `parallelId` as the key. This is noted as "Backward compatibility," suggesting older parts of the system might expect to find the current item directly under the `parallelId`.
    *   `context.loopIterations.set(parallelInfo.parallelId, parallelInfo.iterationIndex)`: Stores the current `iterationIndex` under the `parallelId`. This allows blocks to know which iteration they are currently running.
    *   `logger.info(...)`: Logs that the iteration context has been set up, including the item being processed.

---

#### `storeIterationResult`

```typescript
  /**
   * Stores the result of a parallel iteration.
   */
  storeIterationResult(
    context: ExecutionContext,
    parallelId: string,
    iterationIndex: number,
    output: NormalizedBlockOutput
  ): void {
    const parallelState = context.parallelExecutions?.get(parallelId)
    if (parallelState) {
      const iterationKey = `iteration_${iterationIndex}`
      const existingResult = parallelState.executionResults.get(iterationKey)

      if (existingResult) {
        if (Array.isArray(existingResult)) {
          existingResult.push(output)
        } else {
          parallelState.executionResults.set(iterationKey, [existingResult, output])
        }
      } else {
        parallelState.executionResults.set(iterationKey, output)
      }
    }
  }
```

*   **Purpose**: To save the `output` (result) of a completed virtual block instance (i.e., a single iteration) into the `ParallelState` for later aggregation.
*   **Parameters**:
    *   `context`: The overall `ExecutionContext`.
    *   `parallelId`: The ID of the parent parallel block.
    *   `iterationIndex`: The index of the iteration that produced this output.
    *   `output`: The `NormalizedBlockOutput` produced by the virtual block instance.
*   **Line-by-line explanation**:
    *   `const parallelState = context.parallelExecutions?.get(parallelId)`: Retrieves the `ParallelState` for the parent parallel block.
    *   `if (parallelState) { ... }`: Ensures a valid parallel state exists.
    *   `const iterationKey = \`iteration_${iterationIndex}\``: Creates a key to store the result, typically just `iteration_0`, `iteration_1`, etc.
    *   `const existingResult = parallelState.executionResults.get(iterationKey)`: Checks if a result for this `iterationKey` already exists. This is important if a single iteration can produce multiple outputs, or if block outputs are appended.
    *   `if (existingResult) { ... }`: If there's an `existingResult`:
        *   `if (Array.isArray(existingResult)) { existingResult.push(output) }`: If the `existingResult` is already an array (meaning previous outputs for this iteration were stored as an array), push the new `output` to it.
        *   `else { parallelState.executionResults.set(iterationKey, [existingResult, output]) }`: If `existingResult` was not an array (e.g., a single output), convert it into an array containing the `existingResult` and the new `output`.
    *   `else { parallelState.executionResults.set(iterationKey, output) }`: If no `existingResult` was found, simply store the new `output` directly under the `iterationKey`.

---

### Conclusion

The `ParallelManager` is a sophisticated component that enables robust parallel processing within a workflow system. It handles the intricate details of:

1.  **Initialization**: Setting up the state for parallel execution based on distribution items.
2.  **Iteration Management**: Providing the correct data item for each specific iteration.
3.  **Completion Tracking**: Determining when all necessary parts of a parallel block's iterations have completed, even with conditional routing.
4.  **Contextualization**: Ensuring each virtual block instance operates with its specific iteration data.
5.  **Result Aggregation**: Collecting and organizing the outputs from all individual parallel iterations.

By orchestrating these steps, the `ParallelManager` allows workflow designers to define complex parallel loops that process data concurrently and then seamlessly integrate the aggregated results back into the main workflow flow.