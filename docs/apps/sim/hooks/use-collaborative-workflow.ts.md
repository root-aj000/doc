This TypeScript file defines a React hook named `useCollaborativeWorkflow` that provides the core logic for enabling real-time collaboration on a workflow editor. It integrates various state management solutions (Zustand stores), WebSocket communication, and an undo/redo system to create a seamless collaborative experience.

## Purpose of this File

The primary purpose of `useCollaborativeWorkflow` is to:

1.  **Synchronize Workflow Changes:** Allow multiple users to concurrently edit a workflow (add/remove blocks, move blocks, connect edges, update sub-block values, manage variables, configure subflows like loops/parallels) in real-time.
2.  **Optimistic UI with Reliability:** Apply user changes locally immediately (optimistic UI) while queuing them for transmission to other collaborators. It handles confirmations and retries for operations to ensure eventual consistency.
3.  **Undo/Redo Functionality:** Integrate a robust undo/redo system that works correctly even in a collaborative environment, carefully managing which actions are recorded and when.
4.  **Conflict Resolution (Partial):** Implement mechanisms like `lastPositionTimestamps` to prevent out-of-order updates for frequently changing properties like block positions, and `isApplyingRemoteChange` to avoid infinite loops when applying changes from other users.
5.  **Event Handling:** Listen for local UI events to record actions for undo/redo and subscribe to WebSocket events to receive and apply remote changes.

In essence, it's the central nervous system for collaborative workflow editing, handling local state, remote synchronization, and historical tracking of changes.

## Simplified Explanation of Complex Logic

The file juggles several complex concerns to achieve collaboration:

1.  **Local vs. Remote Changes:**
    *   When *you* make a change (e.g., drag a block), it first updates your local UI immediately (`workflowStore.updateBlockPosition`). This is "optimistic UI" – it feels fast.
    *   Then, this change is put into an "operation queue" (`useOperationQueue`). This queue is like a reliable to-do list. Each item is sent to the server via WebSockets.
    *   When the server confirms your operation, it's removed from your queue. If it fails, it might be retried.
    *   When *another user* makes a change, your client receives it via WebSockets. The `isApplyingRemoteChange` flag is set to `true` to ensure your client doesn't try to send this remote change back to the server (preventing an infinite loop). The change is then applied to your local state.

2.  **Order of Operations:**
    *   For things like block positions, which can change very rapidly, simply applying updates in any order can lead to "jagged" or incorrect visual states. The `lastPositionTimestamps` ref stores the timestamp of the last *successful* position update for each block. If a new update comes in with an older timestamp, it's ignored. This prioritizes newer updates, making movements smoother.

3.  **Undo/Redo in Collaboration:**
    *   The `useUndoRedo` hook manages your personal undo/redo stack.
    *   Crucially, when you perform an undo or redo, the `isUndoRedoInProgress` flag is temporarily set to `true`. This prevents the undo/redo action itself from being recorded *again* into the undo stack, which would lead to circular references and incorrect behavior.
    *   When a block or edge is *removed*, the undo/redo stacks of *all* users are "pruned" (`undoRedoStore.pruneInvalidEntries`) to remove any actions that referenced the now-deleted items. This keeps the undo/redo history clean and valid.

4.  **The Operation Queue:**
    *   Instead of just "emitting" every change directly via WebSockets, many operations go through `useOperationQueue`. This provides robustness:
        *   **Retry Mechanism:** If a network request fails, the queue can retry sending the operation.
        *   **Ordering:** Ensures operations are processed in the order they were intended.
        *   **Confirmation:** Operations are only considered "done" when the server explicitly confirms them. This helps maintain consistency.
        *   Some high-frequency operations (like block movement, when not committed) or declarative subblock clearing might bypass the full queue for immediate emission or direct local application, balancing responsiveness with strict consistency.

5.  **Subflow Configuration (Loops/Parallels):**
    *   When a user changes settings for a "loop" or "parallel" block, the component needs to identify all child blocks within that subflow. This information is bundled into the payload sent to the server, ensuring the server has the context for the subflow update.

## Line-by-Line Explanation

