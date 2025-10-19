This TypeScript file defines the `Executor` class, the core engine responsible for running workflows. It meticulously orchestrates the execution of various "blocks" (individual steps in a workflow, like API calls, conditions, loops, etc.) in the correct order, manages their inputs and outputs, handles errors, and supports features like streaming, debugging, and parallel/loop execution.

## 1. Overview: The Workflow Engine

At its heart, the `Executor` class is a state machine that traverses a workflow graph (a collection of blocks and connections) and executes each block. It ensures that blocks only run when their dependencies are met, manages complex control flow structures like loops and parallel branches, and provides mechanisms for observability (logging, telemetry, debug mode) and interaction (streaming outputs, cancellation).

Think of it as the conductor of an orchestra: it knows which instrument (block) plays when, passes notes (inputs) between them, listens to the sound (outputs), and handles any mistakes (errors) or special sections (loops, parallels).

## 2. Imports and Setup

This section imports all the necessary components for the `Executor` to function.

```typescript
// Imports from various modules within the application
import { BlockPathCalculator } from '@/lib/block-path-calculator' // Utility to calculate block accessibility
import { createLogger } from '@/lib/logs/console/logger' // Logger for console output
import type { TraceSpan } from '@/lib/logs/types' // Type definition for tracing spans (used for child workflows)
import { getBlock } from '@/blocks' // Function to retrieve block metadata by ID
import type { BlockOutput } from '@/blocks/types' // Type definition for a block's output
import { BlockType, isWorkflowBlockType } from '@/executor/consts' // Constants and type guards for block types
import {
  AgentBlockHandler,
  ApiBlockHandler,
  ConditionBlockHandler,
  EvaluatorBlockHandler,
  FunctionBlockHandler,
  GenericBlockHandler,
  LoopBlockHandler,
  ParallelBlockHandler,
  ResponseBlockHandler,
  RouterBlockHandler,
  TriggerBlockHandler,
  WorkflowBlockHandler,
} from '@/executor/handlers' // Individual handlers for different block types
import { LoopManager } from '@/executor/loops/loops' // Manages the state and execution of loop blocks
import { ParallelManager } from '@/executor/parallels/parallels' // Manages the state and execution of parallel blocks
import { ParallelRoutingUtils } from '@/executor/parallels/utils' // Utilities for routing within parallel blocks
import { PathTracker } from '@/executor/path/path' // Tracks the active execution path in the workflow
import { InputResolver } from '@/executor/resolver/resolver' // Resolves input values for blocks
import type {
  BlockHandler,
  BlockLog,
  ExecutionContext,
  ExecutionResult,
  NormalizedBlockOutput,
  StreamingExecution,
}
// Type definitions for various execution-related entities
from '@/executor/types'
import { streamingResponseFormatProcessor } from '@/executor/utils' // Utility for processing streaming responses
import { VirtualBlockUtils } from '@/executor/utils/virtual-blocks' // Utilities for managing "virtual" blocks (e.g., in parallels)
import type { SerializedBlock, SerializedWorkflow } from '@/serializer/types' // Type definitions for workflow and block serialization
import { useExecutionStore } from '@/stores/execution/store' // Zustand store for execution-related UI state
import { useConsoleStore } from '@/stores/panel/console/store' // Zustand store for console panel logs
import { useGeneralStore } from '@/stores/settings/general/store' // Zustand store for general application settings (e.g., debug mode)

const logger = createLogger('Executor') // Initializes a logger instance specifically for the Executor, named 'Executor'.

// This declares global properties on the `window` object, typically for client-side telemetry.
// `__SIM_TELEMETRY_ENABLED`: A boolean indicating if telemetry is active.
// `__SIM_TRACK_EVENT`: A function to track custom events, with an event name and properties.
declare global {
  interface Window {
    __SIM_TELEMETRY_ENABLED?: boolean
    __SIM_TRACK_EVENT?: (eventName: string, properties?: Record<string, any>) => void
  }
}
```

**Simplified Explanation of Imports:**

*   **Workflow Structure & Logic:** Imports related to how blocks are connected (`BlockPathCalculator`), the definition of a workflow (`SerializedWorkflow`), and types for block inputs/outputs (`BlockOutput`).
*   **Execution Components:** Imports for specialized `BlockHandler` classes (one for each type of block like `ApiBlockHandler`, `LoopBlockHandler`), managers for complex patterns like loops and parallels (`LoopManager`, `ParallelManager`), and utilities for input resolution (`InputResolver`) and path tracking (`PathTracker`).
*   **Logging & UI State:** Imports for logging (`createLogger`, `BlockLog`), and Zustand stores (`useExecutionStore`, `useConsoleStore`, `useGeneralStore`) to manage UI state during execution (e.g., active blocks, console output, debug mode).
*   **Types:** Numerous `type` imports ensure strong typing throughout the executor, defining the structure of execution context, results, and other data.
*   **Telemetry:** A `declare global` block exposes a mechanism for tracking workflow execution events in a browser environment, likely for analytics or debugging.

## 3. `trackWorkflowTelemetry` Function

```typescript
/**
 * Tracks telemetry events for workflow execution if telemetry is enabled
 */
function trackWorkflowTelemetry(eventName: string, data: Record<string, any>) {
  // Checks if running in a browser environment and if the telemetry tracking function is available
  if (typeof window !== 'undefined' && window.__SIM_TRACK_EVENT) {
    // Adds a timestamp to the event data and spreads existing data.
    // This creates a shallow copy to prevent potential circular references from `data`.
    const safeData = {
      ...data,
      timestamp: Date.now(),
    }

    // Calls the global telemetry tracking function, categorizing the event as 'workflow'.
    window.__SIM_TRACK_EVENT(eventName, {
      category: 'workflow',
      ...safeData,
    })
  }
}
```

**Simplified Explanation:**

This function is a wrapper for a global telemetry system. If the application is running in a browser and a specific tracking function (`window.__SIM_TRACK_EVENT`) is available, it sends workflow-related events (like "execution started," "block completed") to that system. It also automatically adds a timestamp and categorizes the event as a "workflow" event.

## 4. `Executor` Class

This is the main class where all the execution logic resides.

