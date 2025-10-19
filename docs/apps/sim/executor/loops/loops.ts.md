This TypeScript file defines the `LoopManager` class, a critical component in a workflow or execution engine responsible for orchestrating the execution of loops. It ensures that loops iterate correctly, respect defined limits, and properly reset their internal state between iterations.

---

## **`LoopManager.ts` - Detailed Explanation**

This file defines the `LoopManager` class, which is a core part of an execution engine, specifically designed to handle the complexities of running "loop" blocks within a workflow.

### **Purpose of this file**

The primary purpose of `LoopManager.ts` is to manage the lifecycle of loops in a workflow. This includes:

1.  **Detecting Loop Completion:** Determining when all necessary blocks *within* a single iteration of a loop have finished executing.
2.  **Managing Iterations:** Keeping track of the current iteration count for each loop and incrementing it.
3.  **Applying Iteration Limits:** Ensuring loops run for the specified number of times (or over the specified items for "forEach" loops).
4.  **Resetting Loop State:** Clearing the execution state of blocks *inside* a loop after an iteration so they can run again in the next iteration.
5.  **Aggregating Results:** Collecting the outputs of each iteration.
6.  **Activating Post-Loop Path:** Triggering the next blocks in the workflow once a loop has completed all its iterations.

In essence, `LoopManager` acts like a conductor for the loop sections of your workflow, ensuring each iteration plays out correctly before signaling the next, or finally ending the entire loop sequence.

---

### **Core Concepts**

Before diving into the code, let's clarify some key concepts:

*   **`ExecutionContext`**: This is a central object that holds the entire runtime state of the workflow execution. It contains information like which blocks have been executed, current loop iteration counts, decisions made by router/condition blocks, the workflow's structure (`blocks`, `connections`), and more.
*   **`SerializedLoop`**: This is the blueprint or configuration for a loop block in the workflow. It defines things like the loop's ID, its internal blocks (`nodes`), its type (`'for'` or `'forEach'`), and the number of iterations.
*   **`SerializedBlock` / `SerializedConnection`**: These represent the individual blocks and connections that make up the workflow's structure.
*   **Loop Block (`BlockType.LOOP`)**: In this system, a loop itself is represented as a special `BlockType.LOOP` block in the workflow graph. This block acts as both the entry point and exit point for the loop's execution.
*   **Inner Loop Blocks**: These are the regular workflow blocks that reside *inside* the loop container.

---

### **Simplifying Complex Logic**

The most complex logic in this file resides in `processLoopIterations` and `allReachableBlocksExecuted`. Let's simplify them:

1.  **`processLoopIterations` (The Loop Orchestrator):**
    *   **Simplified Goal:** After *every* batch of workflow blocks has run, this method checks if any *loop iteration* has completed, needs to run again, or is finally finished.
    *   **How it works:**
        *   It looks at *each* loop defined in the workflow.
        *   For a given loop, it first verifies that the loop's *entry point block* (the loop block itself) has already run in the current "layer" of execution.
        *   Then, it figures out *which blocks inside that loop were supposed to run in this iteration* (the "reachable" blocks, considering any conditional paths taken).
        *   If all those reachable blocks *have* run:
            *   It checks if the loop has hit its `maxIterations`.
            *   If **yes (finished)**: It gathers all the results from the different iterations, marks the loop as complete, and tells the workflow engine to activate the blocks *after* the loop.
            *   If **no (needs another iteration)**: It increments the loop's iteration counter, and *resets the execution state of all the blocks inside the loop* so they can run fresh in the next iteration. It also "un-executes" the loop block itself, so it can be re-triggered.

2.  **`allReachableBlocksExecuted` (The Internal Path Checker):**
    *   **Simplified Goal:** This helper function determines if *all the blocks that were actually part of the current path within a loop iteration* have successfully executed. It's crucial because not all blocks *visually inside* a loop might be executed in every iteration (e.g., if a conditional block skips a path).
    *   **How it works:**
        *   It starts by identifying the "entry points" into the loop's internal graph (blocks that receive connections from *outside* the loop).
        *   It then performs a "walk-through" (a graph traversal) of the loop's internal connections, *but it's smart about it*:
            *   If it encounters a `ROUTER` or `CONDITION` block, it only follows the *specific path that was chosen* during the current iteration (based on decisions stored in the `context`). It doesn't follow all possible paths.
            *   It also considers error paths (`sourceHandle === 'error'`) if a block had an error.
        *   It collects a list of *all the blocks that were actually reachable* based on the chosen paths.
        *   Finally, it checks if *every single one* of these reachable blocks has a record in `context.executedBlocks` (meaning they finished running). If even one is missing, the iteration isn't complete.