```typescript
// Import necessary React hooks for state management and side effects
import { useCallback, useEffect, useRef } from 'react' 
// Type definition for an Edge in the React Flow library
import type { Edge } from 'reactflow'
// Custom hook for accessing user session data (e.g., current user's ID)
import { useSession } from '@/lib/auth-client'
// Utility for creating a logger instance for console output
import { createLogger } from '@/lib/logs/console/logger'
// Functions to get output configurations for different block types
import { getBlockOutputs } from '@/lib/workflows/block-outputs'
// Function to retrieve the configuration of a specific block type
import { getBlock } from '@/blocks'
// Utility to resolve output types, likely for block configuration
import { resolveOutputType } from '@/blocks/utils'
// Context hook for accessing WebSocket connection and related functions
import { useSocket } from '@/contexts/socket-context'
// Custom hook for managing local undo/redo stacks
import { useUndoRedo } from '@/hooks/use-undo-redo'
// Functions and hook for interacting with the operation queue store (Zustand)
import { registerEmitFunctions, useOperationQueue } from '@/stores/operation-queue/store'
// Hook for accessing and manipulating variables state (Zustand)
import { useVariablesStore } from '@/stores/panel/variables/store'
// Hook for accessing and manipulating the global undo/redo store (Zustand)
import { useUndoRedoStore } from '@/stores/undo-redo'
// Hook for accessing state related to workflow differences (e.g., diff mode)
import { useWorkflowDiffStore } from '@/stores/workflow-diff/store'
// Hook for accessing the workflow registry (list of available workflows)
import { useWorkflowRegistry } from '@/stores/workflows/registry/store'
// Hook for accessing and manipulating sub-block state (Zustand)
import { useSubBlockStore } from '@/stores/workflows/subblock/store'
// Utility functions for workflow management, like generating unique names and merging states
import { getUniqueBlockName, mergeSubblockState } from '@/stores/workflows/utils'
// Hook for accessing and manipulating the main workflow state (blocks, edges, etc.) (Zustand)
import { useWorkflowStore } from '@/stores/workflows/workflow/store'
// Type definitions for a block's state and a position in the workflow editor
import type { BlockState, Position } from '@/stores/workflows/workflow/types'

// Initialize a logger instance specifically for this collaborative workflow module.
// This helps in distinguishing log messages from different parts of the application.
const logger = createLogger('CollaborativeWorkflow')

// Define the main custom React hook for collaborative workflow management.
export function useCollaborativeWorkflow() {
  // Initialize the undo/redo manager. This hook provides functions to record and perform undo/redo actions.
  const undoRedo = useUndoRedo()
  // A ref to track if an undo/redo operation is currently in progress.
  // This is crucial to prevent recording undo/redo actions themselves into the undo stack.
  const isUndoRedoInProgress = useRef(false)
  // A ref to temporarily skip recording certain edge operations (e.g., when moving a parent block).
  const skipEdgeRecording = useRef(false)

  // useEffect hook to set up and clean up global event listeners.
  // These listeners capture specific UI events related to workflow changes that need to be recorded for undo/redo.
  useEffect(() => {
    // Handler for the 'workflow-record-move' event. This event is dispatched when a block is moved.
    const moveHandler = (e: any) => {
      // Destructure event details to get block ID, and its position before and after the move.
      const { blockId, before, after } = e.detail || {}
      // If essential data is missing, do nothing.
      if (!blockId || !before || !after) return
      // IMPORTANT: Don't record moves if an undo/redo operation is currently active.
      if (isUndoRedoInProgress.current) return
      // Record the block movement for undo/redo.
      undoRedo.recordMove(blockId, before, after)
    }

    // Handler for the 'workflow-record-parent-update' event. This event is dispatched when a block's parent changes.
    const parentUpdateHandler = (e: any) => {
      // Destructure event details including block ID, old/new parent IDs, positions, and affected edges.
      const { blockId, oldParentId, newParentId, oldPosition, newPosition, affectedEdges } =
        e.detail || {}
      // If block ID is missing, do nothing.
      if (!blockId) return
      // IMPORTANT: Don't record parent updates if an undo/redo operation is currently active.
      if (isUndoRedoInProgress.current) return
      // Record the parent update for undo/redo.
      undoRedo.recordUpdateParent(
        blockId,
        oldParentId,
        newParentId,
        oldPosition,
        newPosition,
        affectedEdges
      )
    }

    // Handler for the 'skip-edge-recording' event. This event can temporarily disable edge recording.
    const skipEdgeHandler = (e: any) => {
      const { skip } = e.detail || {}
      // Update the ref to control whether edge recording should be skipped.
      skipEdgeRecording.current = skip
    }

    // Add event listeners to the window object.
    window.addEventListener('workflow-record-move', moveHandler)
    window.addEventListener('workflow-record-parent-update', parentUpdateHandler)
    window.addEventListener('skip-edge-recording', skipEdgeHandler)

    // Cleanup function: remove event listeners when the component unmounts or dependencies change.
    return () => {
      window.removeEventListener('workflow-record-move', moveHandler)
      window.removeEventListener('workflow-record-parent-update', parentUpdateHandler)
      window.removeEventListener('skip-edge-recording', skipEdgeHandler)
    }
  }, [undoRedo]) // Dependency array: re-run effect if `undoRedo` instance changes (unlikely for a hook).

  // Destructure various state and functions from the useSocket hook.
  // This hook manages the WebSocket connection and real-time communication.
  const {
    isConnected, // Boolean: Is the socket connected?
    currentWorkflowId, // String: ID of the workflow currently joined via socket.
    presenceUsers, // Array: List of other users currently in the same workflow room.
    joinWorkflow, // Function: To join a specific workflow room.
    leaveWorkflow, // Function: To leave the current workflow room.
    emitWorkflowOperation, // Function: To send a generic workflow operation to the server.
    emitSubblockUpdate, // Function: To send a sub-block value update to the server.
    emitVariableUpdate, // Function: To send a variable update to the server.
    onWorkflowOperation, // Function: To register a handler for incoming workflow operations.
    onSubblockUpdate, // Function: To register a handler for incoming sub-block updates.
    onVariableUpdate, // Function: To register a handler for incoming variable updates.
    onUserJoined, // Function: To register a handler for when another user joins.
    onUserLeft, // Function: To register a handler for when another user leaves.
    onWorkflowDeleted, // Function: To register a handler for when the workflow is deleted.
    onWorkflowReverted, // Function: To register a handler for when the workflow is reverted.
    onOperationConfirmed, // Function: To register a handler for when a sent operation is confirmed by the server.
    onOperationFailed, // Function: To register a handler for when a sent operation fails.
  } = useSocket()

  // Get the active workflow ID from the workflow registry store. This is the ID of the workflow
  // currently loaded in the UI, which might be different from `currentWorkflowId` if, for example,
  // the user navigates away before the socket connection updates.
  const { activeWorkflowId } = useWorkflowRegistry()
  // Get the workflow store instance for managing blocks, edges, etc.
  const workflowStore = useWorkflowStore()
  // Get the sub-block store instance for managing values within blocks.
  const subBlockStore = useSubBlockStore()
  // Get the variables store instance for managing global workflow variables.
  const variablesStore = useVariablesStore()
  // Get session data, primarily for the current user's ID.
  const { data: session } = useSession()
  // Get state from the workflow diff store, specifically if diff mode is active.
  const { isShowingDiff } = useWorkflowDiffStore()

  // A ref to track if a remote change is currently being applied to the local state.
  // This prevents an infinite loop where applying a remote change inadvertently triggers
  // a local change handler that then tries to emit the same change back.
  const isApplyingRemoteChange = useRef(false)

  // A ref to store the timestamp of the last applied position update for each block.
  // This is used to prevent "jagged" movements or out-of-order updates when multiple
  // users are moving blocks simultaneously, ensuring only the newest updates are applied.
  const lastPositionTimestamps = useRef<Map<string, number>>(new Map())

  // Destructure functions and state from the operation queue store.
  // This queue manages optimistic updates and ensures reliable delivery of operations.
  const {
    queue, // The current list of pending operations.
    hasOperationError, // Boolean: Does the queue have any failed operations?
    addToQueue, // Function: To add a new operation to the queue.
    confirmOperation, // Function: To mark an operation in the queue as confirmed.
    failOperation, // Function: To mark an operation in the queue as failed.
    cancelOperationsForBlock, // Function: To cancel pending operations related to a specific block.
    cancelOperationsForVariable, // Function: To cancel pending operations related to a specific variable.
  } = useOperationQueue()

  // useCallback memoizes this function, preventing unnecessary re-renders.
  // It checks if the current workflow (from the socket) matches the active workflow (from the UI).
  const isInActiveRoom = useCallback(() => {
    return !!currentWorkflowId && activeWorkflowId === currentWorkflowId
  }, [currentWorkflowId, activeWorkflowId]) // Dependencies: re-create if these IDs change.

  // useEffect hook to handle workflow switching.
  useEffect(() => {
    // If there's an active workflow and it's different from the one the socket is currently in,
    // it indicates a workflow switch.
    if (activeWorkflowId && currentWorkflowId !== activeWorkflowId) {
      // Log the workflow change for debugging.
      logger.info(`Active workflow changed to: ${activeWorkflowId}`, {
        isConnected,
        currentWorkflowId,
        activeWorkflowId,
        presenceUsers: presenceUsers.length,
      })

      // When switching workflows, clear the stored position timestamps to prevent applying
      // old updates from a previous workflow to the new one.
      lastPositionTimestamps.current.clear()
    }
  }, [activeWorkflowId, isConnected, currentWorkflowId]) // Dependencies: re-run if these values change.

  // useEffect hook to register the socket's emit functions with the operation queue store.
  // This allows the operation queue to use the socket to send its operations.
  useEffect(() => {
    registerEmitFunctions(
      emitWorkflowOperation,
      emitSubblockUpdate,
      emitVariableUpdate,
      currentWorkflowId
    )
  }, [emitWorkflowOperation, emitSubblockUpdate, emitVariableUpdate, currentWorkflowId]) // Dependencies: re-run if emit functions or workflow ID change.

  // The main useEffect hook responsible for listening to incoming WebSocket events.
  // This is where remote collaborative changes are received and applied to the local state.
  useEffect(() => {
    // Handler for generic workflow operations (add/remove blocks, update names, etc.)
    const handleWorkflowOperation = (data: any) => {
      // Destructure operation details: type of operation, target (block/edge/subflow/variable), payload, and user ID.
      const { operation, target, payload, userId } = data

      // If we are currently applying a *remote* change, return early to prevent reprocessing
      // the same change or triggering unintended local side effects.
      if (isApplyingRemoteChange.current) return

      // Log the received operation for debugging.
      logger.info(`Received ${operation} on ${target} from user ${userId}`)

      // Set the flag to true to indicate that we are now applying a remote change.
      isApplyingRemoteChange.current = true

      try {
        // Handle operations based on the 'target' of the operation.
        if (target === 'block') {
          // Process different 'operation' types for blocks.
          switch (operation) {
            case 'add':
              // Add a new block to the workflow store.
              workflowStore.addBlock(
                payload.id,
                payload.type,
                payload.name,
                payload.position,
                payload.data,
                payload.parentId,
                payload.extent,
                {
                  enabled: payload.enabled,
                  horizontalHandles: payload.horizontalHandles,
                  isWide: payload.isWide,
                  advancedMode: payload.advancedMode,
                  triggerMode: payload.triggerMode ?? false, // Default to false if undefined
                  height: payload.height,
                }
              )
              // If an auto-connect edge is part of the payload (e.g., when adding a block with a connection), add it.
              if (payload.autoConnectEdge) {
                workflowStore.addEdge(payload.autoConnectEdge)
              }
              // If sub-block values are provided in the payload (e.g., for a duplicated block), apply them.
              if (payload.subBlocks && typeof payload.subBlocks === 'object') {
                Object.entries(payload.subBlocks).forEach(([subblockId, subblock]) => {
                  const value = (subblock as any)?.value
                  // Only set value if it's explicitly defined (not undefined/null).
                  if (value !== undefined && value !== null) {
                    subBlockStore.setValue(payload.id, subblockId, value)
                  }
                })
              }
              break
            case 'update-position': {
              const blockId = payload.id

              // Handle cases where the timestamp is missing (older operations or unexpected data).
              if (!data.timestamp) {
                logger.warn('Position update missing timestamp, applying without ordering check', {
                  blockId,
                })
                workflowStore.updateBlockPosition(payload.id, payload.position)
                break
              }

              // Get the timestamp from the incoming update and the last recorded timestamp for this block.
              const updateTimestamp = data.timestamp
              const lastTimestamp = lastPositionTimestamps.current.get(blockId) || 0

              // Only apply the update if its timestamp is newer than or equal to the last applied one.
              // This prevents out-of-order updates that can cause "jagged" block movements.
              if (updateTimestamp >= lastTimestamp) {
                workflowStore.updateBlockPosition(payload.id, payload.position)
                lastPositionTimestamps.current.set(blockId, updateTimestamp)
              } else {
                // Log and skip older updates.
                logger.debug('Skipping out-of-order position update', {
                  blockId,
                  updateTimestamp,
                  lastTimestamp,
                  position: payload.position,
                })
              }
              break
            }
            case 'update-name':
              workflowStore.updateBlockName(payload.id, payload.name)
              break
            case 'remove': {
              const blockId = payload.id
              // Create a set to keep track of all blocks to remove, starting with the primary block.
              const blocksToRemove = new Set<string>([blockId])

              // Recursive helper function to find all descendant blocks (nested blocks within a subflow).
              const findAllDescendants = (parentId: string) => {
                Object.entries(workflowStore.blocks).forEach(([id, block]) => {
                  // If a block's parent ID matches the current parent ID, add it and its descendants.
                  if (block.data?.parentId === parentId) {
                    blocksToRemove.add(id)
                    findAllDescendants(id)
                  }
                })
              }
              // Find all descendants of the block being removed.
              findAllDescendants(blockId)

              // Remove the main block from the workflow store. The store's internal logic will
              // typically handle removing associated edges and nested blocks.
              workflowStore.removeBlock(blockId)
              // Clear any position timestamps for the removed block.
              lastPositionTimestamps.current.delete(blockId)

              // After removing blocks, get the updated graph state (blocks and edges).
              const updatedBlocks = useWorkflowStore.getState().blocks
              const updatedEdges = useWorkflowStore.getState().edges
              const graph = {
                blocksById: updatedBlocks,
                edgesById: Object.fromEntries(updatedEdges.map((e) => [e.id, e])),
              }

              // Access the global undo/redo store to prune invalid entries.
              // This ensures that any undo/redo actions referencing the deleted blocks/edges
              // are removed from all users' stacks, preventing errors during undo/redo.
              const undoRedoStore = useUndoRedoStore.getState()
              const stackKeys = Object.keys(undoRedoStore.stacks)
              stackKeys.forEach((key) => {
                const [workflowId, userId] = key.split(':')
                if (workflowId === activeWorkflowId) {
                  undoRedoStore.pruneInvalidEntries(workflowId, userId, graph)
                }
              })
              break
            }
            case 'toggle-enabled':
              workflowStore.toggleBlockEnabled(payload.id)
              break
            case 'update-parent':
              workflowStore.updateParentId(payload.id, payload.parentId, payload.extent)
              break
            case 'update-wide':
              workflowStore.setBlockWide(payload.id, payload.isWide)
              break
            case 'update-advanced-mode':
              workflowStore.setBlockAdvancedMode(payload.id, payload.advancedMode)
              break
            case 'update-trigger-mode':
              workflowStore.setBlockTriggerMode(payload.id, payload.triggerMode)
              break
            case 'toggle-handles': {
              // Check if the block exists and if the handles state has actually changed to avoid unnecessary updates.
              const currentBlock = workflowStore.blocks[payload.id]
              if (currentBlock && currentBlock.horizontalHandles !== payload.horizontalHandles) {
                workflowStore.toggleBlockHandles(payload.id)
              }
              break
            }
            case 'duplicate':
              // Add the duplicated block using the provided payload.
              workflowStore.addBlock(
                payload.id,
                payload.type,
                payload.name,
                payload.position,
                payload.data,
                payload.parentId,
                payload.extent,
                {
                  enabled: payload.enabled,
                  horizontalHandles: payload.horizontalHandles,
                  isWide: payload.isWide,
                  advancedMode: payload.advancedMode,
                  triggerMode: payload.triggerMode ?? false,
                  height: payload.height,
                }
              )
              // Handle auto-connect edge for duplicates.
              if (payload.autoConnectEdge) {
                workflowStore.addEdge(payload.autoConnectEdge)
              }
              // Apply sub-block values from the duplicate payload immediately.
              if (payload.subBlocks && typeof payload.subBlocks === 'object') {
                Object.entries(payload.subBlocks).forEach(([subblockId, subblock]) => {
                  const value = (subblock as any)?.value
                  if (value !== undefined) {
                    subBlockStore.setValue(payload.id, subblockId, value)
                  }
                })
              }
              break
          }
        } else if (target === 'edge') {
          // Process different 'operation' types for edges.
          switch (operation) {
            case 'add':
              workflowStore.addEdge(payload as Edge) // Cast payload to Edge type.
              break
            case 'remove': {
              workflowStore.removeEdge(payload.id)

              // Similar to block removal, get updated graph state and prune undo/redo entries.
              const updatedBlocks = useWorkflowStore.getState().blocks
              const updatedEdges = useWorkflowStore.getState().edges
              const graph = {
                blocksById: updatedBlocks,
                edgesById: Object.fromEntries(updatedEdges.map((e) => [e.id, e])),
              }

              const undoRedoStore = useUndoRedoStore.getState()
              const stackKeys = Object.keys(undoRedoStore.stacks)
              stackKeys.forEach((key) => {
                const [workflowId, userId] = key.split(':')
                if (workflowId === activeWorkflowId) {
                  undoRedoStore.pruneInvalidEntries(workflowId, userId, graph)
                }
              })
              break
            }
          }
        } else if (target === 'subflow') {
          // Process different 'operation' types for subflows (e.g., loop or parallel blocks).
          switch (operation) {
            case 'update':
              // Handle configuration updates specific to loop or parallel subflows.
              if (payload.type === 'loop') {
                const { config } = payload
                if (config.loopType !== undefined) {
                  workflowStore.updateLoopType(payload.id, config.loopType)
                }
                if (config.iterations !== undefined) {
                  workflowStore.updateLoopCount(payload.id, config.iterations)
                }
                if (config.forEachItems !== undefined) {
                  workflowStore.updateLoopCollection(payload.id, config.forEachItems)
                }
              } else if (payload.type === 'parallel') {
                const { config } = payload
                if (config.parallelType !== undefined) {
                  workflowStore.updateParallelType(payload.id, config.parallelType)
                }
                if (config.count !== undefined) {
                  workflowStore.updateParallelCount(payload.id, config.count)
                }
                if (config.distribution !== undefined) {
                  workflowStore.updateParallelCollection(payload.id, config.distribution)
                }
              }
              break
          }
        } else if (target === 'variable') {
          // Process different 'operation' types for variables.
          switch (operation) {
            case 'add':
              variablesStore.addVariable(
                {
                  workflowId: payload.workflowId,
                  name: payload.name,
                  type: payload.type,
                  value: payload.value,
                },
                payload.id
              )
              break
            case 'remove':
              variablesStore.deleteVariable(payload.variableId)
              break
            case 'duplicate':
              variablesStore.duplicateVariable(payload.sourceVariableId, payload.id)
              break
          }
        }
      } catch (error) {
        logger.error('Error applying remote operation:', error)
      } finally {
        // Reset the flag after attempting to apply the remote change, regardless of success or failure.
        isApplyingRemoteChange.current = false
      }
    }

    // Handler for incoming sub-block updates.
    const handleSubblockUpdate = (data: any) => {
      const { blockId, subblockId, value, userId } = data

      if (isApplyingRemoteChange.current) return

      logger.info(`Received subblock update from user ${userId}: ${blockId}.${subblockId}`)

      isApplyingRemoteChange.current = true // Set flag before applying.

      try {
        // Apply the sub-block value to the sub-block store.
        // The setValue function automatically handles the active workflow ID.
        subBlockStore.setValue(blockId, subblockId, value)
      } catch (error) {
        logger.error('Error applying remote subblock update:', error)
      } finally {
        isApplyingRemoteChange.current = false // Reset flag.
      }
    }

    // Handler for incoming variable updates.
    const handleVariableUpdate = (data: any) => {
      const { variableId, field, value, userId } = data

      if (isApplyingRemoteChange.current) return

      logger.info(`Received variable update from user ${userId}: ${variableId}.${field}`)

      isApplyingRemoteChange.current = true // Set flag before applying.

      try {
        // Update the variable in the variables store based on the field being updated.
        if (field === 'name') {
          variablesStore.updateVariable(variableId, { name: value })
        } else if (field === 'value') {
          variablesStore.updateVariable(variableId, { value })
        } else if (field === 'type') {
          variablesStore.updateVariable(variableId, { type: value })
        }
      } catch (error) {
        logger.error('Error applying remote variable update:', error)
      } finally {
        isApplyingRemoteChange.current = false // Reset flag.
      }
    }

    // Handler for when another user joins the workflow room.
    const handleUserJoined = (data: any) => {
      logger.info(`User joined: ${data.userName}`)
    }

    // Handler for when another user leaves the workflow room.
    const handleUserLeft = (data: any) => {
      logger.info(`User left: ${data.userId}`)
    }

    // Handler for when the workflow is deleted remotely.
    const handleWorkflowDeleted = (data: any) => {
      const { workflowId } = data
      logger.warn(`Workflow ${workflowId} has been deleted`)

      // If the deleted workflow is the one currently active in the UI.
      if (activeWorkflowId === workflowId) {
        logger.info(
          `Currently active workflow ${workflowId} was deleted, stopping collaborative operations`
        )

        // Clear the undo/redo stacks for this workflow and current user.
        const currentUserId = session?.user?.id || 'unknown'
        useUndoRedoStore.getState().clear(workflowId, currentUserId)

        // Reset the remote change flag to ensure local operations are not blocked.
        isApplyingRemoteChange.current = false
      }
    }

    // Handler for when the workflow is reverted to a previously deployed state.
    const handleWorkflowReverted = async (data: any) => {
      const { workflowId } = data
      logger.info(`Workflow ${workflowId} has been reverted to deployed state`)

      // If the reverted workflow is the one currently active in the UI.
      if (activeWorkflowId === workflowId) {
        logger.info(`Currently active workflow ${workflowId} was reverted, reloading state`)

        try {
          // Fetch the entire updated workflow state from the server's API.
          const response = await fetch(`/api/workflows/${workflowId}`)
          if (response.ok) {
            const responseData = await response.json()
            const workflowData = responseData.data

            // If workflow data and its state are present.
            if (workflowData?.state) {
              isApplyingRemoteChange.current = true // Set flag before applying fetched state.
              try {
                // Directly update the main workflow store with the fetched state.
                // This replaces the entire local workflow state with the reverted version.
                useWorkflowStore.setState({
                  blocks: workflowData.state.blocks || {},
                  edges: workflowData.state.edges || [],
                  loops: workflowData.state.loops || {},
                  parallels: workflowData.state.parallels || {},
                  isDeployed: workflowData.state.isDeployed || false,
                  deployedAt: workflowData.state.deployedAt,
                  lastSaved: workflowData.state.lastSaved || Date.now(),
                  deploymentStatuses: workflowData.state.deploymentStatuses || {},
                })

                // Reconstruct and update the sub-block store with reverted values from the fetched block state.
                const subblockValues: Record<string, Record<string, any>> = {}
                Object.entries(workflowData.state.blocks || {}).forEach(([blockId, block]) => {
                  const blockState = block as any // Cast for easier property access.
                  subblockValues[blockId] = {}
                  Object.entries(blockState.subBlocks || {}).forEach(([subblockId, subblock]) => {
                    subblockValues[blockId][subblockId] = (subblock as any).value
                  })
                })

                // Update the sub-block store for the current workflow ID.
                useSubBlockStore.setState((state: any) => ({
                  workflowValues: {
                    ...state.workflowValues,
                    [workflowId]: subblockValues,
                  },
                }))

                logger.info(`Successfully loaded reverted workflow state for ${workflowId}`)

                // After reloading, prune undo/redo stacks again to ensure they only contain
                // valid actions for the new (reverted) workflow state.
                const graph = {
                  blocksById: workflowData.state.blocks || {},
                  edgesById: Object.fromEntries(
                    (workflowData.state.edges || []).map((e: any) => [e.id, e])
                  ),
                }

                const undoRedoStore = useUndoRedoStore.getState()
                const stackKeys = Object.keys(undoRedoStore.stacks)
                stackKeys.forEach((key) => {
                  const [wfId, userId] = key.split(':')
                  if (wfId === workflowId) {
                    undoRedoStore.pruneInvalidEntries(wfId, userId, graph)
                  }
                })
              } finally {
                isApplyingRemoteChange.current = false // Reset flag.
              }
            } else {
              logger.error('No state found in workflow data after revert', { workflowData })
            }
          } else {
            logger.error(`Failed to fetch workflow data after revert: ${response.statusText}`)
          }
        } catch (error) {
          logger.error('Error reloading workflow state after revert:', error)
        }
      }
    }

    // Handler for when a local operation is confirmed by the server.
    const handleOperationConfirmed = (data: any) => {
      const { operationId } = data
      logger.debug('Operation confirmed', { operationId })
      confirmOperation(operationId) // Tell the operation queue to remove this operation.
    }

    // Handler for when a local operation fails on the server.
    const handleOperationFailed = (data: any) => {
      const { operationId, error, retryable } = data
      logger.warn('Operation failed', { operationId, error, retryable })

      failOperation(operationId, retryable) // Tell the operation queue to mark as failed (and potentially retry).
    }

    // Register all the event handlers with the useSocket hook.
    onWorkflowOperation(handleWorkflowOperation)
    onSubblockUpdate(handleSubblockUpdate)
    onVariableUpdate(handleVariableUpdate)
    onUserJoined(handleUserJoined)
    onUserLeft(handleUserLeft)
    onWorkflowDeleted(handleWorkflowDeleted)
    onWorkflowReverted(handleWorkflowReverted)
    onOperationConfirmed(handleOperationConfirmed)
    onOperationFailed(handleOperationFailed)

    // The cleanup function for this useEffect is handled by the socket context itself,
    // which manages unregistering these listeners upon component unmount.
    return () => {
      // Cleanup handled by socket context
    }
  }, [
    onWorkflowOperation, // Dependencies: These are stable functions from useSocket.
    onSubblockUpdate,
    onVariableUpdate,
    onUserJoined,
    onUserLeft,
    onWorkflowDeleted,
    onWorkflowReverted,
    onOperationConfirmed,
    onOperationFailed,
    workflowStore, // Store instances are dependencies because their internal state changes, though the instance itself is stable.
    subBlockStore,
    variablesStore,
    activeWorkflowId, // activeWorkflowId is used in deletion/reversion logic.
    confirmOperation, // Functions from useOperationQueue.
    failOperation,
    emitWorkflowOperation, // Used in the explanation, but not directly called here.
    queue, // The queue state is a dependency, but not directly used in handlers, mostly for debug context.
    session?.user?.id, // session for current user ID in `handleWorkflowDeleted`
  ])

  // A generic callback to execute an operation collaboratively through the operation queue.
  // This function adds the operation to a local queue and performs the local state update immediately (optimistic UI).
  const executeQueuedOperation = useCallback(
    (operation: string, target: string, payload: any, localAction: () => void) => {
      // If a remote change is being applied, skip local operations to prevent conflicts.
      if (isApplyingRemoteChange.current) {
        return
      }

      // If the workflow is in "diff mode," skip collaborative operations.
      if (isShowingDiff) {
        logger.debug('Skipping socket operation in diff mode:', operation)
        return
      }

      // If not in an active collaborative room, log and skip the operation.
      if (!isInActiveRoom()) {
        logger.debug('Skipping operation - not in active workflow', {
          currentWorkflowId,
          activeWorkflowId,
          operation,
          target,
        })
        return
      }

      // Generate a unique ID for this operation.
      const operationId = crypto.randomUUID()

      // Add the operation to the local operation queue.
      addToQueue({
        id: operationId,
        operation: {
          operation,
          target,
          payload,
        },
        workflowId: activeWorkflowId || '',
        userId: session?.user?.id || 'unknown',
      })

      // Execute the local action immediately for optimistic UI feedback.
      localAction()
    },
    [
      addToQueue,
      session?.user?.id,
      isShowingDiff,
      activeWorkflowId,
      isInActiveRoom,
      currentWorkflowId,
    ]
  )

  // A callback for debounced operations that are directly emitted via WebSocket
  // instead of going through the full operation queue's retry/confirmation mechanism.
  // This is typically used for high-frequency updates where eventual consistency is fine,
  // and immediate emission is preferred over strict queue management.
  const executeQueuedDebouncedOperation = useCallback(
    (operation: string, target: string, payload: any, localAction: () => void) => {
      if (isApplyingRemoteChange.current) return // Skip if applying remote changes.

      if (isShowingDiff) {
        logger.debug('Skipping debounced socket operation in diff mode:', operation)
        return
      }

      if (!isInActiveRoom()) {
        logger.debug('Skipping debounced operation - not in active workflow', {
          currentWorkflowId,
          activeWorkflowId,
          operation,
          target,
        })
        return
      }

      // Execute the local action immediately.
      localAction()

      // Directly emit the operation via WebSocket, bypassing the local operation queue.
      // This means no local retry mechanism is explicitly managed for this type of operation.
      emitWorkflowOperation(operation, target, payload)
    },
    [emitWorkflowOperation, isShowingDiff, isInActiveRoom, currentWorkflowId, activeWorkflowId]
  )

  // Callback to add a new block collaboratively.
  const collaborativeAddBlock = useCallback(
    (
      id: string,
      type: string,
      name: string,
      position: Position,
      data?: Record<string, any>,
      parentId?: string,
      extent?: 'parent',
      autoConnectEdge?: Edge,
      triggerMode?: boolean
    ) => {
      if (isShowingDiff) {
        logger.debug('Skipping collaborative add block in diff mode')
        return
      }

      if (!isInActiveRoom()) {
        logger.debug('Skipping collaborative add block - not in active workflow', {
          currentWorkflowId,
          activeWorkflowId,
        })
        return
      }

      // Retrieve the block configuration based on its type.
      const blockConfig = getBlock(type)

      // Special handling for 'loop' or 'parallel' block types, which might not have a full BlockConfig.
      if (!blockConfig && (type === 'loop' || type === 'parallel')) {
        // For these block types, initialize with empty subBlocks and outputs.
        const completeBlockData = {
          id,
          type,
          name,
          position,
          data: data || {},
          subBlocks: {}, // No pre-defined sub-blocks for these.
          outputs: {}, // No pre-defined outputs.
          enabled: true,
          horizontalHandles: true,
          isWide: false,
          advancedMode: false,
          triggerMode: triggerMode || false,
          height: 0,
          parentId,
          extent,
          autoConnectEdge, // Include edge data for a single atomic operation.
        }

        // If applying remote changes, just add the block locally and return.
        // This pathway is triggered when `handleWorkflowOperation` receives an 'add' operation.
        if (isApplyingRemoteChange.current) {
          workflowStore.addBlock(id, type, name, position, data, parentId, extent, {
            triggerMode: triggerMode || false,
          })
          if (autoConnectEdge) {
            workflowStore.addEdge(autoConnectEdge)
          }
          return
        }

        // Generate an operation ID for the queue.
        const operationId = crypto.randomUUID()

        // Add the block creation operation to the queue.
        addToQueue({
          id: operationId,
          operation: {
            operation: 'add',
            target: 'block',
            payload: completeBlockData,
          },
          workflowId: activeWorkflowId || '',
          userId: session?.user?.id || 'unknown',
        })

        // Apply the block locally immediately for optimistic UI.
        workflowStore.addBlock(id, type, name, position, data, parentId, extent, {
          triggerMode: triggerMode || false,
        })
        if (autoConnectEdge) {
          workflowStore.addEdge(autoConnectEdge)
        }

        // Record the block addition for undo/redo *after* it's added locally.
        undoRedo.recordAddBlock(id, autoConnectEdge)

        return
      }

      // If the block configuration isn't found, log an error.
      if (!blockConfig) {
        logger.error(`Block type ${type} not found`)
        return
      }

      // Initialize sub-blocks for the new block based on its configuration.
      const subBlocks: Record<string, any> = {}
      if (blockConfig.subBlocks) {
        blockConfig.subBlocks.forEach((subBlock) => {
          subBlocks[subBlock.id] = {
            id: subBlock.id,
            type: subBlock.type,
            value: subBlock.defaultValue ?? null,
          }
        })
      }

      // Determine outputs based on block type and whether it's in trigger mode.
      const isTriggerMode = triggerMode || false
      const outputs = isTriggerMode
        ? getBlockOutputs(type, subBlocks, isTriggerMode)
        : resolveOutputType(blockConfig.outputs)

      // Construct the complete data payload for the new block.
      const completeBlockData = {
        id,
        type,
        name,
        position,
        data: data || {},
        subBlocks,
        outputs,
        enabled: true,
        horizontalHandles: true,
        isWide: false,
        advancedMode: false,
        triggerMode: isTriggerMode,
        height: 0, // Default height, will be set by the UI.
        parentId,
        extent,
        autoConnectEdge,
      }

      // If applying remote changes, return early (handled by the specific `handleWorkflowOperation` case).
      if (isApplyingRemoteChange.current) return

      // Generate operation ID.
      const operationId = crypto.randomUUID()

      // Add operation to the queue.
      addToQueue({
        id: operationId,
        operation: {
          operation: 'add',
          target: 'block',
          payload: completeBlockData,
        },
        workflowId: activeWorkflowId || '',
        userId: session?.user?.id || 'unknown',
      })

      // Apply locally for optimistic UI.
      workflowStore.addBlock(id, type, name, position, data, parentId, extent, {
        triggerMode: triggerMode || false,
      })
      if (autoConnectEdge) {
        workflowStore.addEdge(autoConnectEdge)
      }

      // Record for undo/redo.
      undoRedo.recordAddBlock(id, autoConnectEdge)
    },
    [
      workflowStore,
      activeWorkflowId,
      addToQueue,
      session?.user?.id,
      isShowingDiff,
      isInActiveRoom,
      currentWorkflowId,
      undoRedo,
    ]
  )

  // Callback to remove a block collaboratively.
  const collaborativeRemoveBlock = useCallback(
    (id: string) => {
      // Cancel any pending operations in the queue related to this block to avoid conflicts.
      cancelOperationsForBlock(id)

      // Find all blocks that will be removed, including descendants within subflows.
      const blocksToRemove = new Set<string>([id])
      const findAllDescendants = (parentId: string) => {
        Object.entries(workflowStore.blocks).forEach(([blockId, block]) => {
          if (block.data?.parentId === parentId) {
            blocksToRemove.add(blockId)
            findAllDescendants(blockId)
          }
        })
      }
      findAllDescendants(id)

      // Capture the state of all blocks being removed (including their sub-block values)
      // *before* removal, so it can be used for undo/redo.
      const allBlocks = mergeSubblockState(workflowStore.blocks, activeWorkflowId || undefined)
      const capturedBlocks: Record<string, BlockState> = {}
      blocksToRemove.forEach((blockId) => {
        if (allBlocks[blockId]) {
          capturedBlocks[blockId] = allBlocks[blockId]
        }
      })

      // Capture all edges connected to any of the blocks being removed for undo/redo.
      const edges = workflowStore.edges.filter(
        (edge) => blocksToRemove.has(edge.source) || blocksToRemove.has(edge.target)
      )

      // Record the removal for undo/redo, but only if there are blocks actually being removed.
      if (Object.keys(capturedBlocks).length > 0) {
        undoRedo.recordRemoveBlock(id, capturedBlocks[id], edges, capturedBlocks)
      }

      // Execute the remove operation through the queue.
      // The `localAction` simply calls `workflowStore.removeBlock(id)`.
      executeQueuedOperation('remove', 'block', { id }, () => workflowStore.removeBlock(id))
    },
    [executeQueuedOperation, workflowStore, cancelOperationsForBlock, undoRedo, activeWorkflowId]
  )

  // Callback to update a block's position collaboratively.
  const collaborativeUpdateBlockPosition = useCallback(
    (id: string, position: Position, commit = true) => {
      // If `commit` is true, use the full queued operation (e.g., for a final position after dragging).
      if (commit) {
        executeQueuedOperation('update-position', 'block', { id, position, commit }, () => {
          workflowStore.updateBlockPosition(id, position)
        })
        return
      }

      // If `commit` is false, use the debounced operation (e.g., for live dragging updates).
      // This bypasses the full operation queue for faster, less critical updates.
      executeQueuedDebouncedOperation('update-position', 'block', { id, position }, () => {
        workflowStore.updateBlockPosition(id, position)
      })
    },
    [executeQueuedDebouncedOperation, executeQueuedOperation, workflowStore]
  )

  // Callback to update a block's name collaboratively.
  const collaborativeUpdateBlockName = useCallback(
    (id: string, name: string) => {
      executeQueuedOperation('update-name', 'block', { id, name }, () => {
        workflowStore.updateBlockName(id, name)
      })
    },
    [executeQueuedOperation, workflowStore]
  )

  // Callback to toggle a block's enabled state collaboratively.
  const collaborativeToggleBlockEnabled = useCallback(
    (id: string) => {
      executeQueuedOperation('toggle-enabled', 'block', { id }, () =>
        workflowStore.toggleBlockEnabled(id)
      )
    },
    [executeQueuedOperation, workflowStore]
  )

  // Callback to update a block's parent ID collaboratively.
  const collaborativeUpdateParentId = useCallback(
    (id: string, parentId: string, extent: 'parent') => {
      executeQueuedOperation('update-parent', 'block', { id, parentId, extent }, () =>
        workflowStore.updateParentId(id, parentId, extent)
      )
    },
    [executeQueuedOperation, workflowStore]
  )

  // Callback to toggle a block's 'isWide' property collaboratively.
  const collaborativeToggleBlockWide = useCallback(
    (id: string) => {
      const currentBlock = workflowStore.blocks[id]
      if (!currentBlock) return

      const newIsWide = !currentBlock.isWide // Calculate the new state.

      executeQueuedOperation('update-wide', 'block', { id, isWide: newIsWide }, () =>
        workflowStore.toggleBlockWide(id)
      )
    },
    [executeQueuedOperation, workflowStore]
  )

  // Callback to toggle a block's 'advancedMode' property collaboratively.
  const collaborativeToggleBlockAdvancedMode = useCallback(
    (id: string) => {
      const currentBlock = workflowStore.blocks[id]
      if (!currentBlock) return

      const newAdvancedMode = !currentBlock.advancedMode

      executeQueuedOperation(
        'update-advanced-mode',
        'block',
        { id, advancedMode: newAdvancedMode },
        () => workflowStore.toggleBlockAdvancedMode(id)
      )
    },
    [executeQueuedOperation, workflowStore]
  )

  // Callback to toggle a block's 'triggerMode' property collaboratively.
  const collaborativeToggleBlockTriggerMode = useCallback(
    (id: string) => {
      const currentBlock = workflowStore.blocks[id]
      if (!currentBlock) return

      const newTriggerMode = !currentBlock.triggerMode

      executeQueuedOperation(
        'update-trigger-mode',
        'block',
        { id, triggerMode: newTriggerMode },
        () => workflowStore.toggleBlockTriggerMode(id)
      )
    },
    [executeQueuedOperation, workflowStore]
  )

  // Callback to toggle a block's 'horizontalHandles' property collaboratively.
  const collaborativeToggleBlockHandles = useCallback(
    (id: string) => {
      const currentBlock = workflowStore.blocks[id]
      if (!currentBlock) return

      const newHorizontalHandles = !currentBlock.horizontalHandles

      executeQueuedOperation(
        'toggle-handles',
        'block',
        { id, horizontalHandles: newHorizontalHandles },
        () => workflowStore.toggleBlockHandles(id)
      )
    },
    [executeQueuedOperation, workflowStore]
  )

  // Callback to add an edge collaboratively.
  const collaborativeAddEdge = useCallback(
    (edge: Edge) => {
      executeQueuedOperation('add', 'edge', edge, () => workflowStore.addEdge(edge))
      // Only record the edge addition for undo/redo if it's not part of a larger parent update operation.
      if (!skipEdgeRecording.current) {
        undoRedo.recordAddEdge(edge.id)
      }
    },
    [executeQueuedOperation, workflowStore, undoRedo]
  )

  // Callback to remove an edge collaboratively.
  const collaborativeRemoveEdge = useCallback(
    (edgeId: string) => {
      const edge = workflowStore.edges.find((e) => e.id === edgeId)

      // If the edge doesn't exist (e.g., already removed by a cascade deletion), skip.
      if (!edge) {
        logger.debug('Edge already removed, skipping operation', { edgeId })
        return
      }

      // Check if the source and target blocks of the edge still exist.
      // This is important to prevent errors if blocks were removed out of order.
      const sourceExists = workflowStore.blocks[edge.source]
      const targetExists = workflowStore.blocks[edge.target]

      if (!sourceExists || !targetExists) {
        logger.debug('Edge source or target block no longer exists, skipping operation', {
          edgeId,
          sourceExists: !!sourceExists,
          targetExists: !!targetExists,
        })
        return
      }

      // Only record the edge removal for undo/redo if it's not part of a larger parent update.
      if (!skipEdgeRecording.current) {
        undoRedo.recordRemoveEdge(edgeId, edge)
      }

      executeQueuedOperation('remove', 'edge', { id: edgeId }, () =>
        workflowStore.removeEdge(edgeId)
      )
    },
    [executeQueuedOperation, workflowStore, undoRedo]
  )

  // Callback to set the value of a sub-block collaboratively.
  const collaborativeSetSubblockValue = useCallback(
    (blockId: string, subblockId: string, value: any, options?: { _visited?: Set<string> }) => {
      if (isApplyingRemoteChange.current) return

      if (isShowingDiff) {
        logger.debug('Skipping collaborative subblock update in diff mode')
        return
      }

      if (!isInActiveRoom()) {
        logger.debug('Skipping subblock update - not in active workflow', {
          currentWorkflowId,
          activeWorkflowId,
          blockId,
          subblockId,
        })
        return
      }

      const operationId = crypto.randomUUID()

      // Add to queue for reliable delivery.
      addToQueue({
        id: operationId,
        operation: {
          operation: 'subblock-update',
          target: 'subblock',
          payload: { blockId, subblockId, value },
        },
        workflowId: activeWorkflowId || '',
        userId: session?.user?.id || 'unknown',
      })

      // Apply locally first for immediate UI feedback.
      subBlockStore.setValue(blockId, subblockId, value)

      // Declarative clearing logic: automatically clear dependent sub-blocks.
      // This is a "best-effort" attempt to maintain data consistency.
      try {
        // Use `_visited` to prevent infinite recursion in case of circular dependencies.
        const visited = options?._visited || new Set<string>()
        if (visited.has(subblockId)) return
        visited.add(subblockId)

        const blockType = useWorkflowStore.getState().blocks?.[blockId]?.type
        const blockConfig = blockType ? getBlock(blockType) : null

        if (blockConfig?.subBlocks && Array.isArray(blockConfig.subBlocks)) {
          // Find sub-blocks that declare a dependency on the currently updated `subblockId`.
          const dependents = blockConfig.subBlocks.filter(
            (sb: any) => Array.isArray(sb.dependsOn) && sb.dependsOn.includes(subblockId)
          )
          for (const dep of dependents) {
            // Skip clearing if the dependent ID is the same as the current subblock ID.
            if (!dep?.id || dep.id === subblockId) continue
            // Recursively call `collaborativeSetSubblockValue` for dependents to clear them,
            // ensuring the change is also propagated collaboratively and further cascades.
            collaborativeSetSubblockValue(blockId, dep.id, '', { _visited: visited })
          }
        }
      } catch {
        // Log errors but do not block the main operation, as this is a best-effort feature.
      }
    },
    [
      subBlockStore,
      currentWorkflowId,
      activeWorkflowId,
      addToQueue,
      session?.user?.id,
      isShowingDiff,
      isInActiveRoom,
    ]
  )

  // Callback specifically for setting tag selections. This uses the operation queue
  // but with an `immediate: true` flag, indicating it should be processed quickly.
  const collaborativeSetTagSelection = useCallback(
    (blockId: string, subblockId: string, value: any) => {
      if (isApplyingRemoteChange.current) return

      if (!isInActiveRoom()) {
        logger.debug('Skipping tag selection - not in active workflow', {
          currentWorkflowId,
          activeWorkflowId,
          blockId,
          subblockId,
        })
        return
      }

      // Apply locally first.
      subBlockStore.setValue(blockId, subblockId, value)

      const operationId = crypto.randomUUID()

      // Add to queue with `immediate: true`.
      addToQueue({
        id: operationId,
        operation: {
          operation: 'subblock-update',
          target: 'subblock',
          payload: { blockId, subblockId, value },
        },
        workflowId: activeWorkflowId || '',
        userId: session?.user?.id || 'unknown',
        immediate: true, // This flag indicates the operation should be sent as soon as possible.
      })
    },
    [
      subBlockStore,
      addToQueue,
      currentWorkflowId,
      activeWorkflowId,
      session?.user?.id,
      isInActiveRoom,
    ]
  )

  // Callback to duplicate a block collaboratively.
  const collaborativeDuplicateBlock = useCallback(
    (sourceId: string) => {
      if (!isInActiveRoom()) {
        logger.debug('Skipping duplicate block - not in active workflow', {
          currentWorkflowId,
          activeWorkflowId,
          sourceId,
        })
        return
      }

      const sourceBlock = workflowStore.blocks[sourceId]
      if (!sourceBlock) return

      // Generate a new unique ID and an offset position for the duplicated block.
      const newId = crypto.randomUUID()
      const offsetPosition = {
        x: sourceBlock.position.x + 250,
        y: sourceBlock.position.y + 20,
      }

      // Generate a unique name for the duplicated block.
      const newName = getUniqueBlockName(sourceBlock.name, workflowStore.blocks)

      // Get current sub-block values for the source block from the store.
      const subBlockValues = subBlockStore.workflowValues[activeWorkflowId || '']?.[sourceId] || {}

      // Merge the sub-block structure with their actual values to create a complete subBlocks object.
      const mergedSubBlocks = sourceBlock.subBlocks
        ? JSON.parse(JSON.stringify(sourceBlock.subBlocks))
        : {} // Deep copy
      Object.entries(subBlockValues).forEach(([subblockId, value]) => {
        if (mergedSubBlocks[subblockId]) {
          mergedSubBlocks[subblockId].value = value
        } else {
          // If a subblock exists in values but not in definition, add it with unknown type.
          mergedSubBlocks[subblockId] = {
            id: subblockId,
            type: 'unknown',
            value: value,
          }
        }
      })

      // Prepare the complete block data for the duplicated block to be sent in the operation.
      const duplicatedBlockData = {
        sourceId, // Original block's ID for reference.
        id: newId,
        type: sourceBlock.type,
        name: newName,
        position: offsetPosition,
        data: sourceBlock.data ? JSON.parse(JSON.stringify(sourceBlock.data)) : {}, // Deep copy.
        subBlocks: mergedSubBlocks,
        outputs: sourceBlock.outputs ? JSON.parse(JSON.stringify(sourceBlock.outputs)) : {}, // Deep copy.
        parentId: sourceBlock.data?.parentId || null,
        extent: sourceBlock.data?.extent || null,
        enabled: sourceBlock.enabled ?? true,
        horizontalHandles: sourceBlock.horizontalHandles ?? true,
        isWide: sourceBlock.isWide ?? false,
        advancedMode: sourceBlock.advancedMode ?? false,
        triggerMode: false, // Duplicated blocks are always created in normal mode to prevent webhook conflicts.
        height: sourceBlock.height || 0,
      }

      // Add the block to the local workflow store immediately for optimistic UI.
      workflowStore.addBlock(
        newId,
        sourceBlock.type,
        newName,
        offsetPosition,
        sourceBlock.data ? JSON.parse(JSON.stringify(sourceBlock.data)) : {},
        sourceBlock.data?.parentId,
        sourceBlock.data?.extent,
        {
          enabled: sourceBlock.enabled,
          horizontalHandles: sourceBlock.horizontalHandles,
          isWide: sourceBlock.isWide,
          advancedMode: sourceBlock.advancedMode,
          triggerMode: false, // Ensure normal mode.
          height: sourceBlock.height,
        }
      )

      // Execute the 'duplicate' operation through the queue.
      // The `localAction` passed to `executeQueuedOperation` performs the actual local state updates again.
      executeQueuedOperation('duplicate', 'block', duplicatedBlockData, () => {
        // This localAction is redundant if the above `workflowStore.addBlock` already ran,
        // but it ensures the block is added if `executeQueuedOperation`'s internal
        // `localAction` is the primary path (e.g., if the initial `workflowStore.addBlock` was skipped).
        workflowStore.addBlock(
          newId,
          sourceBlock.type,
          newName,
          offsetPosition,
          sourceBlock.data ? JSON.parse(JSON.stringify(sourceBlock.data)) : {},
          sourceBlock.data?.parentId,
          sourceBlock.data?.extent,
          {
            enabled: sourceBlock.enabled,
            horizontalHandles: sourceBlock.horizontalHandles,
            isWide: sourceBlock.isWide,
            advancedMode: sourceBlock.advancedMode,
            triggerMode: false,
            height: sourceBlock.height,
          }
        )

        // Apply sub-block values locally for the duplicated block.
        if (activeWorkflowId && Object.keys(subBlockValues).length > 0) {
          Object.entries(subBlockValues).forEach(([subblockId, value]) => {
            subBlockStore.setValue(newId, subblockId, value)
          })
        }

        // Record the duplicate action for undo/redo.
        undoRedo.recordDuplicateBlock(sourceId, newId, duplicatedBlockData, undefined)
      })
    },
    [
      executeQueuedOperation,
      workflowStore,
      subBlockStore,
      activeWorkflowId,
      isInActiveRoom,
      currentWorkflowId,
      undoRedo,
    ]
  )

  // Callback to update the type of a loop subflow collaboratively.
  const collaborativeUpdateLoopType = useCallback(
    (loopId: string, loopType: 'for' | 'forEach') => {
      const currentBlock = workflowStore.blocks[loopId]
      if (!currentBlock || currentBlock.type !== 'loop') return

      // Get all child nodes within the loop to include in the payload.
      const childNodes = Object.values(workflowStore.blocks)
        .filter((b) => b.data?.parentId === loopId)
        .map((b) => b.id)

      const currentIterations = currentBlock.data?.count || 5
      const currentCollection = currentBlock.data?.collection || ''

      // Construct the configuration payload for the update.
      const config = {
        id: loopId,
        nodes: childNodes,
        iterations: currentIterations,
        loopType,
        forEachItems: currentCollection,
      }

      executeQueuedOperation('update', 'subflow', { id: loopId, type: 'loop', config }, () =>
        workflowStore.updateLoopType(loopId, loopType)
      )
    },
    [executeQueuedOperation, workflowStore]
  )

  // Callback to update the type of a parallel subflow collaboratively.
  const collaborativeUpdateParallelType = useCallback(
    (parallelId: string, parallelType: 'count' | 'collection') => {
      const currentBlock = workflowStore.blocks[parallelId]
      if (!currentBlock || currentBlock.type !== 'parallel') return

      // Get all child nodes within the parallel subflow.
      const childNodes = Object.values(workflowStore.blocks)
        .filter((b) => b.data?.parentId === parallelId)
        .map((b) => b.id)

      let newCount = currentBlock.data?.count || 5
      let newDistribution = currentBlock.data?.collection || ''

      // Adjust `newCount` or `newDistribution` based on the `parallelType` being set.
      if (parallelType === 'count') {
        newDistribution = '' // Clear distribution if switching to count-based.
      } else {
        newCount = 1 // Set count to 1 if switching to collection-based.
        newDistribution = newDistribution || ''
      }

      // Construct the configuration payload.
      const config = {
        id: parallelId,
        nodes: childNodes,
        count: newCount,
        distribution: newDistribution,
        parallelType,
      }

      executeQueuedOperation(
        'update',
        'subflow',
        { id: parallelId, type: 'parallel', config },
        () => {
          // Perform multiple local updates for the parallel block's properties.
          workflowStore.updateParallelType(parallelId, parallelType)
          workflowStore.updateParallelCount(parallelId, newCount)
          workflowStore.updateParallelCollection(parallelId, newDistribution)
        }
      )
    },
    [executeQueuedOperation, workflowStore]
  )

  // Unified callback to update iteration count for both loop and parallel subflows.
  const collaborativeUpdateIterationCount = useCallback(
    (nodeId: string, iterationType: 'loop' | 'parallel', count: number) => {
      const currentBlock = workflowStore.blocks[nodeId]
      if (!currentBlock || currentBlock.type !== iterationType) return

      const childNodes = Object.values(workflowStore.blocks)
        .filter((b) => b.data?.parentId === nodeId)
        .map((b) => b.id)

      if (iterationType === 'loop') {
        const currentLoopType = currentBlock.data?.loopType || 'for'
        const currentCollection = currentBlock.data?.collection || ''

        const config = {
          id: nodeId,
          nodes: childNodes,
          iterations: Math.max(1, Math.min(100, count)), // Clamp loop iterations between 1 and 100.
          loopType: currentLoopType,
          forEachItems: currentCollection,
        }

        executeQueuedOperation('update', 'subflow', { id: nodeId, type: 'loop', config }, () =>
          workflowStore.updateLoopCount(nodeId, count)
        )
      } else {
        // iterationType === 'parallel'
        const currentDistribution = currentBlock.data?.collection || ''
        const currentParallelType = currentBlock.data?.parallelType || 'count'

        const config = {
          id: nodeId,
          nodes: childNodes,
          count: Math.max(1, Math.min(20, count)), // Clamp parallel count between 1 and 20.
          distribution: currentDistribution,
          parallelType: currentParallelType,
        }

        executeQueuedOperation('update', 'subflow', { id: nodeId, type: 'parallel', config }, () =>
          workflowStore.updateParallelCount(nodeId, count)
        )
      }
    },
    [executeQueuedOperation, workflowStore]
  )

  // Unified callback to update iteration collection for both loop and parallel subflows.
  const collaborativeUpdateIterationCollection = useCallback(
    (nodeId: string, iterationType: 'loop' | 'parallel', collection: string) => {
      const currentBlock = workflowStore.blocks[nodeId]
      if (!currentBlock || currentBlock.type !== iterationType) return

      const childNodes = Object.values(workflowStore.blocks)
        .filter((b) => b.data?.parentId === nodeId)
        .map((b) => b.id)

      if (iterationType === 'loop') {
        const currentIterations = currentBlock.data?.count || 5
        const currentLoopType = currentBlock.data?.loopType || 'for'

        const config = {
          id: nodeId,
          nodes: childNodes,
          iterations: currentIterations,
          loopType: currentLoopType,
          forEachItems: collection,
        }

        executeQueuedOperation('update', 'subflow', { id: nodeId, type: 'loop', config }, () =>
          workflowStore.updateLoopCollection(nodeId, collection)
        )
      } else {
        // iterationType === 'parallel'
        const currentCount = currentBlock.data?.count || 5
        const currentParallelType = currentBlock.data?.parallelType || 'count'

        const config = {
          id: nodeId,
          nodes: childNodes,
          count: currentCount,
          distribution: collection,
          parallelType: currentParallelType,
        }

        executeQueuedOperation('update', 'subflow', { id: nodeId, type: 'parallel', config }, () =>
          workflowStore.updateParallelCollection(nodeId, collection)
        )
      }
    },
    [executeQueuedOperation, workflowStore]
  )

  // Callback to update a variable's name, value, or type collaboratively.
  const collaborativeUpdateVariable = useCallback(
    (variableId: string, field: 'name' | 'value' | 'type', value: any) => {
      executeQueuedOperation('variable-update', 'variable', { variableId, field, value }, () => {
        if (field === 'name') {
          variablesStore.updateVariable(variableId, { name: value })
        } else if (field === 'value') {
          variablesStore.updateVariable(variableId, { value })
        } else if (field === 'type') {
          variablesStore.updateVariable(variableId, { type: value })
        }
      })
    },
    [executeQueuedOperation, variablesStore]
  )

  // Callback to add a new variable collaboratively.
  const collaborativeAddVariable = useCallback(
    (variableData: { name: string; type: any; value: any; workflowId: string }) => {
      const id = crypto.randomUUID() // Generate a new ID for the variable.
      variablesStore.addVariable(variableData, id) // Add locally first.
      const processedVariable = useVariablesStore.getState().variables[id] // Get the variable, potentially with a processed name (e.g., unique).

      if (processedVariable) {
        const payloadWithProcessedName = {
          ...variableData,
          id,
          name: processedVariable.name, // Use the potentially unique/processed name.
        }

        // Add to queue. The local action is empty because it was already performed above.
        executeQueuedOperation('add', 'variable', payloadWithProcessedName, () => {})
      }

      return id // Return the ID of the new variable.
    },
    [executeQueuedOperation, variablesStore]
  )

  // Callback to delete a variable collaboratively.
  const collaborativeDeleteVariable = useCallback(
    (variableId: string) => {
      cancelOperationsForVariable(variableId) // Cancel any pending operations related to this variable.

      executeQueuedOperation('remove', 'variable', { variableId }, () => {
        variablesStore.deleteVariable(variableId) // Delete locally.
      })
    },
    [executeQueuedOperation, variablesStore, cancelOperationsForVariable]
  )

  // Callback to duplicate a variable collaboratively.
  const collaborativeDuplicateVariable = useCallback(
    (variableId: string) => {
      const newId = crypto.randomUUID()
      const sourceVariable = useVariablesStore.getState().variables[variableId]
      if (!sourceVariable) return null

      executeQueuedOperation(
        'duplicate',
        'variable',
        { sourceVariableId: variableId, id: newId },
        () => {
          variablesStore.duplicateVariable(variableId, newId) // Duplicate locally.
        }
      )
      return newId
    },
    [executeQueuedOperation, variablesStore]
  )

  // Return an object containing all collaborative functions, state, and direct store access.
  return {
    // Connection status and presence information.
    isConnected,
    currentWorkflowId,
    presenceUsers,
    hasOperationError,

    // Functions to manage the workflow room.
    joinWorkflow,
    leaveWorkflow,

    // Collaborative functions for blocks, edges, sub-blocks.
    collaborativeAddBlock,
    collaborativeUpdateBlockPosition,
    collaborativeUpdateBlockName,
    collaborativeRemoveBlock,
    collaborativeToggleBlockEnabled,
    collaborativeUpdateParentId,
    collaborativeToggleBlockWide,
    collaborativeToggleBlockAdvancedMode,
    collaborativeToggleBlockTriggerMode,
    collaborativeToggleBlockHandles,
    collaborativeDuplicateBlock,
    collaborativeAddEdge,
    collaborativeRemoveEdge,
    collaborativeSetSubblockValue,
    collaborativeSetTagSelection,

    // Collaborative functions for variables.
    collaborativeUpdateVariable,
    collaborativeAddVariable,
    collaborativeDeleteVariable,
    collaborativeDuplicateVariable,

    // Collaborative functions for loop/parallel subflows.
    collaborativeUpdateLoopType,
    collaborativeUpdateParallelType,
    collaborativeUpdateIterationCount,
    collaborativeUpdateIterationCollection,

    // Direct access to core stores (for non-collaborative operations or reading state).
    workflowStore,
    subBlockStore,

    // Wrapped undo/redo functions.
    // These are wrapped to set `isUndoRedoInProgress` flag, preventing undo/redo actions
    // from recording themselves, which would cause an infinite loop in the undo stack.
    undo: useCallback(() => {
      isUndoRedoInProgress.current = true // Set flag.
      undoRedo.undo() // Perform undo.
      // Use queueMicrotask to reset the flag *after* the current event loop turn,
      // ensuring all related effects/handlers complete before allowing new recordings.
      queueMicrotask(() => {
        isUndoRedoInProgress.current = false
      })
    }, [undoRedo]), // Dependency: undoRedo instance.
    redo: useCallback(() => {
      isUndoRedoInProgress.current = true // Set flag.
      undoRedo.redo() // Perform redo.
      queueMicrotask(() => {
        isUndoRedoInProgress.current = false
      })
    }, [undoRedo]), // Dependency: undoRedo instance.
    getUndoRedoSizes: undoRedo.getStackSizes, // Function to get sizes of undo/redo stacks.
    clearUndoRedo: undoRedo.clearStacks, // Function to clear undo/redo stacks.
  }
}
```