```typescript
/**
 * Core execution engine that runs workflow blocks in topological order.
 *
 * Handles block execution, state management, and error handling.
 */
export class Executor {
  // Core components are initialized once and remain immutable
  private resolver: InputResolver // Resolves block inputs by looking up values from other blocks or environment variables.
  private loopManager: LoopManager // Manages the state and progress of loop blocks.
  private parallelManager: ParallelManager // Manages the state and progress of parallel blocks.
  private pathTracker: PathTracker // Keeps track of the active execution paths (e.g., after a condition or router block).
  private blockHandlers: BlockHandler[] // A list of specialized handlers, each capable of executing a specific type of block.
  private workflowInput: any // The initial input provided to the workflow.
  private isDebugging = false // Flag indicating if the workflow is currently running in debug mode.
  private contextExtensions: any = {} // Additional context passed from the caller, e.g., for streaming or specific outputs.
  private actualWorkflow: SerializedWorkflow // The workflow definition (blocks, connections, loops, parallels).
  private isCancelled = false // Flag indicating if the workflow execution has been cancelled.
  private isChildExecution = false // Flag indicating if this is an execution of a child workflow within a parent.

  /**
   * Updates block output with streamed content, handling both structured and unstructured responses
   */
  private updateBlockOutputWithStreamedContent(
    blockId: string, // The ID of the block whose output is being updated.
    fullContent: string, // The complete streamed content received so far.
    blockState: any, // The current state object for the block.
    context: ExecutionContext // The overall execution context.
  ): void {
    if (!blockState?.output) return // If the block state doesn't have an output, there's nothing to update.

    // Check if we have response format - if so, preserve structured response
    let responseFormat: any
    // Tries to get the response format from the initial block states, which are passed during setup.
    if (this.initialBlockStates?.[blockId]) {
      const initialBlockState = this.initialBlockStates[blockId] as any
      responseFormat = initialBlockState.responseFormat
    }

    // If a response format is defined AND we have content, try to parse it as JSON.
    if (responseFormat && fullContent) {
      try {
        const parsedContent = JSON.parse(fullContent) // Attempt to parse the streamed content as JSON.
        // If successful, create a structured output by merging parsed content with existing metadata (tokens, toolCalls, etc.).
        const structuredOutput = {
          ...parsedContent,
          tokens: blockState.output.tokens,
          toolCalls: blockState.output.toolCalls,
          providerTiming: blockState.output.providerTiming,
          cost: blockState.output.cost,
        }
        blockState.output = structuredOutput // Update the block's output with the structured content.

        // Also update the corresponding block log entry if found.
        const blockLog = context.blockLogs.find((log) => log.blockId === blockId)
        if (blockLog) {
          blockLog.output = structuredOutput
        }
      } catch (parseError) {
        // If JSON parsing fails (e.g., incomplete or invalid JSON), fall back to treating content as plain text.
        blockState.output.content = fullContent
      }
    } else {
      // If no response format or no content, simply assign the full content to the 'content' property of the output.
      blockState.output.content = fullContent
    }
  }

  constructor(
    private workflowParam:
      | SerializedWorkflow // The workflow definition itself.
      | {
          workflow: SerializedWorkflow // The workflow definition (alternative constructor format).
          currentBlockStates?: Record<string, BlockOutput> // Initial states/outputs for some blocks.
          envVarValues?: Record<string, string> // Environment variable values.
          workflowInput?: any // Initial input for the workflow.
          workflowVariables?: Record<string, any> // Workflow-level variables.
          contextExtensions?: {
            stream?: boolean // Whether to enable streaming.
            selectedOutputs?: string[] // Specific outputs to stream.
            edges?: Array<{ source: string; target: string }> // Connections between blocks.
            onStream?: (streamingExecution: StreamingExecution) => Promise<void> // Callback for streamed outputs.
            onBlockComplete?: (blockId: string, output: any) => Promise<void> // Callback when a block completes.
            executionId?: string // Unique ID for the current execution.
            workspaceId?: string // ID of the workspace.
            isChildExecution?: boolean // Flag if this is a child workflow execution.
            // Marks executions that must use deployed constraints (API/webhook/schedule/chat)
            isDeployedContext?: boolean
          }
        },
    private initialBlockStates: Record<string, BlockOutput> = {}, // Default empty initial block states.
    private environmentVariables: Record<string, string> = {}, // Default empty environment variables.
    workflowInput?: any, // Default empty workflow input.
    private workflowVariables: Record<string, any> = {} // Default empty workflow variables.
  ) {
    // Determine which constructor format was used.
    if (typeof workflowParam === 'object' && 'workflow' in workflowParam) {
      // New format: options object
      const options = workflowParam
      this.actualWorkflow = options.workflow
      this.initialBlockStates = options.currentBlockStates || {}
      this.environmentVariables = options.envVarValues || {}
      this.workflowInput = options.workflowInput || {}
      this.workflowVariables = options.workflowVariables || {}

      // Store context extensions for streaming and output selection
      if (options.contextExtensions) {
        this.contextExtensions = options.contextExtensions
        this.isChildExecution = options.contextExtensions.isChildExecution || false
      }
    } else {
      // Old format: direct parameters
      this.actualWorkflow = workflowParam

      if (workflowInput) {
        this.workflowInput = workflowInput
      } else {
        this.workflowInput = {}
      }
    }

    this.validateWorkflow() // Perform initial validation of the workflow structure.

    this.loopManager = new LoopManager(this.actualWorkflow.loops || {}) // Initialize loop manager with workflow's loop definitions.
    this.parallelManager = new ParallelManager(this.actualWorkflow.parallels || {}) // Initialize parallel manager with workflow's parallel definitions.

    // Calculate accessible blocks for consistent reference resolution
    // This pre-calculates which blocks can "see" the outputs of other blocks, important for input resolution.
    const accessibleBlocksMap = BlockPathCalculator.calculateAccessibleBlocksForWorkflow(
      this.actualWorkflow
    )

    this.resolver = new InputResolver(
      this.actualWorkflow,
      this.environmentVariables,
      this.workflowVariables,
      this.loopManager,
      accessibleBlocksMap
    ) // Initialize input resolver.
    this.pathTracker = new PathTracker(this.actualWorkflow) // Initialize path tracker.

    // Instantiate all concrete block handlers. The Executor will use these to execute different block types.
    this.blockHandlers = [
      new TriggerBlockHandler(),
      new AgentBlockHandler(),
      new RouterBlockHandler(this.pathTracker),
      new ConditionBlockHandler(this.pathTracker, this.resolver),
      new EvaluatorBlockHandler(),
      new FunctionBlockHandler(),
      new ApiBlockHandler(),
      new LoopBlockHandler(this.resolver, this.pathTracker),
      new ParallelBlockHandler(this.resolver, this.pathTracker),
      new ResponseBlockHandler(),
      new WorkflowBlockHandler(),
      new GenericBlockHandler(), // A fallback handler for blocks without a specific handler.
    ]

    // Get the debug mode state from the general settings store.
    this.isDebugging = useGeneralStore.getState().isDebugModeEnabled
  }

  /**
   * Cancels the current workflow execution.
   * Sets the cancellation flag to stop further execution.
   */
  public cancel(): void {
    logger.info('Workflow execution cancelled') // Logs a cancellation message.
    this.isCancelled = true // Sets a flag to true, which the main execution loop checks.
  }

  /**
   * Executes the workflow and returns the result.
   *
   * @param workflowId - Unique identifier for the workflow execution
   * @param startBlockId - Optional block ID to start execution from (for webhook or schedule triggers)
   * @returns Execution result containing output, logs, and metadata, or a stream, or combined execution and stream
   */
  async execute(
    workflowId: string, // A unique identifier for this specific workflow run.
    startBlockId?: string // Optional: if provided, execution starts from this specific block.
  ): Promise<ExecutionResult | StreamingExecution> {
    // Destructure actions from the execution store to manage UI state.
    const { setIsExecuting, setIsDebugging, setPendingBlocks, reset } = useExecutionStore.getState()
    const startTime = new Date() // Record the start time of the execution.
    const executorStartMs = startTime.getTime() // Milliseconds timestamp for calculation.
    let finalOutput: NormalizedBlockOutput = {} // Placeholder for the final output of the workflow.

    // Track workflow execution start using telemetry.
    trackWorkflowTelemetry('workflow_execution_started', {
      workflowId,
      blockCount: this.actualWorkflow.blocks.length,
      connectionCount: this.actualWorkflow.connections.length,
      startTime: startTime.toISOString(),
    })

    const beforeValidation = Date.now() // Record time before validation.
    this.validateWorkflow(startBlockId) // Validate the workflow structure again, possibly with a specific start block.
    const validationTime = Date.now() - beforeValidation // Calculate validation duration.

    // Create the initial execution context.
    const context = this.createExecutionContext(workflowId, startTime, startBlockId)

    try {
      // Only update global execution state for the top-level (parent) execution, not child workflows.
      if (!this.isChildExecution) {
        setIsExecuting(true) // Indicate that workflow execution is active in the UI.

        if (this.isDebugging) {
          setIsDebugging(true) // If in debug mode, update UI to reflect debugging.
        }
      }

      let hasMoreLayers = true // Flag to continue the main execution loop.
      let iteration = 0 // Counter for execution iterations (layers).
      // const firstBlockExecutionTime: number | null = null; // Unused variable.
      const maxIterations = 500 // Safety limit to prevent infinite loops in the executor.

      // Main execution loop: continues as long as there are blocks to execute, within limits, and not cancelled.
      while (hasMoreLayers && iteration < maxIterations && !this.isCancelled) {
        const iterationStart = Date.now() // Record start time of the current iteration.
        const nextLayer = this.getNextExecutionLayer(context) // Determine which blocks are ready to execute in the current "layer".
        const getNextLayerTime = Date.now() - iterationStart // Calculate time taken to get the next layer.

        if (this.isDebugging) {
          // If in debug mode:
          setPendingBlocks(nextLayer) // Update the UI with blocks waiting to be executed.

          if (nextLayer.length === 0) {
            hasMoreLayers = false // If no more blocks, execution is complete.
          } else {
            // If there are pending blocks, return early to allow manual stepping by the user.
            // The `useWorkflowExecution` hook (caller) will resume execution.
            return {
              success: true,
              output: finalOutput,
              metadata: {
                duration: Date.now() - startTime.getTime(),
                startTime: context.metadata.startTime!,
                pendingBlocks: nextLayer,
                isDebugSession: true,
                context: context, // Return the full context for resumption.
                workflowConnections: this.actualWorkflow.connections.map((conn: any) => ({
                  source: conn.source,
                  target: conn.target,
                })),
              },
              logs: context.blockLogs,
            }
          }
        } else {
          // Normal execution (not debug mode):
          if (nextLayer.length === 0) {
            // If no immediate blocks are ready, check if there's ongoing parallel work that needs attention.
            hasMoreLayers = this.hasMoreParallelWork(context)
          } else {
            // Execute all blocks in the current layer concurrently.
            const outputs = await this.executeLayer(nextLayer, context)

            // Process outputs, especially for streaming blocks.
            for (const output of outputs) {
              // If an output contains 'stream' and 'execution', it's a streaming block.
              if (
                output &&
                typeof output === 'object' &&
                'stream' in output &&
                'execution' in output
              ) {
                if (context.onStream) {
                  const streamingExec = output as StreamingExecution
                  // `tee()` creates two independent, readable streams from a single `ReadableStream`.
                  // One for the client (UI), one for the executor (internal processing).
                  const [streamForClient, streamForExecutor] = streamingExec.stream.tee()

                  // Apply response format processing to the client stream if specified.
                  const blockId = (streamingExec.execution as any).blockId
                  let responseFormat: any
                  if (this.initialBlockStates?.[blockId]) {
                    const blockState = this.initialBlockStates[blockId] as any
                    responseFormat = blockState.responseFormat
                  }

                  const processedClientStream = streamingResponseFormatProcessor.processStream(
                    streamForClient,
                    blockId,
                    context.selectedOutputs || [],
                    responseFormat
                  )

                  const clientStreamingExec = { ...streamingExec, stream: processedClientStream }

                  try {
                    // Call the provided `onStream` callback to handle the client-facing stream.
                    await context.onStream(clientStreamingExec)
                  } catch (streamError: any) {
                    logger.error('Error in onStream callback:', streamError)
                    // Continue execution even if the client stream callback fails.
                  }

                  // Process the executor's internal stream to get the full content for the block's final output.
                  const reader = streamForExecutor.getReader()
                  const decoder = new TextDecoder()
                  let fullContent = ''

                  try {
                    while (true) {
                      const { done, value } = await reader.read() // Read chunks from the stream.
                      if (done) break // If done, exit loop.
                      fullContent += decoder.decode(value, { stream: true }) // Decode and append.
                    }

                    const blockId = (streamingExec.execution as any).blockId
                    const blockState = context.blockStates.get(blockId)
                    // Update the block's output with the complete streamed content.
                    this.updateBlockOutputWithStreamedContent(
                      blockId,
                      fullContent,
                      blockState,
                      context
                    )
                  } catch (readerError: any) {
                    logger.error('Error reading stream for executor:', readerError)
                    // If an error occurs, update with any partial content received.
                    const blockId = (streamingExec.execution as any).blockId
                    const blockState = context.blockStates.get(blockId)
                    if (fullContent) {
                      this.updateBlockOutputWithStreamedContent(
                        blockId,
                        fullContent,
                        blockState,
                        context
                      )
                    }
                  } finally {
                    try {
                      reader.releaseLock() // Release the reader lock.
                    } catch (releaseError: any) {
                      // Reader might already be released - this is expected and safe to ignore
                    }
                  }
                }
              }
            }

            // Filter out streaming outputs, leaving only the final, normalized block outputs.
            const normalizedOutputs = outputs
              .filter(
                (output) =>
                  !(
                    typeof output === 'object' &&
                    output !== null &&
                    'stream' in output &&
                    'execution' in output
                  )
              )
              .map((output) => output as NormalizedBlockOutput)

            // The final output of the workflow is typically the output of the last executed non-streaming block.
            if (normalizedOutputs.length > 0) {
              finalOutput = normalizedOutputs[normalizedOutputs.length - 1]
            }

            // After a layer, process loop iterations (this can activate new paths).
            await this.loopManager.processLoopIterations(context)

            // After a layer, process parallel iterations (similar to loops).
            await this.parallelManager.processParallelIterations(context)

            // Re-evaluate the next layer after processing loops/parallels, as new blocks might become ready.
            const updatedNextLayer = this.getNextExecutionLayer(context)
            if (updatedNextLayer.length === 0) {
              hasMoreLayers = false // If no more blocks are ready, the execution can stop.
            }
          }
        }

        iteration++ // Increment iteration counter.
      }

      // --- Execution Completion / Cancellation ---

      // Handle cancellation if the `isCancelled` flag was set.
      if (this.isCancelled) {
        trackWorkflowTelemetry('workflow_execution_cancelled', {
          workflowId,
          duration: Date.now() - startTime.getTime(),
          blockCount: this.actualWorkflow.blocks.length,
          executedBlockCount: context.executedBlocks.size,
          startTime: startTime.toISOString(),
        })

        return {
          success: false,
          output: finalOutput,
          error: 'Workflow execution was cancelled',
          metadata: {
            duration: Date.now() - startTime.getTime(),
            startTime: context.metadata.startTime!,
            workflowConnections: this.actualWorkflow.connections.map((conn: any) => ({
              source: conn.source,
              target: conn.target,
            })),
          },
          logs: context.blockLogs,
        }
      }

      // If execution completes successfully:
      const endTime = new Date()
      context.metadata.endTime = endTime.toISOString()
      const duration = endTime.getTime() - startTime.getTime()

      trackWorkflowTelemetry('workflow_execution_completed', {
        workflowId,
        duration,
        blockCount: this.actualWorkflow.blocks.length,
        executedBlockCount: context.executedBlocks.size,
        startTime: startTime.toISOString(),
        endTime: endTime.toISOString(),
        success: true,
      })

      return {
        success: true,
        output: finalOutput,
        metadata: {
          duration: duration,
          startTime: context.metadata.startTime!,
          endTime: context.metadata.endTime!,
          workflowConnections: this.actualWorkflow.connections.map((conn: any) => ({
            source: conn.source,
            target: conn.target,
          })),
        },
        logs: context.blockLogs,
      }
    } catch (error: any) {
      // --- Error Handling ---
      logger.error('Workflow execution failed:', this.sanitizeError(error)) // Log the error.

      // Track workflow execution failure using telemetry.
      trackWorkflowTelemetry('workflow_execution_failed', {
        workflowId,
        duration: Date.now() - startTime.getTime(),
        error: this.extractErrorMessage(error),
        executedBlockCount: context.executedBlocks.size,
        blockLogs: context.blockLogs.length,
      })

      return {
        success: false,
        output: finalOutput,
        error: this.extractErrorMessage(error),
        metadata: {
          duration: Date.now() - startTime.getTime(),
          startTime: context.metadata.startTime!,
          workflowConnections: this.actualWorkflow.connections.map((conn: any) => ({
            source: conn.source,
            target: conn.target,
          })),
        },
        logs: context.blockLogs,
      }
    } finally {
      // --- Cleanup ---
      // Only reset global state for parent executions and not in debug mode.
      if (!this.isChildExecution && !this.isDebugging) {
        reset() // Reset execution-related UI state in the store.
      }
    }
  }

  /**
   * Continues execution in debug mode from the current state.
   *
   * @param blockIds - Block IDs to execute in this step
   * @param context - The current execution context
   * @returns Updated execution result
   */
  async continueExecution(blockIds: string[], context: ExecutionContext): Promise<ExecutionResult> {
    const { setPendingBlocks } = useExecutionStore.getState() // Get `setPendingBlocks` action from the store.
    let finalOutput: NormalizedBlockOutput = {} // Initialize final output.

    // Check for cancellation.
    if (this.isCancelled) {
      return {
        success: false,
        output: finalOutput,
        error: 'Workflow execution was cancelled',
        metadata: {
          duration: Date.now() - new Date(context.metadata.startTime!).getTime(),
          startTime: context.metadata.startTime!,
          workflowConnections: this.actualWorkflow.connections.map((conn: any) => ({
            source: conn.source,
            target: conn.target,
          })),
        },
        logs: context.blockLogs,
      }
    }

    try {
      // Execute the current layer of blocks provided (specific for debug stepping).
      const outputs = await this.executeLayer(blockIds, context)

      // Process outputs, similar to the main `execute` method, but specifically for non-streaming.
      if (outputs.length > 0) {
        const nonStreamingOutputs = outputs.filter(
          (o) => !(o && typeof o === 'object' && 'stream' in o)
        ) as NormalizedBlockOutput[]
        if (nonStreamingOutputs.length > 0) {
          finalOutput = nonStreamingOutputs[nonStreamingOutputs.length - 1]
        }
      }
      // Process loop and parallel iterations after executing the layer.
      await this.loopManager.processLoopIterations(context)
      await this.parallelManager.processParallelIterations(context)
      // Determine the next set of blocks ready for execution.
      const nextLayer = this.getNextExecutionLayer(context)
      setPendingBlocks(nextLayer) // Update the UI with these pending blocks.

      const isComplete = nextLayer.length === 0 // Check if all blocks have been executed.

      if (isComplete) {
        // If execution is complete, finalize metadata and return success.
        const endTime = new Date()
        context.metadata.endTime = endTime.toISOString()

        return {
          success: true,
          output: finalOutput,
          metadata: {
            duration: endTime.getTime() - new Date(context.metadata.startTime!).getTime(),
            startTime: context.metadata.startTime!,
            endTime: context.metadata.endTime!,
            pendingBlocks: [], // No pending blocks.
            isDebugSession: false, // Debug session ends.
            workflowConnections: this.actualWorkflow.connections.map((conn) => ({
              source: conn.source,
              target: conn.target,
            })),
          },
          logs: context.blockLogs,
        }
      }

      // If not complete, return the current state for the next debug step.
      return {
        success: true,
        output: finalOutput,
        metadata: {
          duration: Date.now() - new Date(context.metadata.startTime!).getTime(),
          startTime: context.metadata.startTime!,
          pendingBlocks: nextLayer,
          isDebugSession: true, // Still in debug session.
          context: context, // Return the context for continuity.
        },
        logs: context.blockLogs,
      }
    } catch (error: any) {
      // Error handling for debug step execution.
      logger.error('Debug step execution failed:', this.sanitizeError(error))

      return {
        success: false,
        output: finalOutput,
        error: this.extractErrorMessage(error),
        metadata: {
          duration: Date.now() - new Date(context.metadata.startTime!).getTime(),
          startTime: context.metadata.startTime!,
          workflowConnections: this.actualWorkflow.connections.map((conn: any) => ({
            source: conn.source,
            target: conn.target,
          })),
        },
        logs: context.blockLogs,
      }
    }
  }

  /**
   * Validates that the workflow meets requirements for execution.
   * Checks for starter block, webhook trigger block, or schedule trigger block, connections, and loop configurations.
   *
   * @param startBlockId - Optional specific block to start from
   * @throws Error if workflow validation fails
   */
  private validateWorkflow(startBlockId?: string): void {
    // If a specific startBlockId is provided, just check if it exists and is enabled.
    if (startBlockId) {
      const startBlock = this.actualWorkflow.blocks.find((block) => block.id === startBlockId)
      if (!startBlock || !startBlock.enabled) {
        throw new Error(`Start block ${startBlockId} not found or disabled`)
      }
      return
    }

    // Find the legacy "Starter" block.
    const starterBlock = this.actualWorkflow.blocks.find(
      (block) => block.metadata?.id === BlockType.STARTER
    )

    // Check for any type of trigger block (dedicated triggers or blocks with trigger mode enabled).
    const hasTriggerBlocks = this.actualWorkflow.blocks.some((block) => {
      if (block.metadata?.category === 'triggers') return true // Dedicated trigger category.
      if (block.config?.params?.triggerMode === true) return true // Blocks configured as triggers.
      return false
    })

    if (hasTriggerBlocks) {
      // If trigger blocks exist, a legacy starter block is not strictly required.
      // The actual start block will be determined by the execution context at runtime.
    } else {
      // For older workflows or workflows without explicit triggers, a Starter block is mandatory.
      if (!starterBlock || !starterBlock.enabled) {
        throw new Error('Workflow must have an enabled starter block')
      }

      // Validate connections for the Starter block.
      const incomingToStarter = this.actualWorkflow.connections.filter(
        (conn) => conn.target === starterBlock.id
      )
      if (incomingToStarter.length > 0) {
        throw new Error('Starter block cannot have incoming connections')
      }

      const outgoingFromStarter = this.actualWorkflow.connections.filter(
        (conn) => conn.source === starterBlock.id
      )
      if (outgoingFromStarter.length === 0) {
        throw new Error('Starter block must have at least one outgoing connection')
      }
    }

    // General graph validations: check if all connections reference existing blocks.
    const blockIds = new Set(this.actualWorkflow.blocks.map((block) => block.id))
    for (const conn of this.actualWorkflow.connections) {
      if (!blockIds.has(conn.source)) {
        throw new Error(`Connection references non-existent source block: ${conn.source}`)
      }
      if (!blockIds.has(conn.target)) {
        throw new Error(`Connection references non-existent target block: ${conn.target}`)
      }
    }

    // Validate loop configurations.
    for (const [loopId, loop] of Object.entries(this.actualWorkflow.loops || {})) {
      for (const nodeId of loop.nodes) {
        if (!blockIds.has(nodeId)) {
          throw new Error(`Loop ${loopId} references non-existent block: ${nodeId}`)
        }
      }

      if (loop.iterations <= 0) {
        throw new Error(`Loop ${loopId} must have a positive iterations value`)
      }

      if (loop.loopType === 'forEach') {
        if (
          !loop.forEachItems ||
          (typeof loop.forEachItems === 'string' && loop.forEachItems.trim() === '')
        ) {
          throw new Error(`forEach loop ${loopId} requires a collection to iterate over`)
        }
      }
    }
  }

  /**
   * Creates the initial execution context with predefined states.
   * Sets up the starter block, webhook trigger block, or schedule trigger block and its connections in the active execution path.
   *
   * @param workflowId - Unique identifier for the workflow execution
   * @param startTime - Execution start time
   * @param startBlockId - Optional specific block to start from
   * @returns Initialized execution context
   */
  private createExecutionContext(
    workflowId: string,
    startTime: Date,
    startBlockId?: string
  ): ExecutionContext {
    // Initialize the main execution context object with default values and provided parameters.
    const context: ExecutionContext = {
      workflowId,
      workspaceId: this.contextExtensions.workspaceId,
      executionId: this.contextExtensions.executionId,
      isDeployedContext: this.contextExtensions.isDeployedContext || false,
      blockStates: new Map(), // Stores output and status for each block.
      blockLogs: [], // List of execution logs for each block.
      metadata: {
        startTime: startTime.toISOString(),
        duration: 0, // Placeholder, updated later.
      },
      environmentVariables: this.environmentVariables, // Environment variables passed in.
      workflowVariables: this.workflowVariables, // Workflow-specific variables.
      decisions: {
        router: new Map(), // Stores decisions made by router blocks.
        condition: new Map(), // Stores decisions made by condition blocks.
      },
      loopIterations: new Map(), // Current iteration count for each loop.
      loopItems: new Map(), // Items being iterated over for 'forEach' loops.
      completedLoops: new Set(), // Set of completed loop/parallel IDs.
      executedBlocks: new Set(), // Set of all blocks that have completed execution.
      activeExecutionPath: new Set(), // Blocks currently considered part of the active path.
      workflow: this.actualWorkflow,
      // Pass through context extensions for streaming, selected outputs, etc.
      stream: this.contextExtensions.stream || false,
      selectedOutputs: this.contextExtensions.selectedOutputs || [],
      edges: this.contextExtensions.edges || [],
      onStream: this.contextExtensions.onStream,
      onBlockComplete: this.contextExtensions.onBlockComplete,
    }

    // Populate `blockStates` with any `initialBlockStates` provided to the constructor.
    Object.entries(this.initialBlockStates).forEach(([blockId, output]) => {
      context.blockStates.set(blockId, {
        output: output as NormalizedBlockOutput,
        executed: true,
        executionTime: 0,
      })
    })

    // Initialize loop iterations: set all loops to iteration 0.
    if (this.actualWorkflow.loops) {
      for (const loopId of Object.keys(this.actualWorkflow.loops)) {
        context.loopIterations.set(loopId, 0)
      }
    }

    // Determine the initial block to start execution from.
    let initBlock: SerializedBlock | undefined
    if (startBlockId) {
      // If a specific `startBlockId` is provided (e.g., for webhook/schedule triggers), use it.
      initBlock = this.actualWorkflow.blocks.find((block) => block.id === startBlockId)
    } else {
      // Otherwise, try to find the legacy 'STARTER' block.
      initBlock = this.actualWorkflow.blocks.find(
        (block) => block.metadata?.id === BlockType.STARTER
      )

      // If no 'STARTER' block, look for other trigger blocks based on execution context.
      if (!initBlock) {
        if (this.isChildExecution) {
          // For child workflows, look for a single 'input_trigger' block.
          const inputTriggerBlocks = this.actualWorkflow.blocks.filter(
            (block) => block.metadata?.id === 'input_trigger'
          )
          if (inputTriggerBlocks.length === 1) {
            initBlock = inputTriggerBlocks[0]
          } else if (inputTriggerBlocks.length > 1) {
            throw new Error('Child workflow has multiple Input Trigger blocks. Keep only one.')
          }
        } else {
          // For parent workflows, prioritize any trigger block.
          const triggerBlocks = this.actualWorkflow.blocks.filter(
            (block) =>
              block.metadata?.id === 'input_trigger' ||
              block.metadata?.id === 'api_trigger' ||
              block.metadata?.id === 'chat_trigger' ||
              block.metadata?.category === 'triggers' || // Generic triggers category.
              block.config?.params?.triggerMode === true // Blocks configured in trigger mode.
          )
          if (triggerBlocks.length > 0) {
            initBlock = triggerBlocks[0]
          }
        }
      }
    }

    // If an initial block is found, initialize its state.
    if (initBlock) {
      try {
        // Resolve input format from block config or metadata.
        const blockParams = initBlock.config.params
        let inputFormat = blockParams?.inputFormat
        const metadataWithSubBlocks = initBlock.metadata as any
        if (!inputFormat && metadataWithSubBlocks?.subBlocks?.inputFormat?.value) {
          inputFormat = metadataWithSubBlocks.subBlocks.inputFormat.value
        }

        // If an input format is defined, structure the workflow input according to its schema.
        if (inputFormat && Array.isArray(inputFormat) && inputFormat.length > 0) {
          const structuredInput: Record<string, any> = {}

          for (const field of inputFormat) {
            if (field.name && field.type) {
              // Try to get input value from `workflowInput.input.field` or `workflowInput.field`.
              let inputValue =
                this.workflowInput?.input?.[field.name] !== undefined
                  ? this.workflowInput.input[field.name]
                  : this.workflowInput?.[field.name]

              // Fallback to default `value` from the field definition if input is undefined/null.
              if (inputValue === undefined || inputValue === null) {
                if (Object.hasOwn(field, 'value')) {
                  inputValue = (field as any).value
                }
              }

              // Type coercion based on field type.
              let typedValue = inputValue
              if (inputValue !== undefined && inputValue !== null) {
                if (field.type === 'string' && typeof inputValue !== 'string') {
                  typedValue = String(inputValue)
                } else if (field.type === 'number' && typeof inputValue !== 'number') {
                  const num = Number(inputValue)
                  typedValue = Number.isNaN(num) ? inputValue : num
                } else if (field.type === 'boolean' && typeof inputValue !== 'boolean') {
                  typedValue =
                    inputValue === 'true' ||
                    inputValue === true ||
                    inputValue === 1 ||
                    inputValue === '1'
                } else if (
                  (field.type === 'object' || field.type === 'array') &&
                  typeof inputValue === 'string'
                ) {
                  try {
                    typedValue = JSON.parse(inputValue)
                  } catch (e) {
                    logger.warn(`Failed to parse ${field.type} input for field ${field.name}:`, e)
                  }
                }
              }

              structuredInput[field.name] = typedValue // Add to structured input.
            }
          }

          const hasProcessedFields = Object.keys(structuredInput).length > 0
          // Use `workflowInput.input` or raw `workflowInput` as fallback if no fields matched.
          const rawInputData =
            this.workflowInput?.input !== undefined ? this.workflowInput.input : this.workflowInput

          const finalInput = hasProcessedFields ? structuredInput : rawInputData // Prioritize structured input.

          let blockOutput: any
          // Special handling for API/Input triggers to normalize output structure.
          if (
            initBlock.metadata?.id === 'api_trigger' ||
            initBlock.metadata?.id === 'input_trigger'
          ) {
            const isObject =
              finalInput !== null && typeof finalInput === 'object' && !Array.isArray(finalInput)
            if (isObject) {
              blockOutput = { ...finalInput }
              blockOutput.input = { ...finalInput } // Mirror input object for universal references.
            } else {
              blockOutput = { input: finalInput } // Primitive input gets wrapped under 'input'.
            }
          } else {
            // Legacy starter block handling: spread input fields at top level.
            blockOutput = {
              input: finalInput,
              conversationId: this.workflowInput?.conversationId,
              ...finalInput,
            }
          }

          // Add files if present in the workflow input.
          if (this.workflowInput?.files && Array.isArray(this.workflowInput.files)) {
            blockOutput.files = this.workflowInput.files
          }

          // Set the initial block's state with the resolved output.
          context.blockStates.set(initBlock.id, {
            output: blockOutput,
            executed: true,
            executionTime: 0,
          })

          // Create a log entry for the starter block if it contains files.
          this.createStartedBlockWithFilesLog(initBlock, blockOutput, context)
        } else {
          // Handle triggers WITHOUT an explicit inputFormat definition.
          let starterOutput: any
          if (initBlock.metadata?.id === 'chat_trigger') {
            // Chat trigger: extract input and conversationId.
            starterOutput = {
              input: this.workflowInput?.input || '',
              conversationId: this.workflowInput?.conversationId || '',
            }
            if (this.workflowInput?.files && Array.isArray(this.workflowInput.files)) {
              starterOutput.files = this.workflowInput.files
            }
          } else if (
            initBlock.metadata?.id === 'api_trigger' ||
            initBlock.metadata?.id === 'input_trigger'
          ) {
            // API/Input trigger: normalize raw input.
            const rawCandidate =
              this.workflowInput?.input !== undefined
                ? this.workflowInput.input
                : this.workflowInput
            const isObject =
              rawCandidate !== null &&
              typeof rawCandidate === 'object' &&
              !Array.isArray(rawCandidate)
            if (isObject) {
              starterOutput = {
                ...(rawCandidate as Record<string, any>),
                input: { ...(rawCandidate as Record<string, any>) },
              }
            } else {
              starterOutput = { input: rawCandidate }
            }
          } else {
            // Legacy starter block: handle chat workflow input or spread API workflow input.
            if (this.workflowInput && typeof this.workflowInput === 'object') {
              if (
                Object.hasOwn(this.workflowInput, 'input') &&
                Object.hasOwn(this.workflowInput, 'conversationId')
              ) {
                starterOutput = {
                  input: this.workflowInput.input,
                  conversationId: this.workflowInput.conversationId,
                }
                if (this.workflowInput.files && Array.isArray(this.workflowInput.files)) {
                  starterOutput.files = this.workflowInput.files
                }
              } else {
                starterOutput = { ...this.workflowInput }
              }
            } else {
              starterOutput = {
                input: this.workflowInput,
              }
            }
          }

          context.blockStates.set(initBlock.id, {
            output: starterOutput,
            executed: true,
            executionTime: 0,
          })

          if (starterOutput.files) {
            this.createStartedBlockWithFilesLog(initBlock, starterOutput, context)
          }
        }
      } catch (e) {
        logger.warn('Error processing starter block input format:', e) // Log error if input processing fails.

        // Fallback error handler: try to set a reasonable block output even if input processing failed.
        let blockOutput: any
        if (this.workflowInput && typeof this.workflowInput === 'object') {
          if (
            Object.hasOwn(this.workflowInput, 'input') &&
            Object.hasOwn(this.workflowInput, 'conversationId')
          ) {
            blockOutput = {
              input: this.workflowInput.input,
              conversationId: this.workflowInput.conversationId,
            }
            if (this.workflowInput.files && Array.isArray(this.workflowInput.files)) {
              blockOutput.files = this.workflowInput.files
            }
          } else {
            blockOutput = { ...this.workflowInput }
          }
        } else {
          blockOutput = {
            input: this.workflowInput,
          }
        }

        context.blockStates.set(initBlock.id, {
          output: blockOutput,
          executed: true,
          executionTime: 0,
        })
        this.createStartedBlockWithFilesLog(initBlock, blockOutput, context)
      }
      // Add the initial block to the active execution path and mark as executed.
      context.activeExecutionPath.add(initBlock.id)
      context.executedBlocks.add(initBlock.id)

      // Add all blocks directly connected to the initial block to the active execution path.
      const connectedToStartBlock = this.actualWorkflow.connections
        .filter((conn) => conn.source === initBlock.id)
        .map((conn) => conn.target)

      connectedToStartBlock.forEach((blockId) => {
        context.activeExecutionPath.add(blockId)
      })
    }

    return context
  }

  /**
   * Determines the next layer of blocks to execute based on dependencies and execution path.
   * Handles special cases for blocks in loops, condition blocks, and router blocks.
   * For blocks inside parallel executions, creates multiple virtual instances.
   *
   * @param context - Current execution context
   * @returns Array of block IDs that are ready to be executed
   */
  private getNextExecutionLayer(context: ExecutionContext): string[] {
    const executedBlocks = context.executedBlocks // Get the set of already executed blocks.
    const pendingBlocks = new Set<string>() // Initialize a set for blocks ready to be executed.

    // Check if we have any active parallel executions.
    const activeParallels = new Map<string, any>()
    if (context.parallelExecutions) {
      for (const [parallelId, state] of context.parallelExecutions) {
        // A parallel is active if its current iteration count is valid and it hasn't completed.
        if (
          state.currentIteration > 0 &&
          state.currentIteration <= state.parallelCount &&
          !context.completedLoops.has(parallelId)
        ) {
          activeParallels.set(parallelId, state)
        }
      }
    }

    // Iterate through all blocks in the workflow to find those ready for execution.
    for (const block of this.actualWorkflow.blocks) {
      // Skip if already executed or disabled.
      if (executedBlocks.has(block.id) || block.enabled === false) {
        continue
      }

      // Determine if the current block is inside a parallel execution.
      let insideParallel: string | null = null
      for (const [parallelId, parallel] of Object.entries(this.actualWorkflow.parallels || {})) {
        if (parallel.nodes.includes(block.id)) {
          insideParallel = parallelId
          break
        }
      }

      // If the block is inside an active parallel, its execution is managed by `processParallelBlocks`.
      // The logic here handles blocks *not* actively part of a parallel iteration yet or those outside parallels.
      if (insideParallel && activeParallels.has(insideParallel)) {
        // Do nothing here, these blocks will be processed by `processParallelBlocks`.
      } else if (insideParallel) {
        // Block is inside a parallel, but the parallel itself is not currently active (e.g., waiting for inputs).
        const parallelState = context.parallelExecutions?.get(insideParallel)
        if (parallelState) {
          let allVirtualInstancesExecuted = true
          // Check if all virtual instances of this block within the parallel have already executed.
          for (let i = 0; i < parallelState.parallelCount; i++) {
            const virtualBlockId = VirtualBlockUtils.generateParallelId(block.id, insideParallel, i)
            if (!executedBlocks.has(virtualBlockId)) {
              allVirtualInstancesExecuted = false
              break
            }
          }

          // If all virtual instances are done, skip this block for regular execution.
          if (allVirtualInstancesExecuted) {
            continue
          }
        }

        // If we reach here, it means the parallel hasn't been initialized yet or its iterations haven't started.
        // Proceed with normal dependency checking for the original block ID.
        // Only consider blocks that are currently in the active execution path.
        if (!context.activeExecutionPath.has(block.id)) {
          continue
        }

        const incomingConnections = this.actualWorkflow.connections.filter(
          (conn) => conn.target === block.id
        )

        // Check if all dependencies for this block are met.
        const allDependenciesMet = this.checkDependencies(
          incomingConnections,
          executedBlocks,
          context
        )

        if (allDependenciesMet) {
          pendingBlocks.add(block.id) // Add to pending if dependencies met.
        }
      } else {
        // Regular block handling (not inside any parallel).
        // Only consider blocks that are currently in the active execution path.
        if (!context.activeExecutionPath.has(block.id)) {
          continue
        }

        const incomingConnections = this.actualWorkflow.connections.filter(
          (conn) => conn.target === block.id
        )

        // Check if all dependencies for this block are met.
        const allDependenciesMet = this.checkDependencies(
          incomingConnections,
          executedBlocks,
          context
        )

        if (allDependenciesMet) {
          pendingBlocks.add(block.id)
        }
      }
    }

    // After processing regular blocks, specifically process blocks within active parallel executions.
    this.processParallelBlocks(activeParallels, context, pendingBlocks)

    return Array.from(pendingBlocks) // Return the list of unique block IDs ready for execution.
  }

  /**
   * Process all active parallel blocks with proper dependency ordering within iterations.
   * This ensures that blocks with dependencies within the same iteration are executed
   * in the correct order, preventing race conditions. Only processes one iteration at a time
   * to maintain proper execution order.
   *
   * @param activeParallels - Map of active parallel executions
   * @param context - Execution context
   * @param pendingBlocks - Set to add ready blocks to
   */
  private processParallelBlocks(
    activeParallels: Map<string, any>,
    context: ExecutionContext,
    pendingBlocks: Set<string>
  ): void {
    for (const [parallelId, parallelState] of activeParallels) {
      const parallel = this.actualWorkflow.parallels?.[parallelId]
      if (!parallel) continue

      // Iterate through each parallel instance (iteration) to find blocks ready for execution within it.
      for (let iteration = 0; iteration < parallelState.parallelCount; iteration++) {
        // If an iteration is already complete, skip it.
        if (this.isIterationComplete(parallelId, iteration, parallel, context)) {
          continue
        }

        // Process this specific parallel iteration to find its ready blocks.
        this.processParallelIteration(parallelId, iteration, parallel, context, pendingBlocks)
      }
    }
  }

  /**
   * Check if a specific parallel iteration is complete (all blocks that should execute have executed).
   * This method now considers conditional execution paths - only blocks in the active execution
   * path are expected to execute.
   *
   * @param parallelId - ID of the parallel block
   * @param iteration - Iteration index to check
   * @param parallel - Parallel configuration
   * @param context - Execution context
   * @returns Whether the iteration is complete
   */
  private isIterationComplete(
    parallelId: string,
    iteration: number,
    parallel: any,
    context: ExecutionContext
  ): boolean {
    if (!parallel || !parallel.nodes) {
      return true // If parallel has no nodes, it's considered complete.
    }

    // Get the list of blocks *expected* to execute in this specific iteration (considering conditions).
    const expectedBlocks = this.getExpectedBlocksForIteration(
      parallelId,
      iteration,
      parallel,
      context
    )

    // Check if all *expected* virtual instances of these blocks have been executed.
    for (const nodeId of expectedBlocks) {
      const virtualBlockId = VirtualBlockUtils.generateParallelId(nodeId, parallelId, iteration)
      if (!context.executedBlocks.has(virtualBlockId)) {
        return false // If any expected block isn't executed, the iteration is not complete.
      }
    }
    return true // All expected blocks executed, iteration is complete.
  }

  /**
   * Get the blocks that are expected to execute in a parallel iteration based on
   * the active execution path. This handles conditional logic where some blocks
   * may not execute due to condition or router blocks.
   *
   * @param parallelId - ID of the parallel block
   * @param iteration - Iteration index
   * @param parallel - Parallel configuration
   * @param context - Execution context
   * @returns Array of node IDs that should execute in this iteration
   */
  private getExpectedBlocksForIteration(
    parallelId: string,
    iteration: number,
    parallel: any,
    context: ExecutionContext
  ): string[] {
    if (!parallel || !parallel.nodes) {
      return [] // If no nodes, no blocks are expected.
    }

    const expectedBlocks: string[] = []

    for (const nodeId of parallel.nodes) {
      const block = this.actualWorkflow.blocks.find((b) => b.id === nodeId)

      // Fallback: if block doesn't exist, assume it should execute for compatibility.
      if (!block) {
        expectedBlocks.push(nodeId)
        continue
      }

      if (!block.enabled) {
        continue // Skip disabled blocks.
      }

      const virtualBlockId = VirtualBlockUtils.generateParallelId(nodeId, parallelId, iteration)

      // If the virtual block is already executed, include it as expected (it has completed).
      if (context.executedBlocks.has(virtualBlockId)) {
        expectedBlocks.push(nodeId)
        continue
      }

      // Crucial: Check if this block *should* execute in this iteration, considering conditional paths.
      try {
        const shouldExecute = this.shouldBlockExecuteInParallelIteration(
          nodeId,
          parallelId,
          iteration,
          context
        )

        if (shouldExecute) {
          expectedBlocks.push(nodeId) // Add to expected if it should execute.
        }
      } catch (error) {
        logger.warn(
          `Path check failed for block ${nodeId} in parallel ${parallelId}, iteration ${iteration}:`,
          error
        )
        expectedBlocks.push(nodeId) // If path checking fails, default to including it.
      }
    }

    return expectedBlocks
  }

  /**
   * Determines if a block should execute in a specific parallel iteration
   * based on conditional routing and active execution paths.
   *
   * Blocks are excluded from execution if they are completely unconnected (no incoming connections).
   * Starting blocks (with external connections only) and conditionally routed blocks execute as expected.
   *
   * @param nodeId - ID of the block to check
   * @param parallelId - ID of the parallel block
   * @param iteration - Current iteration index
   * @param context - Execution context
   * @returns Whether the block should execute
   */
  private shouldBlockExecuteInParallelIteration(
    nodeId: string,
    parallelId: string,
    iteration: number,
    context: ExecutionContext
  ): boolean {
    const parallel = this.actualWorkflow.parallels?.[parallelId]
    if (!parallel) return false

    // Delegates the complex routing logic within a parallel iteration to a utility class.
    return ParallelRoutingUtils.shouldBlockExecuteInParallelIteration(
      nodeId,
      parallel,
      iteration,
      context
    )
  }

  /**
   * Check if there are more parallel iterations to process.
   * This ensures the execution loop continues when iterations are being processed sequentially.
   */
  private hasMoreParallelWork(context: ExecutionContext): boolean {
    if (!context.parallelExecutions) {
      return false // No parallel executions means no more parallel work.
    }

    for (const [parallelId, parallelState] of context.parallelExecutions) {
      // Skip parallels that are already marked as completed.
      if (context.completedLoops.has(parallelId)) {
        continue
      }

      // Check if this parallel is in an active state.
      if (
        parallelState.currentIteration > 0 &&
        parallelState.currentIteration <= parallelState.parallelCount
      ) {
        const parallel = this.actualWorkflow.parallels?.[parallelId]
        if (!parallel) continue

        // Iterate through all instances (iterations) of this parallel.
        for (let iteration = 0; iteration < parallelState.parallelCount; iteration++) {
          // If any iteration is not complete, then there's more parallel work.
          if (!this.isIterationComplete(parallelId, iteration, parallel, context)) {
            return true
          }
        }
      }
    }

    return false // No incomplete parallel work found.
  }

  /**
   * Process a single parallel iteration with topological ordering of dependencies.
   * Now includes conditional execution logic - only processes blocks that should execute
   * based on the active execution path (handles conditions, routers, etc.).
   *
   * @param parallelId - ID of the parallel block
   * @param iteration - Current iteration index
   * @param parallel - Parallel configuration
   * @param context - Execution context
   * @param pendingBlocks - Set to add ready blocks to
   */
  private processParallelIteration(
    parallelId: string,
    iteration: number,
    parallel: any,
    context: ExecutionContext,
    pendingBlocks: Set<string>
  ): void {
    const iterationBlocks = new Map<
      string,
      {
        virtualBlockId: string
        originalBlockId: string
        dependencies: string[]
        isExecuted: boolean
      }
    >() // Stores information about blocks within this specific iteration.

    // Build a dependency graph for this iteration, considering only blocks that are expected to execute.
    for (const nodeId of parallel.nodes) {
      const virtualBlockId = VirtualBlockUtils.generateParallelId(nodeId, parallelId, iteration)
      const isExecuted = context.executedBlocks.has(virtualBlockId)

      if (isExecuted) {
        continue // Skip blocks that have already executed.
      }

      const block = this.actualWorkflow.blocks.find((b) => b.id === nodeId)
      if (!block || !block.enabled) continue // Skip non-existent or disabled blocks.

      // Crucial: Check if this block should execute in this specific iteration (conditional path check).
      try {
        const shouldExecute = this.shouldBlockExecuteInParallelIteration(
          nodeId,
          parallelId,
          iteration,
          context
        )

        if (!shouldExecute) {
          continue // If it shouldn't execute, skip it for this iteration.
        }
      } catch (error) {
        logger.warn(
          `Path check failed for block ${nodeId} in parallel ${parallelId}, iteration ${iteration}:`,
          error
        )
      }

      // Find dependencies *within* this parallel iteration.
      const incomingConnections = this.actualWorkflow.connections.filter(
        (conn) => conn.target === nodeId
      )

      const dependencies: string[] = []
      for (const conn of incomingConnections) {
        // If the dependency source is within the same parallel, use its virtual ID.
        if (parallel.nodes.includes(conn.source)) {
          const sourceDependencyId = VirtualBlockUtils.generateParallelId(
            conn.source,
            parallelId,
            iteration
          )
          dependencies.push(sourceDependencyId)
        } else {
          // If it's an external dependency, check if it's already met.
          const isExternalDepMet = this.checkDependencies([conn], context.executedBlocks, context)
          if (!isExternalDepMet) {
            return // If an external dependency isn't met, this block can't run yet.
          }
        }
      }

      iterationBlocks.set(virtualBlockId, {
        virtualBlockId,
        originalBlockId: nodeId,
        dependencies,
        isExecuted,
      })
    }

    // Identify blocks within this iteration that have no unmet dependencies.
    for (const [virtualBlockId, blockInfo] of iterationBlocks) {
      const unmetDependencies = blockInfo.dependencies.filter((depId) => {
        // A dependency is unmet if it hasn't executed AND it's part of this iteration.
        return !context.executedBlocks.has(depId) && iterationBlocks.has(depId)
      })

      if (unmetDependencies.length === 0) {
        pendingBlocks.add(virtualBlockId) // Add to the global pending blocks if ready.

        // Store mapping from virtual ID to original block and iteration info.
        if (!context.parallelBlockMapping) {
          context.parallelBlockMapping = new Map()
        }
        context.parallelBlockMapping.set(virtualBlockId, {
          originalBlockId: blockInfo.originalBlockId,
          parallelId: parallelId,
          iterationIndex: iteration,
        })
      }
    }
  }

  /**
   * Checks if all dependencies for a block are met.
   * Handles special cases for different connection types.
   *
   * @param incomingConnections - Connections coming into the block
   * @param executedBlocks - Set of executed block IDs
   * @param context - Execution context
   * @param insideParallel - ID of parallel block if this block is inside one
   * @param iterationIndex - Index of the parallel iteration if applicable
   * @returns Whether all dependencies are met
   */
  private checkDependencies(
    incomingConnections: any[],
    executedBlocks: Set<string>,
    context: ExecutionContext,
    insideParallel?: string, // Optional: if checking dependencies for a block inside a parallel iteration.
    iterationIndex?: number // Optional: the iteration index for parallel blocks.
  ): boolean {
    if (incomingConnections.length === 0) {
      return true // No incoming connections means no dependencies, so they're met.
    }

    // Check if this is a loop block.
    const isLoopBlock = incomingConnections.some((conn) => {
      const sourceBlock = this.actualWorkflow.blocks.find((b) => b.id === conn.source)
      return sourceBlock?.metadata?.id === BlockType.LOOP
    })

    if (isLoopBlock) {
      // Loop blocks typically have standard dependency checking.
      return incomingConnections.every((conn) => {
        const sourceExecuted = executedBlocks.has(conn.source)
        const sourceBlockState = context.blockStates.get(conn.source)
        const hasSourceError = sourceBlockState?.output?.error !== undefined

        // For "error" connections, it's met if the source executed AND had an error.
        if (conn.sourceHandle === 'error') {
          return sourceExecuted && hasSourceError
        }

        // For regular connections, it's met if source executed WITHOUT an error.
        if (conn.sourceHandle === 'source' || !conn.sourceHandle) {
          return sourceExecuted && !hasSourceError
        }

        // If the source of the connection is not in the active path, it doesn't block execution.
        if (!context.activeExecutionPath.has(conn.source)) {
          return true
        }

        return sourceExecuted // Default: dependency met if source executed.
      })
    }

    // Regular non-loop block handling.
    return incomingConnections.every((conn) => {
      // Adjust sourceId if checking for a block inside a parallel (using virtual IDs).
      let sourceId = conn.source
      if (insideParallel !== undefined && iterationIndex !== undefined) {
        const sourceBlock = this.actualWorkflow.blocks.find((b) => b.id === conn.source)
        if (
          sourceBlock &&
          this.actualWorkflow.parallels?.[insideParallel]?.nodes.includes(conn.source)
        ) {
          sourceId = VirtualBlockUtils.generateParallelId(
            conn.source,
            insideParallel,
            iterationIndex
          )
        }
      }

      const sourceExecuted = executedBlocks.has(sourceId) // Check if source (actual or virtual) is executed.
      const sourceBlock = this.actualWorkflow.blocks.find((b) => b.id === conn.source) // Get original source block.
      const sourceBlockState =
        context.blockStates.get(sourceId) || context.blockStates.get(conn.source) // Get source state.
      const hasSourceError = sourceBlockState?.output?.error !== undefined // Check for source error.

      // Special handling for connections from loop start/end.
      if (conn.sourceHandle === 'loop-start-source') {
        return sourceExecuted // Activated when loop block executes.
      }
      if (conn.sourceHandle === 'loop-end-source') {
        return context.completedLoops.has(conn.source) // Activated when loop completes.
      }

      // Special handling for connections from parallel start/end.
      if (conn.sourceHandle === 'parallel-start-source') {
        return executedBlocks.has(conn.source) // Activated when parallel block executes.
      }
      if (conn.sourceHandle === 'parallel-end-source') {
        return context.completedLoops.has(conn.source) // Activated when parallel completes.
      }

      // Special handling for Condition blocks.
      if (conn.sourceHandle?.startsWith('condition-')) {
        if (sourceBlock?.metadata?.id === BlockType.CONDITION) {
          const conditionId = conn.sourceHandle.replace('condition-', '')
          const selectedCondition = context.decisions.condition.get(conn.source)

          // If source executed and this is NOT the selected path, consider it met (it doesn't block).
          if (sourceExecuted && selectedCondition && conditionId !== selectedCondition) {
            return true
          }
          // Otherwise, it's met if source executed AND this IS the selected path.
          return sourceExecuted && conditionId === selectedCondition
        }
      }

      // Special handling for Router blocks.
      if (sourceBlock?.metadata?.id === BlockType.ROUTER) {
        const selectedTarget = context.decisions.router.get(conn.source)

        // If source executed and this is NOT the selected target, this dependency is NOT met.
        if (sourceExecuted && selectedTarget && conn.target !== selectedTarget) {
          return false
        }
        // Otherwise, it's met if source executed AND this IS the selected target.
        return sourceExecuted && conn.target === selectedTarget
      }

      // If the source block of the connection is not in the active path, it's considered met (doesn't block).
      if (!context.activeExecutionPath.has(conn.source)) {
        return true
      }

      // For "error" connections, met if source executed AND had an error.
      if (conn.sourceHandle === 'error') {
        return sourceExecuted && hasSourceError
      }

      // For regular connections, met if source executed WITHOUT an error.
      if (conn.sourceHandle === 'source' || !conn.sourceHandle) {
        return sourceExecuted && !hasSourceError
      }

      return sourceExecuted // Default: dependency met if source executed.
    })
  }

  /**
   * Executes a layer of blocks in parallel.
   * Updates execution paths based on router and condition decisions.
   *
   * @param blockIds - IDs of blocks to execute
   * @param context - Current execution context
   * @returns Array of block outputs
   */
  private async executeLayer(
    blockIds: string[], // List of block IDs to execute in this layer.
    context: ExecutionContext // Current execution context.
  ): Promise<(NormalizedBlockOutput | StreamingExecution)[]> {
    const { setActiveBlocks } = useExecutionStore.getState() // Get `setActiveBlocks` action from the store.

    try {
      const activeBlockIds = new Set(blockIds) // Blocks currently active/executing.

      // For virtual blocks (parallel iterations), also add the original block ID to `activeBlockIds` for UI representation.
      blockIds.forEach((blockId) => {
        if (context.parallelBlockMapping?.has(blockId)) {
          const parallelInfo = context.parallelBlockMapping.get(blockId)
          if (parallelInfo) {
            activeBlockIds.add(parallelInfo.originalBlockId)
          }
        }
      })

      // Only manage active blocks for parent executions.
      if (!this.isChildExecution) {
        setActiveBlocks(activeBlockIds) // Update UI store with active blocks.
      }

      // Execute all blocks in the layer concurrently using `Promise.allSettled`.
      // `allSettled` allows some promises to reject without stopping the others.
      const settledResults = await Promise.allSettled(
        blockIds.map((blockId) => this.executeBlock(blockId, context))
      )

      const results: (NormalizedBlockOutput | StreamingExecution)[] = [] // Collect successful outputs.
      const errors: Error[] = [] // Collect errors.

      settledResults.forEach((result) => {
        if (result.status === 'fulfilled') {
          results.push(result.value) // Add successful results.
        } else {
          errors.push(result.reason) // Add errors.
          // For failed blocks, add a placeholder output to maintain array length for consistent mapping.
          results.push({
            error: result.reason?.message || 'Block execution failed',
            status: 500,
          })
        }
      })

      if (errors.length > 0) {
        logger.warn(
          `Layer execution completed with ${errors.length} failed blocks out of ${blockIds.length} total`
        )
        // If ALL blocks in the layer failed, throw the first error to propagate failure.
        if (errors.length === blockIds.length) {
          throw errors[0]
        }
      }

      // Mark all blocks in this layer as executed.
      blockIds.forEach((blockId) => {
        context.executedBlocks.add(blockId)
      })

      // Update active execution paths based on decisions made by blocks in this layer (e.g., router, condition).
      this.pathTracker.updateExecutionPaths(blockIds, context)

      return results
    } catch (error) {
      // If an uncaught error occurs during layer execution, clear active blocks.
      if (!this.isChildExecution) {
        setActiveBlocks(new Set())
      }
      throw error // Re-throw the error.
    }
  }

  /**
   * Executes a single block with error handling and logging.
   * Handles virtual block IDs for parallel iterations.
   *
   * @param blockId - ID of the block to execute (may be a virtual ID)
   * @param context - Current execution context
   * @returns Normalized block output
   * @throws Error if block execution fails
   */
  private async executeBlock(
    blockId: string, // The ID of the block to execute (could be a virtual ID for parallels).
    context: ExecutionContext // The current execution context.
  ): Promise<NormalizedBlockOutput | StreamingExecution> {
    let actualBlockId = blockId // Initialize with the given block ID.
    let parallelInfo:
      | { originalBlockId: string; parallelId: string; iterationIndex: number }
      | undefined // Info if this is a virtual block from a parallel.

    // Check if the given `blockId` is a virtual block from a parallel execution.
    if (context.parallelBlockMapping?.has(blockId)) {
      parallelInfo = context.parallelBlockMapping.get(blockId) // Get parallel info.
      actualBlockId = parallelInfo!.originalBlockId // Use the original block ID for handler lookup.

      context.currentVirtualBlockId = blockId // Set current virtual block ID in context for resolver.

      // Set up iteration-specific context for input resolution.
      if (parallelInfo) {
        this.parallelManager.setupIterationContext(context, parallelInfo)
      }
    } else {
      context.currentVirtualBlockId = undefined // Clear for non-virtual blocks.
    }

    const block = this.actualWorkflow.blocks.find((b) => b.id === actualBlockId) // Find the block definition.
    if (!block) {
      throw new Error(`Block ${actualBlockId} not found`)
    }

    // Special case for starter block: it's initialized once. Just return its existing state.
    if (block.metadata?.id === BlockType.STARTER) {
      const starterState = context.blockStates.get(actualBlockId)
      if (starterState) {
        return starterState.output as NormalizedBlockOutput
      }
    }

    const blockLog = this.createBlockLog(block) // Create a new log entry for this block.
    // Adjust log name for virtual blocks (parallels) or blocks inside loops.
    if (parallelInfo) {
      blockLog.blockId = blockId
      blockLog.blockName = `${block.metadata?.name || ''} (iteration ${parallelInfo.iterationIndex + 1})`
    } else {
      const containingLoopId = this.resolver.getContainingLoopId(block.id)
      if (containingLoopId) {
        const currentIteration = context.loopIterations.get(containingLoopId)
        if (currentIteration !== undefined) {
          blockLog.blockName = `${block.metadata?.name || ''} (iteration ${currentIteration})`
        }
      }
    }

    const addConsole = useConsoleStore.getState().addConsole // Get action to add to console logs.

    try {
      if (block.enabled === false) {
        throw new Error(`Cannot execute disabled block: ${block.metadata?.name || block.id}`)
      }

      // Check if starter block output is missing (potential reference issues).
      const starterBlock = this.actualWorkflow.blocks.find(
        (b) => b.metadata?.id === BlockType.STARTER
      )
      if (starterBlock) {
        const starterState = context.blockStates.get(starterBlock.id)
        if (!starterState) {
          logger.warn(
            `Starter block state not found when executing ${block.metadata?.name || actualBlockId}. This may cause reference errors.`
          )
        }
      }

      blockLog.input = block.config.params // Store raw input config for debugging.

      // Resolve actual inputs for the block by evaluating references.
      const inputs = this.resolver.resolveInputs(block, context)

      blockLog.input = inputs // Store resolved inputs in the log.

      // Track block execution start using telemetry.
      trackWorkflowTelemetry('block_execution_start', {
        workflowId: context.workflowId,
        blockId: block.id,
        virtualBlockId: parallelInfo ? blockId : undefined,
        iterationIndex: parallelInfo?.iterationIndex,
        blockType: block.metadata?.id || 'unknown',
        blockName: block.metadata?.name || 'Unnamed Block',
        inputSize: Object.keys(inputs).length,
        startTime: new Date().toISOString(),
      })

      // Find the appropriate handler for this block type.
      const handler = this.blockHandlers.find((h) => h.canHandle(block))
      if (!handler) {
        throw new Error(`No handler found for block type: ${block.metadata?.id}`)
      }

      // Execute the block and measure execution time.
      const startTime = performance.now()
      const rawOutput = await handler.execute(block, inputs, context)
      const executionTime = performance.now() - startTime

      // Remove this block from active blocks immediately after execution (for UI).
      if (!this.isChildExecution) {
        useExecutionStore.setState((state) => {
          const updatedActiveBlockIds = new Set(state.activeBlockIds)
          updatedActiveBlockIds.delete(blockId) // Remove the current block.

          // If it was a virtual block, check if its original block ID should also be removed (if no other virtual instances are active).
          if (parallelInfo) {
            const hasOtherVirtualBlocks = Array.from(state.activeBlockIds).some((activeId) => {
              if (activeId === blockId) return false
              const mapping = context.parallelBlockMapping?.get(activeId)
              return mapping && mapping.originalBlockId === parallelInfo.originalBlockId
            })
            if (!hasOtherVirtualBlocks) {
              updatedActiveBlockIds.delete(parallelInfo.originalBlockId)
            }
          }
          return { activeBlockIds: updatedActiveBlockIds }
        })
      }

      // --- Handle Streaming Output ---
      if (
        rawOutput &&
        typeof rawOutput === 'object' &&
        'stream' in rawOutput &&
        'execution' in rawOutput
      ) {
        const streamingExec = rawOutput as StreamingExecution
        const output = (streamingExec.execution as any).output as NormalizedBlockOutput // Extract the initial output part.

        // Update block state with the initial output.
        context.blockStates.set(blockId, {
          output,
          executed: true,
          executionTime,
        })

        // Store iteration result for parallels and loops.
        if (parallelInfo) {
          this.parallelManager.storeIterationResult(
            context,
            parallelInfo.parallelId,
            parallelInfo.iterationIndex,
            output
          )
        }
        const containingLoopId = this.resolver.getContainingLoopId(block.id)
        if (containingLoopId && !parallelInfo) {
          const currentIteration = context.loopIterations.get(containingLoopId)
          if (currentIteration !== undefined) {
            this.loopManager.storeIterationResult(
              context,
              containingLoopId,
              currentIteration - 1,
              output
            )
          }
        }

        // Update block log.
        blockLog.success = true
        blockLog.output = output
        blockLog.durationMs = Math.round(executionTime)
        blockLog.endedAt = new Date().toISOString()

        this.integrateChildWorkflowLogs(block, output) // Integrate child workflow logs if any.
        context.blockLogs.push(blockLog) // Add log to context.

        // Skip console logging for infrastructure/trigger blocks.
        const blockConfig = getBlock(block.metadata?.id || '')
        const isTriggerBlock =
          blockConfig?.category === 'triggers' || block.metadata?.id === BlockType.STARTER
        if (
          block.metadata?.id !== BlockType.LOOP &&
          block.metadata?.id !== BlockType.PARALLEL &&
          !isTriggerBlock
        ) {
          // Determine iteration context for console display.
          let iterationCurrent: number | undefined
          let iterationTotal: number | undefined
          let iterationType: 'loop' | 'parallel' | undefined
          const blockName = block.metadata?.name || 'Unnamed Block'

          if (parallelInfo) {
            const parallelState = context.parallelExecutions?.get(parallelInfo.parallelId)
            iterationCurrent = parallelInfo.iterationIndex + 1
            iterationTotal = parallelState?.parallelCount
            iterationType = 'parallel'
          } else {
            const containingLoopId = this.resolver.getContainingLoopId(block.id)
            if (containingLoopId) {
              const currentIteration = context.loopIterations.get(containingLoopId)
              const loop = context.workflow?.loops?.[containingLoopId]
              if (currentIteration !== undefined && loop) {
                iterationCurrent = currentIteration
                if (loop.loopType === 'forEach') {
                  const forEachItems = context.loopItems.get(`${containingLoopId}_items`)
                  if (forEachItems) {
                    iterationTotal = Array.isArray(forEachItems)
                      ? forEachItems.length
                      : Object.keys(forEachItems).length
                  }
                } else {
                  iterationTotal = loop.iterations || 5
                }
                iterationType = 'loop'
              }
            }
          }

          addConsole({
            input: blockLog.input,
            output: blockLog.output,
            success: true,
            durationMs: blockLog.durationMs,
            startedAt: blockLog.startedAt,
            endedAt: blockLog.endedAt,
            workflowId: context.workflowId,
            blockId: parallelInfo ? blockId : block.id,
            executionId: this.contextExtensions.executionId,
            blockName,
            blockType: block.metadata?.id || 'unknown',
            iterationCurrent,
            iterationTotal,
            iterationType,
          })
        }

        // Track block execution with telemetry.
        trackWorkflowTelemetry('block_execution', {
          workflowId: context.workflowId,
          blockId: block.id,
          virtualBlockId: parallelInfo ? blockId : undefined,
          iterationIndex: parallelInfo?.iterationIndex,
          blockType: block.metadata?.id || 'unknown',
          blockName: block.metadata?.name || 'Unnamed Block',
          durationMs: Math.round(executionTime),
          success: true,
        })

        return streamingExec // Return the streaming object for further handling by the caller.
      }

      // --- Handle Non-Streaming Output ---
      // Ensure output is always an object.
      const output: NormalizedBlockOutput =
        typeof rawOutput === 'object' && rawOutput !== null ? rawOutput : { result: rawOutput }

      // Update block state with the final output.
      context.blockStates.set(blockId, {
        output,
        executed: true,
        executionTime,
      })

      // Store iteration result for parallels and loops (similar logic as streaming).
      if (parallelInfo) {
        this.parallelManager.storeIterationResult(
          context,
          parallelInfo.parallelId,
          parallelInfo.iterationIndex,
          output
        )
      }
      const containingLoopId = this.resolver.getContainingLoopId(block.id)
      if (containingLoopId && !parallelInfo) {
        const currentIteration = context.loopIterations.get(containingLoopId)
        if (currentIteration !== undefined) {
          this.loopManager.storeIterationResult(
            context,
            containingLoopId,
            currentIteration - 1,
            output
          )
        }
      }

      // Update block log.
      blockLog.success = true
      blockLog.output = output
      blockLog.durationMs = Math.round(executionTime)
      blockLog.endedAt = new Date().toISOString()

      this.integrateChildWorkflowLogs(block, output)
      context.blockLogs.push(blockLog)

      // Skip console logging for infrastructure/trigger blocks.
      const nonStreamBlockConfig = getBlock(block.metadata?.id || '')
      const isNonStreamTriggerBlock =
        nonStreamBlockConfig?.category === 'triggers' || block.metadata?.id === BlockType.STARTER
      if (
        block.metadata?.id !== BlockType.LOOP &&
        block.metadata?.id !== BlockType.PARALLEL &&
        !isNonStreamTriggerBlock
      ) {
        // Determine iteration context for console display.
        let iterationCurrent: number | undefined
        let iterationTotal: number | undefined
        let iterationType: 'loop' | 'parallel' | undefined
        const blockName = block.metadata?.name || 'Unnamed Block'

        if (parallelInfo) {
          const parallelState = context.parallelExecutions?.get(parallelInfo.parallelId)
          iterationCurrent = parallelInfo.iterationIndex + 1
          iterationTotal = parallelState?.parallelCount
          iterationType = 'parallel'
        } else {
          const containingLoopId = this.resolver.getContainingLoopId(block.id)
          if (containingLoopId) {
            const currentIteration = context.loopIterations.get(containingLoopId)
            const loop = context.workflow?.loops?.[containingLoopId]
            if (currentIteration !== undefined && loop) {
              iterationCurrent = currentIteration
              if (loop.loopType === 'forEach') {
                const forEachItems = context.loopItems.get(`${containingLoopId}_items`)
                if (forEachItems) {
                  iterationTotal = Array.isArray(forEachItems)
                    ? forEachItems.length
                    : Object.keys(forEachItems).length
                }
              } else {
                iterationTotal = loop.iterations || 5
              }
              iterationType = 'loop'
            }
          }
        }

        addConsole({
          input: blockLog.input,
          output: blockLog.output,
          success: true,
          durationMs: blockLog.durationMs,
          startedAt: blockLog.startedAt,
          endedAt: blockLog.endedAt,
          workflowId: context.workflowId,
          blockId: parallelInfo ? blockId : block.id,
          executionId: this.contextExtensions.executionId,
          blockName,
          blockType: block.metadata?.id || 'unknown',
          iterationCurrent,
          iterationTotal,
          iterationType,
        })
      }

      // Track block execution with telemetry.
      trackWorkflowTelemetry('block_execution', {
        workflowId: context.workflowId,
        blockId: block.id,
        virtualBlockId: parallelInfo ? blockId : undefined,
        iterationIndex: parallelInfo?.iterationIndex,
        blockType: block.metadata?.id || 'unknown',
        blockName: block.metadata?.name || 'Unnamed Block',
        durationMs: Math.round(executionTime),
        success: true,
      })

      if (context.onBlockComplete && !isNonStreamTriggerBlock) {
        try {
          await context.onBlockComplete(blockId, output)
        } catch (callbackError: any) {
          logger.error('Error in onBlockComplete callback:', callbackError)
        }
      }

      return output
    } catch (error: any) {
      // --- Error Handling for a single block ---
      // Remove block from active blocks if an error occurs.
      if (!this.isChildExecution) {
        useExecutionStore.setState((state) => {
          const updatedActiveBlockIds = new Set(state.activeBlockIds)
          updatedActiveBlockIds.delete(blockId)
          if (parallelInfo) {
            const hasOtherVirtualBlocks = Array.from(state.activeBlockIds).some((activeId) => {
              if (activeId === blockId) return false
              const mapping = context.parallelBlockMapping?.get(activeId)
              return mapping && mapping.originalBlockId === parallelInfo.originalBlockId
            })
            if (!hasOtherVirtualBlocks) {
              updatedActiveBlockIds.delete(parallelInfo.originalBlockId)
            }
          }
          return { activeBlockIds: updatedActiveBlockIds }
        })
      }

      // Update block log with error details.
      blockLog.success = false
      blockLog.error =
        error.message ||
        `Error executing ${block.metadata?.id || 'unknown'} block: ${String(error)}`
      blockLog.endedAt = new Date().toISOString()
      blockLog.durationMs =
        new Date(blockLog.endedAt).getTime() - new Date(blockLog.startedAt).getTime()

      // If error came from child workflow, attach its trace spans.
      if (isWorkflowBlockType(block.metadata?.id)) {
        this.attachChildWorkflowSpansToLog(blockLog, error)
      }

      context.blockLogs.push(blockLog) // Add error log to context.

      // Skip console logging for infrastructure/trigger blocks.
      const errorBlockConfig = getBlock(block.metadata?.id || '')
      const isErrorTriggerBlock =
        errorBlockConfig?.category === 'triggers' || block.metadata?.id === BlockType.STARTER
      if (
        block.metadata?.id !== BlockType.LOOP &&
        block.metadata?.id !== BlockType.PARALLEL &&
        !isErrorTriggerBlock
      ) {
        // Determine iteration context for console display.
        let iterationCurrent: number | undefined
        let iterationTotal: number | undefined
        let iterationType: 'loop' | 'parallel' | undefined
        const blockName = block.metadata?.name || 'Unnamed Block'

        if (parallelInfo) {
          const parallelState = context.parallelExecutions?.get(parallelInfo.parallelId)
          iterationCurrent = parallelInfo.iterationIndex + 1
          iterationTotal = parallelState?.parallelCount
          iterationType = 'parallel'
        } else {
          const containingLoopId = this.resolver.getContainingLoopId(block.id)
          if (containingLoopId) {
            const currentIteration = context.loopIterations.get(containingLoopId)
            const loop = context.workflow?.loops?.[containingLoopId]
            if (currentIteration !== undefined && loop) {
              iterationCurrent = currentIteration
              if (loop.loopType === 'forEach') {
                const forEachItems = context.loopItems.get(`${containingLoopId}_items`)
                if (forEachItems) {
                  iterationTotal = Array.isArray(forEachItems)
                    ? forEachItems.length
                    : Object.keys(forEachItems).length
                }
              } else {
                iterationTotal = loop.iterations || 5
              }
              iterationType = 'loop'
            }
          }
        }

        addConsole({
          input: blockLog.input,
          output: {}, // Output is empty on error.
          success: false,
          error:
            error.message ||
            `Error executing ${block.metadata?.id || 'unknown'} block: ${String(error)}`,
          durationMs: blockLog.durationMs,
          startedAt: blockLog.startedAt,
          endedAt: blockLog.endedAt,
          workflowId: context.workflowId,
          blockId: parallelInfo ? blockId : block.id,
          executionId: this.contextExtensions.executionId,
          blockName,
          blockType: block.metadata?.id || 'unknown',
          iterationCurrent,
          iterationTotal,
          iterationType,
        })
      }

      // Check for and activate any error paths connected to this block.
      const hasErrorPath = this.activateErrorPath(actualBlockId, context)

      logger.error(
        `Error executing block ${block.metadata?.name || actualBlockId}:`,
        this.sanitizeError(error)
      )

      // Create an error output object.
      const errorOutput: NormalizedBlockOutput = {
        error: this.extractErrorMessage(error),
        status: error.status || 500,
      }

      // Preserve child workflow spans on the block state for downstream logging.
      if (isWorkflowBlockType(block.metadata?.id)) {
        this.attachChildWorkflowSpansToOutput(errorOutput, error)
      }

      // Set block state with error output.
      context.blockStates.set(blockId, {
        output: errorOutput,
        executed: true,
        executionTime: blockLog.durationMs,
      })

      // Update metadata with failure details.
      const failureEndTime = context.metadata.endTime ?? new Date().toISOString()
      if (!context.metadata.endTime) {
        context.metadata.endTime = failureEndTime
      }
      const failureDuration = context.metadata.startTime
        ? Math.max(
            0,
            new Date(failureEndTime).getTime() - new Date(context.metadata.startTime).getTime()
          )
        : (context.metadata.duration ?? 0)
      context.metadata.duration = failureDuration

      const failureMetadata = {
        ...context.metadata,
        endTime: failureEndTime,
        duration: failureDuration,
        workflowConnections: this.actualWorkflow.connections.map((conn) => ({
          source: conn.source,
          target: conn.target,
        })),
      }

      const upstreamExecutionResult = (error as { executionResult?: ExecutionResult } | null)
        ?.executionResult
      const executionResultPayload: ExecutionResult = {
        success: false,
        output: upstreamExecutionResult?.output ?? errorOutput,
        error: upstreamExecutionResult?.error ?? this.extractErrorMessage(error),
        logs: [...context.blockLogs],
        metadata: {
          ...failureMetadata,
          ...(upstreamExecutionResult?.metadata ?? {}),
          workflowConnections: failureMetadata.workflowConnections,
        },
      }

      if (hasErrorPath) {
        return errorOutput // If an error path exists, return the error output to allow workflow to continue.
      }

      // If no error path, create a new error object for propagation.
      let errorMessage = error.message
      if (!errorMessage || errorMessage === 'undefined (undefined)') {
        errorMessage = `Error executing ${block.metadata?.id || 'unknown'} block: ${block.metadata?.name || 'Unnamed Block'}`
        if (error && typeof error === 'object') {
          if (error.code) errorMessage += ` (code: ${error.code})`
          if (error.status) errorMessage += ` (status: ${error.status})`
          if (error.type) errorMessage += ` (type: ${error.type})`
        }
      }

      // Track block execution error with telemetry.
      trackWorkflowTelemetry('block_execution_error', {
        workflowId: context.workflowId,
        blockId: block.id,
        virtualBlockId: parallelInfo ? blockId : undefined,
        iterationIndex: parallelInfo?.iterationIndex,
        blockType: block.metadata?.id || 'unknown',
        blockName: block.metadata?.name || 'Unnamed Block',
        durationMs: blockLog.durationMs,
        errorType: error.name || 'Error',
        errorMessage: this.extractErrorMessage(error),
      })

      // Attach execution result payload and child trace spans to the error object.
      const executionError = new Error(errorMessage)
      ;(executionError as any).executionResult = executionResultPayload
      if (Array.isArray((error as { childTraceSpans?: TraceSpan[] } | null)?.childTraceSpans)) {
        ;(executionError as any).childTraceSpans = (
          error as { childTraceSpans?: TraceSpan[] }
        ).childTraceSpans
        ;(executionError as any).childWorkflowName = (
          error as { childWorkflowName?: string }
        ).childWorkflowName
      }
      throw executionError // Re-throw the enhanced error.
    }
  }

  /**
   * Copies child workflow trace spans from an error object into a block log.
   * Ensures consistent structure and avoids duplication of inline guards.
   */
  private attachChildWorkflowSpansToLog(blockLog: BlockLog, error: unknown): void {
    const spans = (
      error as { childTraceSpans?: TraceSpan[]; childWorkflowName?: string } | null | undefined
    )?.childTraceSpans
    if (Array.isArray(spans) && spans.length > 0) {
      const childWorkflowName = (error as { childWorkflowName?: string } | null | undefined)
        ?.childWorkflowName
      // Adds child trace spans and workflow name to the block log's output.
      blockLog.output = {
        ...(blockLog.output || {}),
        childTraceSpans: spans,
        childWorkflowName,
      }
    }
  }

  /**
   * Copies child workflow trace spans from an error object into a normalized output.
   */
  private attachChildWorkflowSpansToOutput(output: NormalizedBlockOutput, error: unknown): void {
    const spans = (
      error as { childTraceSpans?: TraceSpan[]; childWorkflowName?: string } | null | undefined
    )?.childTraceSpans
    if (Array.isArray(spans) && spans.length > 0) {
      // Directly adds child trace spans and workflow name to the normalized output.
      output.childTraceSpans = spans
      output.childWorkflowName = (
        error as { childWorkflowName?: string } | null | undefined
      )?.childWorkflowName
    }
  }

  /**
   * Activates error paths from a block that had an error.
   * Checks for connections from the block's "error" handle and adds them to the active execution path.
   *
   * @param blockId - ID of the block that had an error
   * @param context - Current execution context
   * @returns Whether there was an error path to follow
   */
  private activateErrorPath(blockId: string, context: ExecutionContext): boolean {
    // Skip specific block types that don't have error handles or handle errors internally.
    const block = this.actualWorkflow.blocks.find((b) => b.id === blockId)
    if (
      block?.metadata?.id === BlockType.STARTER ||
      block?.metadata?.id === BlockType.CONDITION ||
      block?.metadata?.id === BlockType.LOOP ||
      block?.metadata?.id === BlockType.PARALLEL
    ) {
      return false
    }

    // Find connections originating from this block's 'error' handle.
    const errorConnections = this.actualWorkflow.connections.filter(
      (conn) => conn.source === blockId && conn.sourceHandle === 'error'
    )

    if (errorConnections.length === 0) {
      return false // No error paths found.
    }

    // Add targets of error connections to the active execution path.
    for (const conn of errorConnections) {
      context.activeExecutionPath.add(conn.target)
      logger.info(`Activated error path from ${blockId} to ${conn.target}`)
    }

    return true // An error path was activated.
  }

  /**
   * Creates a new block log entry with initial values.
   *
   * @param block - Block to create log for
   * @returns Initialized block log
   */
  private createBlockLog(block: SerializedBlock): BlockLog {
    return {
      blockId: block.id,
      blockName: block.metadata?.name || '',
      blockType: block.metadata?.id || '',
      startedAt: new Date().toISOString(),
      endedAt: '',
      durationMs: 0,
      success: false,
    }
  }

  /**
   * Extracts a meaningful error message from any error object structure.
   * Handles nested error objects, undefined messages, and various error formats.
   *
   * @param error - The error object to extract a message from
   * @returns A meaningful error message string
   */
  private extractErrorMessage(error: any): string {
    if (typeof error === 'string') {
      return error
    }
    if (error.message) {
      return error.message
    }
    // If it's an object with a `response.data` structure (common in HTTP errors).
    if (error.response?.data) {
      const data = error.response.data
      if (typeof data === 'string') {
        return data
      }
      if (data.message) {
        return data.message
      }
      return JSON.stringify(data)
    }
    if (typeof error === 'object') {
      return JSON.stringify(error) // Fallback to stringifying the object.
    }
    return String(error) // Final fallback.
  }

  /**
   * Sanitizes an error object for logging purposes.
   * Ensures the error is in a format that won't cause "undefined" to appear in logs.
   * This is similar to `extractErrorMessage` but might return a slightly different representation
   * suitable for logging entire error objects.
   *
   * @param error - The error object to sanitize
   * @returns A sanitized version of the error for logging
   */
  private sanitizeError(error: any): any {
    if (typeof error === 'string') {
      return error
    }
    if (error.message) {
      return error.message
    }
    if (error.response?.data) {
      const data = error.response.data
      if (typeof data === 'string') {
        return data
      }
      if (data.message) {
        return data.message
      }
      return JSON.stringify(data)
    }
    if (typeof error === 'object') {
      return JSON.stringify(error)
    }
    return String(error)
  }

  /**
   * Creates a block log for the starter block if it contains files.
   * This ensures files are captured in trace spans and execution logs.
   */
  private createStartedBlockWithFilesLog(
    initBlock: SerializedBlock,
    blockOutput: any,
    context: ExecutionContext
  ): void {
    // If the starter block's output contains files, create a log entry for it.
    if (blockOutput.files && Array.isArray(blockOutput.files) && blockOutput.files.length > 0) {
      const starterBlockLog: BlockLog = {
        blockId: initBlock.id,
        blockName: initBlock.metadata?.name || 'Start',
        blockType: initBlock.metadata?.id || 'start',
        startedAt: new Date().toISOString(),
        endedAt: new Date().toISOString(),
        success: true,
        input: this.workflowInput, // The raw workflow input.
        output: blockOutput, // The processed output containing files.
        durationMs: 0, // Starter block has no real execution time.
      }
      context.blockLogs.push(starterBlockLog)
    }
  }

  /**
   * Preserves child workflow trace spans for proper nesting
   * This method appears to be a stub or primarily intended to integrate child workflow trace spans
   * that might be part of the `output` from a `WorkflowBlockHandler`.
   */
  private integrateChildWorkflowLogs(block: SerializedBlock, output: NormalizedBlockOutput): void {
    // Only applies to blocks that are themselves workflow blocks (i.e., child workflows).
    if (!isWorkflowBlockType(block.metadata?.id)) {
      return
    }

    // If output doesn't contain child trace spans, nothing to integrate.
    if (!output || typeof output !== 'object' || !output.childTraceSpans) {
      return
    }

    const childTraceSpans = output.childTraceSpans as TraceSpan[]
    if (!Array.isArray(childTraceSpans) || childTraceSpans.length === 0) {
      return
    }
    // (Actual integration logic would go here, e.g., adding them to the global trace or processing them)
    // The current implementation ensures they are part of the block's output and thus logged.
  }
}
```