---

### **Line-by-Line Explanation**

```typescript
import { createLogger } from '@/lib/logs/console/logger'
import { BlockType } from '@/executor/consts'
import type { ExecutionContext } from '@/executor/types'
import { ConnectionUtils } from '@/executor/utils/connections'
import type { SerializedBlock, SerializedConnection, SerializedLoop } from '@/serializer/types'

// Initializes a logger specifically for 'LoopManager' to output informative messages
// during execution, useful for debugging and monitoring.
const logger = createLogger('LoopManager')

/**
 * Manages loop detection, iteration limits, and state resets.
 * With the new loop block approach, this class is significantly simplified.
 */
export class LoopManager {
  // The constructor is called when a new LoopManager instance is created.
  constructor(
    // private loops: A private property holding a map (key-value store) of all defined loops in the workflow.
    // The key is the loop's ID, and the value is its SerializedLoop configuration.
    private loops: Record<string, SerializedLoop>,
    // private defaultIterations: A private property that sets the default number of iterations
    // if a loop's configuration doesn't specify one. Defaults to 5.
    private defaultIterations = 5
  ) {}

  /**
   * Processes all loops and checks if any need to be iterated.
   * This is called after each execution layer to handle loop iterations.
   *
   * @param context - Current execution context
   * @returns Whether any loop has reached its maximum iterations
   */
  async processLoopIterations(context: ExecutionContext): Promise<boolean> {
    // hasLoopReachedMaxIterations: A flag initialized to false. It will be set to true
    // if any loop finishes all its iterations during this processing cycle.
    let hasLoopReachedMaxIterations = false

    // Nothing to do if no loops
    // An early exit condition: if there are no loops defined in the workflow,
    // there's nothing for the manager to do, so it immediately returns `false`.
    if (Object.keys(this.loops).length === 0) return hasLoopReachedMaxIterations

    // Check each loop to see if it should iterate
    // Iterates over each loop defined in the `this.loops` map.
    // `loopId` is the ID of the loop, `loop` is its configuration.
    for (const [loopId, loop] of Object.entries(this.loops)) {
      // Skip if this loop has already been marked as completed
      // Checks if the current loop has already finished all its iterations in a previous cycle.
      // If so, it skips to the next loop.
      if (context.completedLoops.has(loopId)) {
        continue
      }

      // Check if the loop block itself has been executed
      // Verifies if the main loop block (the entry point for the loop) has been executed
      // in the current execution layer.
      const loopBlockExecuted = context.executedBlocks.has(loopId)
      if (!loopBlockExecuted) {
        // Loop block hasn't been executed yet, skip processing
        // If the loop block hasn't run yet, it means the workflow hasn't even entered this loop's
        // current iteration, so we skip processing for this loop.
        continue
      }

      // Check if all blocks in the loop have been executed
      // Calls a private helper method to determine if all *reachable* blocks within the
      // current loop iteration have completed their execution. This is a crucial check.
      const allBlocksInLoopExecuted = this.allBlocksExecuted(loop.nodes, context)

      // If all reachable blocks within the current iteration have finished:
      if (allBlocksInLoopExecuted) {
        // All blocks in the loop have been executed
        // Retrieves the current iteration count for this loop from the execution context,
        // defaulting to 1 if it's the very first iteration.
        const currentIteration = context.loopIterations.get(loopId) || 1

        // Results are now stored individually as blocks execute (like parallels)
        // No need for bulk collection here
        // (This comment indicates a change in how results are collected, suggesting
        // individual blocks or a separate mechanism handles this now.)

        // The loop block will handle incrementing the iteration when it executes next
        // We just need to reset the blocks so they can run again
        // (This comment suggests that while the LoopManager handles resets, the actual
        // incrementing of context.loopIterations might be done by the loop block's executor).

        // Determine the maximum iterations
        // Initializes `maxIterations` with the value from the loop config or the default.
        let maxIterations = loop.iterations || this.defaultIterations

        // For forEach loops, use the actual items length
        // Special handling for 'forEach' loop types.
        if (loop.loopType === 'forEach' && loop.forEachItems) {
          // First check if the items have already been evaluated and stored by the loop handler
          // If the items to iterate over have already been processed and stored in the context, use those.
          const storedItems = context.loopItems.get(`${loopId}_items`)
          if (storedItems) {
            // Determines the length of the stored items (array length or object key count).
            const itemsLength = Array.isArray(storedItems)
              ? storedItems.length
              : Object.keys(storedItems).length

            // Updates maxIterations to match the number of items.
            maxIterations = itemsLength
            logger.info(
              `forEach loop ${loopId} - Items: ${itemsLength}, Max iterations: ${maxIterations}`
            )
          } else {
            // If items not stored, tries to parse the `forEachItems` from the loop configuration.
            const itemsLength = this.getItemsLength(loop.forEachItems)
            if (itemsLength > 0) {
              // Updates maxIterations if items were successfully parsed.
              maxIterations = itemsLength
              logger.info(
                `forEach loop ${loopId} - Parsed items: ${itemsLength}, Max iterations: ${maxIterations}`
              )
            }
          }
        }

        // Logs the current and maximum iteration numbers for the loop.
        logger.info(`Loop ${loopId} - Current: ${currentIteration}, Max: ${maxIterations}`)

        // Check if we've completed all iterations
        // Compares the current iteration with the maximum allowed iterations.
        if (currentIteration >= maxIterations) {
          // If current iteration meets or exceeds max iterations, the loop is finished.
          hasLoopReachedMaxIterations = true
          logger.info(`Loop ${loopId} has completed all ${maxIterations} iterations`)

          // Collect results for the completed loop.
          const results = []
          // Retrieves the execution state for this loop (which contains results per iteration).
          const loopState = context.loopExecutions?.get(loopId)
          if (loopState) {
            // Iterates through all completed iterations to collect their results.
            for (let i = 0; i < maxIterations; i++) {
              const result = loopState.executionResults.get(`iteration_${i}`)
              if (result) {
                results.push(result)
              }
            }
          }

          // Creates an aggregated output object for the completed loop.
          const aggregatedOutput = {
            loopId,
            currentIteration: maxIterations - 1, // Last iteration index
            maxIterations,
            loopType: loop.loopType || 'for',
            completed: true,
            results,
            message: `Completed all ${maxIterations} iterations`,
          }

          // Stores the aggregated output as the final state for the loop block itself.
          context.blockStates.set(loopId, {
            output: aggregatedOutput,
            executed: true,
            executionTime: 0,
          })

          // Marks this loop as fully completed in the execution context.
          context.completedLoops.add(loopId)

          // Finds connections that originate from the 'loop-end-source' handle of the loop block.
          // These connections define what happens *after* the loop completes.
          const loopEndConnections =
            context.workflow?.connections.filter(
              (conn) => conn.source === loopId && conn.sourceHandle === 'loop-end-source'
            ) || []

          // For each connection found, it adds the target block to the `activeExecutionPath`,
          // effectively activating the next steps in the workflow after the loop.
          for (const conn of loopEndConnections) {
            context.activeExecutionPath.add(conn.target)
            logger.info(`Activated post-loop path from ${loopId} to ${conn.target}`)
          }

          // Logs that the loop has completed and post-loop connections are activated.
          logger.info(`Loop ${loopId} - Completed and activated end connections`)
        } else {
          // If the loop has not completed all iterations:
          // Increments the loop's iteration counter in the execution context.
          context.loopIterations.set(loopId, currentIteration + 1)
          logger.info(`Loop ${loopId} - Incremented counter to ${currentIteration + 1}`)

          // Calls a private helper method to reset the execution state of all blocks *inside* the loop.
          // This prepares them to run again in the next iteration.
          this.resetLoopBlocks(loopId, loop, context)

          // Clears the loop block itself from `executedBlocks` and `blockStates`.
          // This allows the loop block to be considered 'unexecuted' for the next iteration,
          // so it can be re-triggered by the workflow engine to start the next cycle.
          context.executedBlocks.delete(loopId)
          context.blockStates.delete(loopId)

          logger.info(`Loop ${loopId} - Reset for iteration ${currentIteration + 1}`)
        }
      }
    }

    // Returns true if any loop completed all its iterations, false otherwise.
    return hasLoopReachedMaxIterations
  }

  /**
   * Checks if all reachable blocks in a loop have been executed.
   * This method now excludes completely unconnected blocks from consideration,
   * ensuring they don't prevent loop completion.
   *
   * @param nodeIds - All node IDs in the loop
   * @param context - Execution context
   * @returns Whether all reachable blocks have been executed
   */
  private allBlocksExecuted(nodeIds: string[], context: ExecutionContext): boolean {
    // This is a wrapper method, simply delegating to the more specific `allReachableBlocksExecuted`.
    // This structure allows for clearer separation and potential future modifications or testing.
    return this.allReachableBlocksExecuted(nodeIds, context)
  }

  /**
   * Helper method to check if all reachable blocks have been executed.
   * Separated for clarity and potential future testing.
   */
  private allReachableBlocksExecuted(nodeIds: string[], context: ExecutionContext): boolean {
    // Get all connections within the loop
    // Filters all workflow connections to find only those where both source and target
    // blocks are within the current loop's `nodeIds`.
    const loopConnections =
      context.workflow?.connections.filter(
        (conn) => nodeIds.includes(conn.source) && nodeIds.includes(conn.target)
      ) || []

    // Build a map of blocks to their outgoing connections within the loop
    // Creates a map where each key is a block ID within the loop, and its value is an array
    // of connections originating from that block *and staying within the loop*.
    const blockOutgoingConnections = new Map<string, typeof loopConnections>()
    for (const nodeId of nodeIds) {
      const outgoingConnections = ConnectionUtils.getOutgoingConnections(nodeId, loopConnections)
      blockOutgoingConnections.set(nodeId, outgoingConnections)
    }

    // Find blocks that have no incoming connections within the loop (entry points)
    // Only consider blocks as entry points if they have external connections to the loop
    // Identifies "entry points" into the internal graph of the loop. These are blocks
    // that receive a connection from *outside* the loop, or are simply the first in a disconnected subgraph.
    const entryBlocks = nodeIds.filter((nodeId) =>
      ConnectionUtils.isEntryPoint(nodeId, nodeIds, context.workflow?.connections || [])
    )

    // Track which blocks we've visited and determined are reachable
    // `reachableBlocks`: A set to store the IDs of all blocks that are reachable from the `entryBlocks`.
    const reachableBlocks = new Set<string>()
    // `toVisit`: A queue (array used as a queue) for a Breadth-First Search (BFS) or Depth-First Search (DFS)
    // traversal, initialized with the `entryBlocks`.
    const toVisit = [...entryBlocks]

    // Traverse the graph to find all reachable blocks
    // Loops as long as there are blocks in the `toVisit` queue.
    while (toVisit.length > 0) {
      // Dequeues the next block to visit. `!` asserts it's not undefined.
      const currentBlockId = toVisit.shift()!

      // Skip if already visited
      // If this block has already been added to `reachableBlocks`, skip to prevent infinite loops and redundant work.
      if (reachableBlocks.has(currentBlockId)) continue

      // Marks the current block as reachable.
      reachableBlocks.add(currentBlockId)

      // Get the block
      // Retrieves the full block object from the workflow context.
      const block = context.workflow?.blocks.find((b) => b.id === currentBlockId)
      if (!block) continue // If block not found, skip.

      // Get outgoing connections from this block
      // Retrieves the pre-calculated outgoing connections for the current block *within the loop*.
      const outgoing = blockOutgoingConnections.get(currentBlockId) || []

      // Handle routing blocks specially
      // If the current block is a `ROUTER` block:
      if (block.metadata?.id === BlockType.ROUTER) {
        // For router blocks, only follow the selected path
        // Gets the specific target block that the router decided to send execution to.
        const selectedTarget = context.decisions.router.get(currentBlockId)
        // If a target was selected and is part of the loop, add it to `toVisit`.
        if (selectedTarget && nodeIds.includes(selectedTarget)) {
          toVisit.push(selectedTarget)
        }
      } else if (block.metadata?.id === BlockType.CONDITION) {
        // For condition blocks, only follow the selected condition path
        // If the current block is a `CONDITION` block:
        // Gets the specific condition ID that was met.
        const selectedConditionId = context.decisions.condition.get(currentBlockId)
        if (selectedConditionId) {
          // Finds the outgoing connection that corresponds to the selected condition.
          const selectedConnection = outgoing.find(
            (conn) => conn.sourceHandle === `condition-${selectedConditionId}`
          )
          // If a matching connection and target exist, add the target to `toVisit`.
          if (selectedConnection?.target) {
            toVisit.push(selectedConnection.target)
          }
        }
      } else {
        // For regular blocks, use the extracted error handling method
        // For all other types of blocks, use a generic helper method to process their outgoing connections,
        // specifically considering error paths.
        this.handleErrorConnections(currentBlockId, outgoing, context, toVisit)
      }
    }

    // Now check if all reachable blocks have been executed
    // After identifying all `reachableBlocks`, this loop checks each one.
    for (const reachableBlockId of reachableBlocks) {
      // If any reachable block has *not* been executed (i.e., not in `context.executedBlocks`),
      // then the current loop iteration is not complete.
      if (!context.executedBlocks.has(reachableBlockId)) {
        logger.info(
          `Loop iteration not complete - block ${reachableBlockId} is reachable but not executed`
        )
        return false // Returns false immediately if an unexecuted reachable block is found.
      }
    }

    // If the loop completes, it means all reachable blocks have been executed.
    logger.info(
      `All reachable blocks in loop have been executed. Reachable: ${Array.from(reachableBlocks).join(', ')}`
    )
    return true // Returns true, indicating the iteration is complete.
  }

  /**
   * Helper to get the length of items for forEach loops
   */
  private getItemsLength(forEachItems: any): number {
    // Checks if `forEachItems` is an array and returns its length.
    if (Array.isArray(forEachItems)) {
      return forEachItems.length
    }
    // Checks if `forEachItems` is a non-null object and returns the number of its keys.
    if (typeof forEachItems === 'object' && forEachItems !== null) {
      return Object.keys(forEachItems).length
    }
    // If `forEachItems` is a string, it attempts to parse it as JSON.
    if (typeof forEachItems === 'string') {
      try {
        const parsed = JSON.parse(forEachItems)
        // If parsed as an array, returns its length.
        if (Array.isArray(parsed)) {
          return parsed.length
        }
        // If parsed as an object, returns the number of its keys.
        if (typeof parsed === 'object' && parsed !== null) {
          return Object.keys(parsed).length
        }
      } catch {} // Catches JSON parsing errors and ignores them.
    }
    // If none of the above conditions are met, returns 0.
    return 0
  }

  /**
   * Resets all blocks within a loop for the next iteration.
   *
   * @param loopId - ID of the loop
   * @param loop - The loop configuration
   * @param context - Current execution context
   */
  private resetLoopBlocks(loopId: string, loop: SerializedLoop, context: ExecutionContext): void {
    // Reset all blocks in the loop
    // Iterates through all block IDs (`nodeId`) that are part of the current loop.
    for (const nodeId of loop.nodes) {
      // Removes the block from `executedBlocks` so it can be executed again.
      context.executedBlocks.delete(nodeId)

      // Clears any stored state for the block, preparing it for a fresh run.
      context.blockStates.delete(nodeId)

      // Ensures the block is not considered part of the active execution path for the *next* iteration,
      // as paths need to be re-evaluated.
      context.activeExecutionPath.delete(nodeId)

      // Clears any specific decisions (e.g., for routers or conditions) made during the previous iteration
      // for this block, as new decisions will be made in the next iteration.
      context.decisions.router.delete(nodeId)
      context.decisions.condition.delete(nodeId)
    }
  }

  /**
   * Stores the result of a loop iteration.
   */
  storeIterationResult(
    context: ExecutionContext,
    loopId: string,
    iterationIndex: number,
    output: any
  ): void {
    // Initializes `context.loopExecutions` as a new Map if it doesn't already exist.
    if (!context.loopExecutions) {
      context.loopExecutions = new Map()
    }

    // Retrieves the execution state for the current loop.
    let loopState = context.loopExecutions.get(loopId)
    // If no state exists for this loop, initializes it.
    if (!loopState) {
      const loop = this.loops[loopId]
      const loopType = loop?.loopType === 'forEach' ? 'forEach' : 'for'
      const forEachItems = loop?.forEachItems

      // Creates a new loop state object with initial values.
      loopState = {
        maxIterations: loop?.iterations || this.defaultIterations,
        loopType,
        forEachItems:
          Array.isArray(forEachItems) || (typeof forEachItems === 'object' && forEachItems !== null)
            ? forEachItems
            : null,
        executionResults: new Map(), // Map to store results of each iteration
        currentIteration: 0,
      }
      context.loopExecutions.set(loopId, loopState) // Stores the new loop state in the context.
    }

    // Creates a unique key for the current iteration's results.
    const iterationKey = `iteration_${iterationIndex}`
    // Gets any existing result for this iteration.
    const existingResult = loopState.executionResults.get(iterationKey)

    // If results already exist for this iteration:
    if (existingResult) {
      // If the existing result is an array, push the new output to it.
      if (Array.isArray(existingResult)) {
        existingResult.push(output)
      } else {
        // If it's not an array (meaning it's a single result), convert it to an array
        // and add both the old and new results. This handles parallel paths within an iteration.
        loopState.executionResults.set(iterationKey, [existingResult, output])
      }
    } else {
      // If no result exists yet for this iteration, simply store the new output.
      loopState.executionResults.set(iterationKey, output)
    }
  }

  /**
   * Gets the correct loop index based on the current block being executed.
   *
   * @param loopId - ID of the loop
   * @param blockId - ID of the block requesting the index
   * @param context - Current execution context
   * @returns The correct loop index for this block
   */
  getLoopIndex(loopId: string, blockId: string, context: ExecutionContext): number {
    // Retrieves the loop configuration.
    const loop = this.loops[loopId]
    if (!loop) return 0 // If loop not found, returns 0.

    // Return the current iteration counter
    // Returns the current iteration count for the specified loop, defaulting to 0 if not set.
    return context.loopIterations.get(loopId) || 0
  }

  /**
   * Gets the iterations for a loop.
   *
   * @param loopId - ID of the loop
   * @returns Iterations for the loop
   */
  getIterations(loopId: string): number {
    // Returns the defined number of iterations for a loop, or the default if not specified.
    return this.loops[loopId]?.iterations || this.defaultIterations
  }

  /**
   * Gets the current item for a forEach loop.
   *
   * @param loopId - ID of the loop
   * @param context - Current execution context
   * @returns Current item in the loop iteration
   */
  getCurrentItem(loopId: string, context: ExecutionContext): any {
    // Retrieves the current item being processed in a forEach loop from the execution context.
    return context.loopItems.get(loopId)
  }

  /**
   * Checks if a connection forms a feedback path in a loop.
   * With loop blocks, feedback paths are now handled by loop-to-inner-block connections.
   *
   * @param connection - Connection to check
   * @param blocks - All blocks in the workflow
   * @returns Whether the connection forms a feedback path
   */
  isFeedbackPath(connection: SerializedConnection, blocks: SerializedBlock[]): boolean {
    // With the new loop block approach, feedback paths are connections from
    // blocks inside the loop back to the loop block itself
    // Iterates through all defined loops.
    for (const [loopId, loop] of Object.entries(this.loops)) {
      // Use Set for O(1) lookup performance instead of O(n) includes()
      // Creates a Set of node IDs within the loop for efficient lookup.
      const loopNodesSet = new Set(loop.nodes)

      // Check if source is inside the loop and target is the loop block
      // A connection is a feedback path if its source block is inside the loop
      // AND its target block is the loop block itself.
      if (loopNodesSet.has(connection.source) && connection.target === loopId) {
        return true // Found a feedback path.
      }
    }

    return false // No feedback path found.
  }

  /**
   * Handles error connections and follows appropriate paths based on error state.
   *
   * @param blockId - ID of the block to check for error handling
   * @param outgoing - Outgoing connections from the block
   * @param context - Current execution context
   * @param toVisit - Array to add next blocks to visit
   */
  private handleErrorConnections(
    blockId: string,
    outgoing: any[], // An array of SerializedConnection, but typed as any[] for flexibility.
    context: ExecutionContext,
    toVisit: string[] // Blocks to be visited next in the graph traversal.
  ): void {
    // For regular blocks, check if they had an error
    // Retrieves the execution state for the current block.
    const blockState = context.blockStates.get(blockId)
    // Checks if the block's output contains an `error` property.
    const hasError = blockState?.output?.error !== undefined

    // Follow appropriate connections based on error state
    // Iterates through all outgoing connections from the current block.
    for (const conn of outgoing) {
      // If the connection is an 'error' handle AND the block had an error,
      // add the connection's target block to the `toVisit` queue.
      if (conn.sourceHandle === 'error' && hasError) {
        toVisit.push(conn.target)
      } else if (
        // If the connection is a 'source' handle (default success path) or has no handle
        // AND the block did NOT have an error,
        // add the connection's target block to the `toVisit` queue.
        (conn.sourceHandle === 'source' || !conn.sourceHandle) &&
        !hasError
      ) {
        toVisit.push(conn.target)
      }
    }
  }
}
```