**Simplified Explanation of the `Executor` Class:**

1.  **Properties:** The class holds various managers (`LoopManager`, `ParallelManager`), resolvers (`InputResolver`), and trackers (`PathTracker`) as internal tools. It also stores the `actualWorkflow`, initial `workflowInput`, environment variables, and flags like `isDebugging` and `isCancelled`.
2.  **Constructor:** It accepts a workflow definition (and optional initial states/inputs). It initializes all the internal managers and an array of `BlockHandler` instances (one for each type of block it can execute). It also performs initial workflow validation.
3.  **`cancel()`:** A simple method to stop ongoing execution gracefully by setting a flag.
4.  **`execute()` (The Heartbeat):**
    *   This is the main entry point to run a workflow.
    *   It sets up the initial `ExecutionContext` (a central object holding all execution-time data like block outputs, logs, decisions, and active paths).
    *   It enters a `while` loop that continues as long as there are blocks ready to run and the workflow hasn't been cancelled or hit an iteration limit.
    *   **Debug Mode:** If debugging, it pauses and returns the state, waiting for user input to `continueExecution`.
    *   **Normal Mode:**
        *   It identifies the `nextLayer` of blocks that are ready to execute based on their dependencies.
        *   It then executes these blocks concurrently using `executeLayer()`.
        *   It specifically handles streaming outputs, including processing the stream for both client display and internal state updates.
        *   After a layer, it updates the state of loops and parallels (`processLoopIterations`, `processParallelIterations`).
        *   The loop continues until no more blocks are ready.
    *   It includes comprehensive `try...catch...finally` blocks for error handling, logging, and telemetry reporting (for starts, completions, and failures).
5.  **`continueExecution()`:** Used in debug mode to run a specified set of blocks and then re-evaluate the next layer, allowing for step-by-step execution.
6.  **`validateWorkflow()`:** Checks the structural integrity of the workflow, ensuring a starting point (either a `STARTER` block or a `Trigger` block), valid connections, and correct loop configurations.
7.  **`createExecutionContext()`:** Builds the initial `ExecutionContext` map. It's responsible for finding the actual starting block (which can be a legacy "Starter" block, an API/Chat/Input trigger, or a specific `startBlockId` for webhooks/schedules) and processing its initial input based on defined `inputFormat` schemas.
8.  **`getNextExecutionLayer()`:** This critical method determines which blocks are ready to run in the current "layer." It iterates through all blocks, checking dependencies and `activeExecutionPath`. It has specialized logic for handling blocks within parallel executions, creating "virtual" block IDs for each iteration.
9.  **`processParallelBlocks()` & Related:** These methods manage the complex coordination of parallel execution.
    *   `processParallelBlocks` orchestrates multiple parallel *iterations*.
    *   `isIterationComplete` checks if all *expected* blocks within a specific parallel iteration have finished.
    *   `getExpectedBlocksForIteration` figures out which blocks in a parallel *should* run, considering conditional routing *within* the parallel.
    *   `shouldBlockExecuteInParallelIteration` checks if a specific block within a parallel iteration is on an active path based on decisions.
    *   `hasMoreParallelWork` tells the main loop if any parallel work is still pending.
    *   `processParallelIteration` builds a dependency graph for a *single* parallel iteration and adds ready blocks (with virtual IDs) to the `pendingBlocks` queue.
10. **`checkDependencies()`:** This is central to the topological sort. It evaluates if a block's incoming connections (dependencies) are met. It has specific logic for:
    *   Regular connections (source executed without error).
    *   `error` connections (source executed *with* an error).
    *   `loop-start/end` and `parallel-start/end` connections.
    *   `Condition` blocks (checking if the source block chose this specific path).
    *   `Router` blocks (checking if the source block chose this specific target).
    *   Crucially, it handles "virtual" block IDs for blocks within parallel executions.
11. **`executeLayer()`:** Takes a list of ready `blockIds` and runs them in parallel using `Promise.allSettled`. It updates the UI's active blocks, marks blocks as executed, and uses `pathTracker` to update the `activeExecutionPath` based on routing decisions.
12. **`executeBlock()`:** The core logic for running a *single* block:
    *   It handles "virtual" block IDs for parallel execution, setting up context specific to that iteration.
    *   It creates a `BlockLog` entry to record details.
    *   It resolves the block's inputs using `InputResolver`.
    *   It finds the correct `BlockHandler` for the block's type.
    *   It calls `handler.execute()` and measures the time.
    *   It updates the `ExecutionContext` with the block's output and marks it as executed.
    *   It stores iteration results for `LoopManager` and `ParallelManager`.
    *   It pushes log entries to the console store.
    *   It sends telemetry events for block start and completion.
    *   **Error Handling:** If a block fails, it logs the error, updates the block's state with an error output, and attempts to `activateErrorPath()` to reroute execution if "error" connections exist. If no error path, it throws an enhanced error object.
13. **`attachChildWorkflowSpansToLog`/`Output()`:** Methods to correctly associate trace spans from child workflows (when a workflow block runs another workflow) with the parent workflow's logs or output.
14. **`activateErrorPath()`:** A helper that finds outgoing "error" connections from a failed block and adds their target blocks to the `activeExecutionPath`, allowing the workflow to continue along an error handling branch.
15. **`createBlockLog()`:** A utility to create an initial log entry for a block.
16. **`extractErrorMessage()` / `sanitizeError()`:** Robust utilities to get a clean, human-readable error message from various error object structures for logging and display.
17. **`createStartedBlockWithFilesLog()`:** Specific logic to ensure that if the initial trigger block receives files, those are properly recorded in the execution logs.
18. **`integrateChildWorkflowLogs()`:** A method intended for processing trace spans from child workflows, primarily ensuring they are correctly reflected in the overall execution tracing.

In essence, the `Executor` class is a sophisticated graph traversal and state management system that brings workflow definitions to life, handling complexity, errors, and various execution models.