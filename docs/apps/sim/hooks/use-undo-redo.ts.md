This TypeScript file defines a React custom hook, `useUndoRedo`, which provides comprehensive undo and redo functionality for a workflow editor. It tracks various user actions on a workflow, such as adding, removing, moving, or duplicating blocks (nodes) and adding or removing edges (connections), and allows users to reverse or re-apply those actions.

The core idea is that for every user action (called an "operation"), the system also records an "inverse" operation that can reverse the original action. These pairs of operations are stored in an undo/redo stack, managed by the `useUndoRedoStore`.

---

## Detailed Explanation of `useUndoRedo` Hook

### 1. Purpose of this File

This `useUndoRedo.ts` file is a central component for managing the transactional history of changes within a workflow editor. It provides a set of functions to:

1.  **Record User Actions:** When a user performs an action (e.g., adds a block, moves an edge), this hook provides methods (`recordAddBlock`, `recordMove`, etc.) to store that action and its "inverse" (how to undo it) in a history stack.
2.  **Enable Undo:** Allows users to revert the last recorded action by executing its inverse.
3.  **Enable Redo:** Allows users to re-apply an action that was previously undone, by executing the original action again.
4.  **Synchronize with Backend:** All local state changes triggered by undo/redo are also dispatched to a global `OperationQueue` to be synchronized with a backend server, ensuring persistence and consistency.
5.  **Manage Complex State:** Handles intricacies like nested blocks (subflows), block sub-properties (`subBlocks`), and maintaining unique block names upon restoration.

In essence, it's the engine behind the "undo" and "redo" buttons in the workflow UI, providing a robust and persistent history of modifications.

### 2. Simplifying Complex Logic: Operations and Inverses

The fundamental principle behind this undo/redo system is the concept of **Operation-Inverse Pairs**:

*   **Operation:** This describes the actual action the user performed (e.g., "Add Block X").
*   **Inverse:** This describes the action required to completely reverse the "Operation" (e.g., "Remove Block X").

When an action is recorded, both the `operation` and its `inverse` are bundled together into an `OperationEntry` and pushed onto an "undo stack."

*   **`undo()`:** When `undo()` is called, it pops the latest `OperationEntry` from the undo stack, takes its `inverse` operation, and executes it. This `OperationEntry` is then pushed onto a "redo stack."
*   **`redo()`:** When `redo()` is called, it pops the latest `OperationEntry` from the redo stack, takes its original `operation`, and executes it. This `OperationEntry` is then pushed back onto the undo stack.

This mechanism ensures that any change can be reversed and then re-applied as needed, maintaining a coherent history. Crucially, operations involving adding elements (like blocks or edges) often have inverses that involve *removing* those elements, and vice-versa. For state-modifying operations like `move-block` or `update-parent`, the inverse simply swaps the `before` and `after` states.

**Key considerations for complexity:**

*   **Snapshots:** To accurately undo/redo additions or removals, the system often needs to capture a "snapshot" of the block or edge's state *before* it was removed. This ensures that when it's re-added (via undoing a removal or redoing an addition), all its properties are restored correctly. This is particularly important for blocks that contain nested blocks or have complex `data` structures.
*   **Order of Operations (Subflows):** When dealing with subflows (blocks that contain other blocks), the order of operations for adding/removing needs careful management:
    *   **Adding:** Add the parent block first, then its nested blocks, then any connected edges.
    *   **Removing:** Remove edges first, then nested blocks, then the parent block.
*   **Server Synchronization:** All local changes (adding, removing, moving) need to be mirrored on the backend. This is handled by pushing corresponding "remote operations" to the `useOperationQueue`, which then dispatches them to the server. This queue is distinct from the local undo/redo stack.

---

### 3. Line-by-Line Explanation

```typescript
import { useCallback } from 'react'
import type { Edge } from 'reactflow'
import { useSession } from '@/lib/auth-client'
import { createLogger } from '@/lib/logs/console/logger'
import { useOperationQueue } from '@/stores/operation-queue/store'
import {
  createOperationEntry,
  type DuplicateBlockOperation,
  type MoveBlockOperation,
  type Operation,
  type RemoveBlockOperation,
  type RemoveEdgeOperation,
  type UpdateParentOperation,
  useUndoRedoStore,
} from '@/stores/undo-redo'
import { useWorkflowRegistry } from '@/stores/workflows/registry/store'
import { useSubBlockStore } from '@/stores/workflows/subblock/store'
import { getUniqueBlockName, mergeSubblockState } from '@/stores/workflows/utils'
import { useWorkflowStore } from '@/stores/workflows/workflow/store'
import type { BlockState } from '@/stores/workflows/workflow/types'

// Imports: These lines bring in necessary functions, types, and hooks from React, external libraries, and internal modules.

// `useCallback`: A React hook to memoize functions, preventing unnecessary re-creation on re-renders, which is important for performance, especially with functions passed down as props or used in dependencies arrays.
// `Edge` from `reactflow`: A type definition representing a connection between two blocks in the workflow UI.
// `useSession` from `@/lib/auth-client`: A custom hook to access user session data, typically used to get the current user's ID.
// `createLogger` from `@/lib/logs/console/logger`: A utility to create a logger instance for structured logging.
// `useOperationQueue` from `@/stores/operation-queue/store`: A custom hook to interact with a Zustand store that manages a queue of operations to be sent to the backend. This ensures persistence of changes.
// From `@/stores/undo-redo`:
//    `createOperationEntry`: A utility function to bundle an operation and its inverse together.
//    `type DuplicateBlockOperation`, `MoveBlockOperation`, etc.: TypeScript type definitions for various specific operation types handled by the undo/redo system.
//    `type Operation`: A union type encompassing all possible operations.
//    `useUndoRedoStore`: A custom hook to interact with a Zustand store that manages the undo and redo stacks.
// `useWorkflowRegistry` from `@/stores/workflows/registry/store`: A custom hook to get information about the currently active workflow, such as its ID.
// `useSubBlockStore` from `@/stores/workflows/subblock/store`: A custom hook to interact with a Zustand store that manages the state/values of sub-blocks (properties within a block).
// `getUniqueBlockName`, `mergeSubblockState` from `@/stores/workflows/utils`: Utility functions:
//    `getUniqueBlockName`: Ensures that when blocks are added or duplicated, their names are unique within the workflow.
//    `mergeSubblockState`: Merges the state of a block with its associated sub-block values, providing a complete snapshot.
// `useWorkflowStore` from `@/stores/workflows/workflow/store`: A custom hook to interact with a Zustand store that holds the main state of the active workflow (e.g., all blocks and edges).
// `type BlockState` from `@/stores/workflows/workflow/types`: A TypeScript type definition for the detailed state of a single workflow block.

const logger = createLogger('UndoRedo')
// Initializes a logger specifically for the UndoRedo module, which will output debug, info, and error messages to the console.

export function useUndoRedo() {
  // Defines the custom React hook `useUndoRedo`.
  const { data: session } = useSession()
  // Retrieves the current user session data. `data` is destructured and aliased to `session`.
  const { activeWorkflowId } = useWorkflowRegistry()
  // Retrieves the ID of the currently active workflow from the workflow registry store.
  const workflowStore = useWorkflowStore()
  // Gets the instance of the workflow store, which contains the current blocks and edges of the workflow.
  const undoRedoStore = useUndoRedoStore()
  // Gets the instance of the undo/redo store, which manages the undo and redo stacks.
  const { addToQueue } = useOperationQueue()
  // Destructures the `addToQueue` function from the operation queue store. This function is used to send operations to the backend for persistence.

  const userId = session?.user?.id || 'unknown'
  // Extracts the user ID from the session data. If the session or user ID is not available, it defaults to 'unknown'.

  // --- RECORDING OPERATIONS ---

  const recordAddBlock = useCallback(
    (blockId: string, autoConnectEdge?: Edge) => {
      // Memoized function to record when a new block is added.
      // `blockId`: The ID of the newly added block.
      // `autoConnectEdge`: An optional edge that might have been automatically created when the block was added (e.g., connecting it to a previous block).

      if (!activeWorkflowId) return
      // If there's no active workflow, exit early as there's no context to record the operation.

      const operation: Operation = {
        id: crypto.randomUUID(), // Generates a unique ID for this operation.
        type: 'add-block', // Specifies the type of operation.
        timestamp: Date.now(), // Records the time the operation occurred.
        workflowId: activeWorkflowId, // Associates the operation with the current workflow.
        userId, // Associates the operation with the current user.
        data: { blockId }, // The specific data for this operation: the ID of the added block.
      }

      // Get fresh state from store
      const currentBlocks = useWorkflowStore.getState().blocks
      // Directly accesses the current blocks from the workflow store's state to ensure the latest data is used.
      const merged = mergeSubblockState(currentBlocks, activeWorkflowId, blockId)
      // Merges the sub-block state into the `currentBlocks` for the specific `blockId`, providing a complete snapshot.
      const blockSnapshot = merged[blockId] || currentBlocks[blockId]
      // Captures a snapshot of the newly added block's state. This is crucial for correctly undoing/redoing.

      const edgesToRemove = autoConnectEdge ? [autoConnectEdge] : []
      // If an `autoConnectEdge` exists, it's included in `edgesToRemove` to be part of the snapshot.

      const inverse: RemoveBlockOperation = {
        // Defines the inverse operation: removing the block.
        id: crypto.randomUUID(), // Unique ID for the inverse operation.
        type: 'remove-block', // Type of the inverse operation.
        timestamp: Date.now(),
        workflowId: activeWorkflowId,
        userId,
        data: {
          blockId, // The ID of the block to remove.
          blockSnapshot, // The snapshot of the block *before* it was added (captured during the original add), needed to re-create it on undo.
          edgeSnapshots: edgesToRemove, // Snapshots of edges that should be removed if this inverse operation is executed.
        },
      }

      const entry = createOperationEntry(operation, inverse)
      // Bundles the original operation and its inverse into a single entry.
      undoRedoStore.push(activeWorkflowId, userId, entry)
      // Pushes this entry onto the undo stack for the current workflow and user.

      logger.debug('Recorded add block', {
        blockId,
        hasAutoConnect: !!autoConnectEdge,
        edgeCount: edgesToRemove.length,
        workflowId: activeWorkflowId,
        hasSnapshot: !!blockSnapshot,
      })
      // Logs the successful recording of the add block operation.
    },
    [activeWorkflowId, userId, undoRedoStore]
  ) // Dependencies for `useCallback`: Ensures the function is re-created only if these values change.

  const recordRemoveBlock = useCallback(
    (
      blockId: string,
      blockSnapshot: BlockState,
      edgeSnapshots: Edge[],
      allBlockSnapshots?: Record<string, BlockState>
    ) => {
      // Memoized function to record when a block is removed.
      // `blockId`: ID of the block removed.
      // `blockSnapshot`: A snapshot of the block's state *before* removal, essential for re-adding it on undo.
      // `edgeSnapshots`: Snapshots of all edges connected to the block that were also removed.
      // `allBlockSnapshots`: Optional; if the removed block was a subflow, this contains snapshots of all nested blocks.

      if (!activeWorkflowId) return

      const operation: RemoveBlockOperation = {
        // Defines the original operation: removing the block.
        id: crypto.randomUUID(),
        type: 'remove-block',
        timestamp: Date.now(),
        workflowId: activeWorkflowId,
        userId,
        data: {
          blockId,
          blockSnapshot,
          edgeSnapshots,
          allBlockSnapshots,
        },
      }

      const inverse: Operation = {
        // Defines the inverse operation: adding the block back.
        id: crypto.randomUUID(),
        type: 'add-block',
        timestamp: Date.now(),
        workflowId: activeWorkflowId,
        userId,
        data: { blockId }, // The inverse only needs the blockId; the undo logic will use the `blockSnapshot` from the *original* operation.
      }

      const entry = createOperationEntry(operation, inverse)
      undoRedoStore.push(activeWorkflowId, userId, entry)

      logger.debug('Recorded remove block', { blockId, workflowId: activeWorkflowId })
    },
    [activeWorkflowId, userId, undoRedoStore]
  )

  const recordAddEdge = useCallback(
    (edgeId: string) => {
      // Memoized function to record when an edge is added.
      // `edgeId`: The ID of the newly added edge.

      if (!activeWorkflowId) return

      const operation: Operation = {
        id: crypto.randomUUID(),
        type: 'add-edge',
        timestamp: Date.now(),
        workflowId: activeWorkflowId,
        userId,
        data: { edgeId },
      }

      const inverse: RemoveEdgeOperation = {
        // Defines the inverse operation: removing the edge.
        id: crypto.randomUUID(),
        type: 'remove-edge',
        timestamp: Date.now(),
        workflowId: activeWorkflowId,
        userId,
        data: {
          edgeId,
          edgeSnapshot: workflowStore.edges.find((e) => e.id === edgeId) || null,
          // Captures a snapshot of the newly added edge *right after it's added* from the workflow store. This ensures that if the 'add-edge' is undone, we have all details to remove it.
        },
      }

      const entry = createOperationEntry(operation, inverse)
      undoRedoStore.push(activeWorkflowId, userId, entry)

      logger.debug('Recorded add edge', { edgeId, workflowId: activeWorkflowId })
    },
    [activeWorkflowId, userId, workflowStore, undoRedoStore]
  ) // Dependencies include `workflowStore` to get the edge snapshot.

  const recordRemoveEdge = useCallback(
    (edgeId: string, edgeSnapshot: Edge) => {
      // Memoized function to record when an edge is removed.
      // `edgeId`: ID of the edge removed.
      // `edgeSnapshot`: A snapshot of the edge's state *before* removal, essential for re-adding it on undo.

      if (!activeWorkflowId) return

      const operation: RemoveEdgeOperation = {
        // Defines the original operation: removing the edge.
        id: crypto.randomUUID(),
        type: 'remove-edge',
        timestamp: Date.now(),
        workflowId: activeWorkflowId,
        userId,
        data: {
          edgeId,
          edgeSnapshot,
        },
      }

      const inverse: Operation = {
        // Defines the inverse operation: adding the edge back.
        id: crypto.randomUUID(),
        type: 'add-edge',
        timestamp: Date.now(),
        workflowId: activeWorkflowId,
        userId,
        data: { edgeId }, // Similar to add-block, the inverse just needs the ID; the undo logic will use the `edgeSnapshot` from the *original* operation.
      }

      const entry = createOperationEntry(operation, inverse)
      undoRedoStore.push(activeWorkflowId, userId, entry)

      logger.debug('Recorded remove edge', { edgeId, workflowId: activeWorkflowId })
    },
    [activeWorkflowId, userId, undoRedoStore]
  )

  const recordMove = useCallback(
    (
      blockId: string,
      before: { x: number; y: number; parentId?: string },
      after: { x: number; y: number; parentId?: string }
    ) => {
      // Memoized function to record when a block is moved.
      // `blockId`: ID of the block moved.
      // `before`: The block's position and parent ID *before* the move.
      // `after`: The block's position and parent ID *after* the move.

      if (!activeWorkflowId) return

      const operation: MoveBlockOperation = {
        // Defines the original operation: moving the block.
        id: crypto.randomUUID(),
        type: 'move-block',
        timestamp: Date.now(),
        workflowId: activeWorkflowId,
        userId,
        data: {
          blockId,
          before,
          after,
        },
      }

      const inverse: MoveBlockOperation = {
        // Defines the inverse operation: moving the block back to its original position.
        // It's also a 'move-block' operation, but `before` and `after` are swapped.
        id: crypto.randomUUID(),
        type: 'move-block',
        timestamp: Date.now(),
        workflowId: activeWorkflowId,
        userId,
        data: {
          blockId,
          before: after, // The 'after' of the original move becomes the 'before' of the inverse.
          after: before, // The 'before' of the original move becomes the 'after' of the inverse.
        },
      }

      const entry = createOperationEntry(operation, inverse)
      undoRedoStore.push(activeWorkflowId, userId, entry)

      logger.debug('Recorded move', { blockId, from: before, to: after })
    },
    [activeWorkflowId, userId, undoRedoStore]
  )

  const recordDuplicateBlock = useCallback(
    (
      sourceBlockId: string,
      duplicatedBlockId: string,
      duplicatedBlockSnapshot: BlockState,
      autoConnectEdge?: Edge
    ) => {
      // Memoized function to record when a block is duplicated.
      // `sourceBlockId`: The ID of the original block.
      // `duplicatedBlockId`: The ID of the newly created duplicated block.
      // `duplicatedBlockSnapshot`: A snapshot of the *duplicated* block's state.
      // `autoConnectEdge`: Optional edge created during duplication.

      if (!activeWorkflowId) return

      const operation: DuplicateBlockOperation = {
        // Defines the original operation: duplicating a block.
        id: crypto.randomUUID(),
        type: 'duplicate-block',
        timestamp: Date.now(),
        workflowId: activeWorkflowId,
        userId,
        data: {
          sourceBlockId,
          duplicatedBlockId,
          duplicatedBlockSnapshot,
          autoConnectEdge,
        },
      }

      // Inverse is to remove the duplicated block
      const inverse: RemoveBlockOperation = {
        // Defines the inverse operation: removing the duplicated block.
        id: crypto.randomUUID(),
        type: 'remove-block',
        timestamp: Date.now(),
        workflowId: activeWorkflowId,
        userId,
        data: {
          blockId: duplicatedBlockId,
          blockSnapshot: duplicatedBlockSnapshot,
          // Snapshot of the duplicated block is needed to restore it if 'redo' is called.
          edgeSnapshots: autoConnectEdge ? [autoConnectEdge] : [],
          // If an auto-connect edge was created, it's included here to be removed on undo.
        },
      }

      const entry = createOperationEntry(operation, inverse)
      undoRedoStore.push(activeWorkflowId, userId, entry)

      logger.debug('Recorded duplicate block', { sourceBlockId, duplicatedBlockId })
    },
    [activeWorkflowId, userId, undoRedoStore]
  )

  const recordUpdateParent = useCallback(
    (
      blockId: string,
      oldParentId: string | undefined,
      newParentId: string | undefined,
      oldPosition: { x: number; y: number },
      newPosition: { x: number; y: number },
      affectedEdges?: any[] // `any[]` suggests `Edge[]` or a similar type would be more specific.
    ) => {
      // Memoized function to record when a block's parent (nesting) is updated.
      // This happens when a block is moved into or out of a subflow.
      // `blockId`: ID of the block whose parent is updated.
      // `oldParentId`: The ID of the block's parent *before* the update (or undefined if it was a top-level block).
      // `newParentId`: The ID of the block's parent *after* the update (or undefined if it became a top-level block).
      // `oldPosition`, `newPosition`: The block's position *before* and *after* the update.
      // `affectedEdges`: Any edges that were removed or re-added as a result of the parent change (e.g., if moving into a subflow disconnects certain edges).

      if (!activeWorkflowId) return

      const operation: UpdateParentOperation = {
        // Defines the original operation: updating the parent.
        id: crypto.randomUUID(),
        type: 'update-parent',
        timestamp: Date.now(),
        workflowId: activeWorkflowId,
        userId,
        data: {
          blockId,
          oldParentId,
          newParentId,
          oldPosition,
          newPosition,
          affectedEdges,
        },
      }

      const inverse: UpdateParentOperation = {
        // Defines the inverse operation: reverting the parent change.
        // It's also an 'update-parent' operation, but parents and positions are swapped.
        id: crypto.randomUUID(),
        type: 'update-parent',
        timestamp: Date.now(),
        workflowId: activeWorkflowId,
        userId,
        data: {
          blockId,
          oldParentId: newParentId, // The 'new' parent of the original becomes the 'old' of the inverse.
          newParentId: oldParentId, // The 'old' parent of the original becomes the 'new' of the inverse.
          oldPosition: newPosition, // The 'new' position of the original becomes the 'old' of the inverse.
          newPosition: oldPosition, // The 'old' position of the original becomes the 'new' of the inverse.
          affectedEdges, // Same edges need to be considered for restoration/removal.
        },
      }

      const entry = createOperationEntry(operation, inverse)
      undoRedoStore.push(activeWorkflowId, userId, entry)

      logger.debug('Recorded update parent', {
        blockId,
        oldParentId,
        newParentId,
        edgeCount: affectedEdges?.length || 0,
      })
    },
    [activeWorkflowId, userId, undoRedoStore]
  )

  // --- UNDO/REDO LOGIC ---

  const undo = useCallback(() => {
    // Memoized function to perform the undo action.
    if (!activeWorkflowId) return

    const entry = undoRedoStore.undo(activeWorkflowId, userId)
    // Retrieves the latest `OperationEntry` from the undo stack. This also moves the entry to the redo stack.
    if (!entry) {
      logger.debug('No operations to undo')
      return // If there's no entry, nothing to undo.
    }

    const opId = crypto.randomUUID()
    // Generates a unique ID for the current local undo operation (for server queue).

    switch (entry.inverse.type) {
      // The `undo` function executes the `inverse` operation stored in the entry.

      case 'remove-block': {
        // This case handles undoing an 'add-block' operation, which means we now need to remove the block.
        const removeInverse = entry.inverse as RemoveBlockOperation
        const blockId = removeInverse.data.blockId

        if (workflowStore.blocks[blockId]) {
          // Check if the block actually exists in the current workflow state.
          // Refresh inverse snapshot to capture the latest subblock values and edges at undo time
          const mergedNow = mergeSubblockState(workflowStore.blocks, activeWorkflowId, blockId)
          // Gets the latest state of the block, including its sub-block values. This ensures the snapshot is accurate if sub-block values changed since the block was added.
          const latestBlockSnapshot = mergedNow[blockId] || workflowStore.blocks[blockId]
          const latestEdgeSnapshots = workflowStore.edges.filter(
            (e) => e.source === blockId || e.target === blockId
          )
          // Captures current edges connected to the block, as these might have changed.
          removeInverse.data.blockSnapshot = latestBlockSnapshot
          removeInverse.data.edgeSnapshots = latestEdgeSnapshots
          // Updates the inverse's data with the *latest* snapshots, so if this operation is *redone*, it has accurate data.

          // First remove the edges that were added with the block (autoConnect edge)
          const edgesToRemove = removeInverse.data.edgeSnapshots || []
          edgesToRemove.forEach((edge) => {
            // Iterates through the edges that need to be removed.
            if (workflowStore.edges.find((e) => e.id === edge.id)) {
              // Checks if the edge still exists before attempting to remove.
              workflowStore.removeEdge(edge.id)
              // Removes the edge from the local workflow store.
              addToQueue({
                // Adds a server operation to remove the edge.
                id: crypto.randomUUID(),
                operation: {
                  operation: 'remove',
                  target: 'edge',
                  payload: { id: edge.id },
                },
                workflowId: activeWorkflowId,
                userId,
              })
            }
          })

          // Then remove the block
          addToQueue({
            // Adds a server operation to remove the block.
            id: opId,
            operation: {
              operation: 'remove',
              target: 'block',
              payload: { id: blockId, isUndo: true, originalOpId: entry.id },
              // `isUndo: true` indicates this operation is a result of an undo.
              // `originalOpId`: ID of the original operation entry.
            },
            workflowId: activeWorkflowId,
            userId,
          })
          workflowStore.removeBlock(blockId)
          // Removes the block from the local workflow store.
        } else {
          logger.debug('Undo remove-block skipped; block missing', {
            blockId,
          })
        }
        break
      }

      case 'add-block': {
        // This case handles undoing a 'remove-block' operation, which means we now need to re-add the block.
        const originalOp = entry.operation as RemoveBlockOperation
        // Accesses the *original* operation (which was a remove-block) to get the snapshot data.
        const { blockSnapshot, edgeSnapshots, allBlockSnapshots } = originalOp.data
        // Destructures snapshots needed to restore the block and its associated elements.

        if (!blockSnapshot || workflowStore.blocks[blockSnapshot.id]) {
          // Skips if the snapshot is missing or if a block with this ID already exists (to prevent duplicates).
          logger.debug('Undo add-block skipped', {
            hasSnapshot: Boolean(blockSnapshot),
            exists: Boolean(blockSnapshot && workflowStore.blocks[blockSnapshot.id]),
          })
          break
        }

        const currentBlocks = useWorkflowStore.getState().blocks
        const uniqueName = getUniqueBlockName(blockSnapshot.name, currentBlocks)
        // Ensures the restored block has a unique name in the current workflow.

        // FIRST: Add the main block (parent subflow) with subBlocks in payload
        addToQueue({
          // Adds a server operation to add the main block.
          id: opId,
          operation: {
            operation: 'add',
            target: 'block',
            payload: {
              ...blockSnapshot, // Spreads the block's snapshot data.
              name: uniqueName, // Uses the unique name.
              subBlocks: blockSnapshot.subBlocks || {}, // Includes sub-block data.
              autoConnectEdge: undefined, // No auto-connect edge for an undo add.
              isUndo: true,
              originalOpId: entry.id,
            },
          },
          workflowId: activeWorkflowId,
          userId,
        })

        workflowStore.addBlock(
          // Adds the block to the local workflow store.
          blockSnapshot.id,
          blockSnapshot.type,
          uniqueName,
          blockSnapshot.position,
          blockSnapshot.data,
          blockSnapshot.data?.parentId,
          blockSnapshot.data?.extent,
          {
            enabled: blockSnapshot.enabled,
            horizontalHandles: blockSnapshot.horizontalHandles,
            isWide: blockSnapshot.isWide,
            advancedMode: blockSnapshot.advancedMode,
            triggerMode: blockSnapshot.triggerMode,
            height: blockSnapshot.height,
          }
        )

        // Set subblock values for the main block locally
        if (blockSnapshot.subBlocks && activeWorkflowId) {
          // If the block has sub-blocks, restore their values locally.
          const subblockValues: Record<string, any> = {}
          Object.entries(blockSnapshot.subBlocks).forEach(
            ([subBlockId, subBlock]: [string, any]) => {
              if (subBlock.value !== null && subBlock.value !== undefined) {
                subblockValues[subBlockId] = subBlock.value
              }
            }
          )

          if (Object.keys(subblockValues).length > 0) {
            useSubBlockStore.setState((state) => ({
              // Updates the sub-block store with the restored values.
              workflowValues: {
                ...state.workflowValues,
                [activeWorkflowId]: {
                  ...state.workflowValues[activeWorkflowId],
                  [blockSnapshot.id]: subblockValues,
                },
              },
            }))
          }
        }

        // SECOND: If this is a subflow with nested blocks, restore them AFTER the parent exists
        if (allBlockSnapshots) {
          // If `allBlockSnapshots` exists, it means this was a subflow with nested blocks.
          Object.entries(allBlockSnapshots).forEach(([id, snap]: [string, any]) => {
            if (id !== blockSnapshot.id && !workflowStore.blocks[id]) {
              // Iterates through nested block snapshots, skipping the parent block itself, and only adds if not already existing.
              const currentBlocksNested = useWorkflowStore.getState().blocks
              const uniqueNestedName = getUniqueBlockName(snap.name, currentBlocksNested)
              // Ensures unique names for nested blocks.

              // Add nested block locally
              workflowStore.addBlock(
                // Adds the nested block to the local workflow store.
                snap.id,
                snap.type,
                uniqueNestedName,
                snap.position,
                snap.data,
                snap.data?.parentId,
                snap.data?.extent,
                {
                  enabled: snap.enabled,
                  horizontalHandles: snap.horizontalHandles,
                  isWide: snap.isWide,
                  advancedMode: snap.advancedMode,
                  triggerMode: snap.triggerMode,
                  height: snap.height,
                }
              )

              // Send to server with subBlocks included in payload
              addToQueue({
                // Adds a server operation to add the nested block.
                id: crypto.randomUUID(),
                operation: {
                  operation: 'add',
                  target: 'block',
                  payload: {
                    ...snap,
                    name: uniqueNestedName,
                    subBlocks: snap.subBlocks || {},
                    autoConnectEdge: undefined,
                    isUndo: true,
                    originalOpId: entry.id,
                  },
                },
                workflowId: activeWorkflowId,
                userId,
              })

              // Restore subblock values for nested blocks locally
              if (snap.subBlocks && activeWorkflowId) {
                const subBlockStore = useSubBlockStore.getState()
                Object.entries(snap.subBlocks).forEach(([subBlockId, subBlock]: [string, any]) => {
                  if (subBlock.value !== null && subBlock.value !== undefined) {
                    subBlockStore.setValue(snap.id, subBlockId, subBlock.value)
                  }
                })
              }
            }
          })
        }

        // THIRD: Finally restore edges after all blocks exist
        if (edgeSnapshots && edgeSnapshots.length > 0) {
          // Restores edges associated with the block(s).
          edgeSnapshots.forEach((edge) => {
            workflowStore.addEdge(edge) // Adds edge to local store.
            addToQueue({
              // Adds server operation to add the edge.
              id: crypto.randomUUID(),
              operation: {
                operation: 'add',
                target: 'edge',
                payload: edge,
              },
              workflowId: activeWorkflowId,
              userId,
            })
          })
        }
        break
      }

      case 'remove-edge': {
        // This case handles undoing an 'add-edge' operation, meaning we need to remove the edge.
        const removeEdgeInverse = entry.inverse as RemoveEdgeOperation
        const { edgeId } = removeEdgeInverse.data
        if (workflowStore.edges.find((e) => e.id === edgeId)) {
          // Checks if the edge still exists before removing.
          addToQueue({
            // Adds server operation to remove the edge.
            id: opId,
            operation: {
              operation: 'remove',
              target: 'edge',
              payload: {
                id: edgeId,
                isUndo: true,
                originalOpId: entry.id,
              },
            },
            workflowId: activeWorkflowId,
            userId,
          })
          workflowStore.removeEdge(edgeId)
          // Removes the edge from the local workflow store.
        } else {
          logger.debug('Undo remove-edge skipped; edge missing', {
            edgeId,
          })
        }
        break
      }

      case 'add-edge': {
        // This case handles undoing a 'remove-edge' operation, meaning we need to re-add the edge.
        const originalOp = entry.operation as RemoveEdgeOperation
        const { edgeSnapshot } = originalOp.data
        // Accesses the original operation to get the edge snapshot.
        if (!edgeSnapshot || workflowStore.edges.find((e) => e.id === edgeSnapshot.id)) {
          // Skips if snapshot missing or edge already exists.
          logger.debug('Undo add-edge skipped', {
            hasSnapshot: Boolean(edgeSnapshot),
          })
          break
        }
        addToQueue({
          // Adds server operation to add the edge.
          id: opId,
          operation: {
            operation: 'add',
            target: 'edge',
            payload: { ...edgeSnapshot, isUndo: true, originalOpId: entry.id },
          },
          workflowId: activeWorkflowId,
          userId,
        })
        workflowStore.addEdge(edgeSnapshot)
        // Adds the edge to the local workflow store.
        break
      }

      case 'move-block': {
        // This case handles undoing a 'move-block' operation, meaning we need to move the block back to its original position.
        const moveOp = entry.inverse as MoveBlockOperation
        // Accesses the inverse operation, which contains the `before` and `after` positions (swapped).
        const currentBlocks = useWorkflowStore.getState().blocks
        if (currentBlocks[moveOp.data.blockId]) {
          // Checks if the block exists.
          // Apply the inverse's target as the undo result (inverse.after)
          addToQueue({
            // Adds server operation to update the block's position.
            id: opId,
            operation: {
              operation: 'update-position',
              target: 'block',
              payload: {
                id: moveOp.data.blockId,
                position: { x: moveOp.data.after.x, y: moveOp.data.after.y }, // Reverts to the inverse's `after` position (which is the original `before`).
                parentId: moveOp.data.after.parentId,
                commit: true,
                isUndo: true,
                originalOpId: entry.id,
              },
            },
            workflowId: activeWorkflowId,
            userId,
          })
          // Use the store from the hook context for React re-renders
          workflowStore.updateBlockPosition(moveOp.data.blockId, {
            x: moveOp.data.after.x,
            y: moveOp.data.after.y,
          })
          // Updates local block position.
          if (moveOp.data.after.parentId !== moveOp.data.before.parentId) {
            // If parent ID also changed in the move, revert that.
            workflowStore.updateParentId(
              moveOp.data.blockId,
              moveOp.data.after.parentId || '',
              'parent'
            )
          }
        } else {
          logger.debug('Undo move-block skipped; block missing', {
            blockId: moveOp.data.blockId,
          })
        }
        break
      }

      case 'duplicate-block': {
        // Undo duplicate means removing the duplicated block.
        const dupOp = entry.operation as DuplicateBlockOperation
        // Accesses the original operation to get the ID of the duplicated block.
        const duplicatedId = dupOp.data.duplicatedBlockId

        if (workflowStore.blocks[duplicatedId]) {
          // Remove any edges connected to the duplicated block
          const edges = workflowStore.edges.filter(
            (edge) => edge.source === duplicatedId || edge.target === duplicatedId
          )
          edges.forEach((edge) => {
            // Removes associated edges locally and sends to server.
            workflowStore.removeEdge(edge.id)
            addToQueue({
              id: crypto.randomUUID(),
              operation: {
                operation: 'remove',
                target: 'edge',
                payload: { id: edge.id },
              },
              workflowId: activeWorkflowId,
              userId,
            })
          })

          // Remove the duplicated block
          addToQueue({
            // Adds server operation to remove the duplicated block.
            id: opId,
            operation: {
              operation: 'remove',
              target: 'block',
              payload: { id: duplicatedId, isUndo: true, originalOpId: entry.id },
            },
            workflowId: activeWorkflowId,
            userId,
          })
          workflowStore.removeBlock(duplicatedId)
          // Removes the duplicated block locally.
        } else {
          logger.debug('Undo duplicate-block skipped; duplicated block missing', {
            duplicatedId,
          })
        }
        break
      }

      case 'update-parent': {
        // Undo parent update means reverting to the old parent and position.
        const updateOp = entry.inverse as UpdateParentOperation
        // Accesses the inverse operation, which has the `old` and `new` parents/positions swapped.
        const { blockId, newParentId, newPosition, affectedEdges } = updateOp.data
        // `newParentId` here is actually the `oldParentId` from the original operation, etc.

        if (workflowStore.blocks[blockId]) {
          // If we're moving back INTO a subflow, restore edges first
          if (newParentId && affectedEdges && affectedEdges.length > 0) {
            // If the block is moving *into* a parent (i.e., `newParentId` is defined), re-add any affected edges that should now exist.
            affectedEdges.forEach((edge) => {
              if (!workflowStore.edges.find((e) => e.id === edge.id)) {
                workflowStore.addEdge(edge)
                addToQueue({
                  id: crypto.randomUUID(),
                  operation: {
                    operation: 'add',
                    target: 'edge',
                    payload: { ...edge, isUndo: true },
                  },
                  workflowId: activeWorkflowId,
                  userId,
                })
              }
            })
          }

          // Send position update to server
          addToQueue({
            id: crypto.randomUUID(),
            operation: {
              operation: 'update-position',
              target: 'block',
              payload: {
                id: blockId,
                position: newPosition, // Reverts to the inverse's `newPosition` (original `oldPosition`).
                commit: true,
                isUndo: true,
                originalOpId: entry.id,
              },
            },
            workflowId: activeWorkflowId,
            userId,
          })

          // Update position locally
          workflowStore.updateBlockPosition(blockId, newPosition)

          // Send parent update to server
          addToQueue({
            id: opId,
            operation: {
              operation: 'update-parent',
              target: 'block',
              payload: {
                id: blockId,
                parentId: newParentId || '', // Reverts to the inverse's `newParentId` (original `oldParentId`).
                extent: 'parent',
                isUndo: true,
                originalOpId: entry.id,
              },
            },
            workflowId: activeWorkflowId,
            userId,
          })

          // Update parent locally
          workflowStore.updateParentId(blockId, newParentId || '', 'parent')

          // If we're removing FROM a subflow (undo of add to subflow), remove edges after
          if (!newParentId && affectedEdges && affectedEdges.length > 0) {
            // If the block is moving *out of* a parent (i.e., `newParentId` is undefined), remove any affected edges that should no longer exist.
            affectedEdges.forEach((edge) => {
              if (workflowStore.edges.find((e) => e.id === edge.id)) {
                workflowStore.removeEdge(edge.id)
                addToQueue({
                  id: crypto.randomUUID(),
                  operation: {
                    operation: 'remove',
                    target: 'edge',
                    payload: { id: edge.id, isUndo: true },
                  },
                  workflowId: activeWorkflowId,
                  userId,
                })
              }
            })
          }
        } else {
          logger.debug('Undo update-parent skipped; block missing', { blockId })
        }
        break
      }
    }

    logger.info('Undo operation', { type: entry.operation.type, workflowId: activeWorkflowId })
  }, [activeWorkflowId, userId, undoRedoStore, addToQueue, workflowStore])
  // Dependencies for `undo`: `workflowStore` and `addToQueue` are needed to interact with local state and send server operations.

  const redo = useCallback(() => {
    // Memoized function to perform the redo action.
    if (!activeWorkflowId || !userId) return

    const entry = undoRedoStore.redo(activeWorkflowId, userId)
    // Retrieves the latest `OperationEntry` from the redo stack. This also moves the entry back to the undo stack.
    if (!entry) {
      logger.debug('No operations to redo')
      return // If no entry, nothing to redo.
    }

    const opId = crypto.randomUUID()
    // Generates a unique ID for the current local redo operation (for server queue).

    switch (entry.operation.type) {
      // The `redo` function executes the *original* operation stored in the entry.

      case 'add-block': {
        // This case handles redoing an 'add-block' operation, meaning we need to re-add the block.
        const inv = entry.inverse as RemoveBlockOperation
        // Accesses the inverse operation to get the snapshots, as the original add-block operation might not have contained the full snapshot of the *resulting* block.
        const snap = inv.data.blockSnapshot
        const edgeSnapshots = inv.data.edgeSnapshots || []
        const allBlockSnapshots = inv.data.allBlockSnapshots

        if (!snap || workflowStore.blocks[snap.id]) {
          // Skips if snapshot missing or block already exists.
          break
        }

        const currentBlocks = useWorkflowStore.getState().blocks
        const uniqueName = getUniqueBlockName(snap.name, currentBlocks)

        // FIRST: Add the main block (parent subflow) with subBlocks included
        addToQueue({
          // Adds server operation to add the main block.
          id: opId,
          operation: {
            operation: 'add',
            target: 'block',
            payload: {
              ...snap,
              name: uniqueName,
              subBlocks: snap.subBlocks || {},
              isRedo: true, // `isRedo: true` indicates this operation is a result of a redo.
              originalOpId: entry.id,
            },
          },
          workflowId: activeWorkflowId,
          userId,
        })

        workflowStore.addBlock(
          // Adds the block to the local workflow store.
          snap.id,
          snap.type,
          uniqueName,
          snap.position,
          snap.data,
          snap.data?.parentId,
          snap.data?.extent,
          {
            enabled: snap.enabled,
            horizontalHandles: snap.horizontalHandles,
            isWide: snap.isWide,
            advancedMode: snap.advancedMode,
            triggerMode: snap.triggerMode,
            height: snap.height,
          }
        )

        // Set subblock values for the main block locally
        if (snap.subBlocks && activeWorkflowId) {
          const subblockValues: Record<string, any> = {}
          Object.entries(snap.subBlocks).forEach(([subBlockId, subBlock]: [string, any]) => {
            if (subBlock.value !== null && subBlock.value !== undefined) {
              subblockValues[subBlockId] = subBlock.value
            }
          })

          if (Object.keys(subblockValues).length > 0) {
            useSubBlockStore.setState((state) => ({
              workflowValues: {
                ...state.workflowValues,
                [activeWorkflowId]: {
                  ...state.workflowValues[activeWorkflowId],
                  [snap.id]: subblockValues,
                },
              },
            }))
          }
        }

        // SECOND: If this is a subflow with nested blocks, restore them AFTER the parent exists
        if (allBlockSnapshots) {
          Object.entries(allBlockSnapshots).forEach(([id, snapNested]: [string, any]) => {
            if (id !== snap.id && !workflowStore.blocks[id]) {
              const currentBlocksNested = useWorkflowStore.getState().blocks
              const uniqueNestedName = getUniqueBlockName(snapNested.name, currentBlocksNested)

              // Add nested block locally
              workflowStore.addBlock(
                snapNested.id,
                snapNested.type,
                uniqueNestedName,
                snapNested.position,
                snapNested.data,
                snapNested.data?.parentId,
                snapNested.data?.extent,
                {
                  enabled: snapNested.enabled,
                  horizontalHandles: snapNested.horizontalHandles,
                  isWide: snapNested.isWide,
                  advancedMode: snapNested.advancedMode,
                  triggerMode: snapNested.triggerMode,
                  height: snapNested.height,
                }
              )

              // Send to server with subBlocks included
              addToQueue({
                id: crypto.randomUUID(),
                operation: {
                  operation: 'add',
                  target: 'block',
                  payload: {
                    ...snapNested,
                    name: uniqueNestedName,
                    subBlocks: snapNested.subBlocks || {},
                    autoConnectEdge: undefined,
                    isRedo: true,
                    originalOpId: entry.id,
                  },
                },
                workflowId: activeWorkflowId,
                userId,
              })

              // Restore subblock values for nested blocks locally
              if (snapNested.subBlocks && activeWorkflowId) {
                const subBlockStore = useSubBlockStore.getState()
                Object.entries(snapNested.subBlocks).forEach(
                  ([subBlockId, subBlock]: [string, any]) => {
                    if (subBlock.value !== null && subBlock.value !== undefined) {
                      subBlockStore.setValue(snapNested.id, subBlockId, subBlock.value)
                    }
                  }
                )
              }
            }
          })
        }

        // THIRD: Finally restore edges after all blocks exist
        edgeSnapshots.forEach((edge) => {
          if (!workflowStore.edges.find((e) => e.id === edge.id)) {
            workflowStore.addEdge(edge)
            addToQueue({
              id: crypto.randomUUID(),
              operation: {
                operation: 'add',
                target: 'edge',
                payload: { ...edge, isRedo: true, originalOpId: entry.id },
              },
              workflowId: activeWorkflowId,
              userId,
            })
          }
        })
        break
      }

      case 'remove-block': {
        // This case handles redoing a 'remove-block' operation, meaning we need to remove the block again.
        const blockId = entry.operation.data.blockId
        const edgesToRemove = (entry.operation as RemoveBlockOperation).data.edgeSnapshots || []
        edgesToRemove.forEach((edge) => {
          // Removes associated edges locally and sends to server.
          if (workflowStore.edges.find((e) => e.id === edge.id)) {
            workflowStore.removeEdge(edge.id)
            addToQueue({
              id: crypto.randomUUID(),
              operation: {
                operation: 'remove',
                target: 'edge',
                payload: { id: edge.id, isRedo: true, originalOpId: entry.id },
              },
              workflowId: activeWorkflowId,
              userId,
            })
          }
        })

        if (workflowStore.blocks[blockId]) {
          // Adds server operation to remove the block.
          addToQueue({
            id: opId,
            operation: {
              operation: 'remove',
              target: 'block',
              payload: { id: blockId, isRedo: true, originalOpId: entry.id },
            },
            workflowId: activeWorkflowId,
            userId,
          })
          workflowStore.removeBlock(blockId)
          // Removes the block locally.
        } else {
          logger.debug('Redo remove-block skipped; block missing', { blockId })
        }
        break
      }

      case 'add-edge': {
        // This case handles redoing an 'add-edge' operation, meaning we need to re-add the edge.
        const inv = entry.inverse as RemoveEdgeOperation
        const snap = inv.data.edgeSnapshot
        // Accesses the inverse operation to get the edge snapshot.
        if (!snap || workflowStore.edges.find((e) => e.id === snap.id)) {
          // Skips if snapshot missing or edge already exists.
          logger.debug('Redo add-edge skipped', { hasSnapshot: Boolean(snap) })
          break
        }
        addToQueue({
          // Adds server operation to add the edge.
          id: opId,
          operation: {
            operation: 'add',
            target: 'edge',
            payload: { ...snap, isRedo: true, originalOpId: entry.id },
          },
          workflowId: activeWorkflowId,
          userId,
        })
        workflowStore.addEdge(snap)
        // Adds the edge locally.
        break
      }

      case 'remove-edge': {
        // This case handles redoing a 'remove-edge' operation, meaning we need to remove the edge again.
        const { edgeId } = entry.operation.data
        if (workflowStore.edges.find((e) => e.id === edgeId)) {
          // Checks if the edge exists before removing.
          addToQueue({
            // Adds server operation to remove the edge.
            id: opId,
            operation: {
              operation: 'remove',
              target: 'edge',
              payload: { id: edgeId, isRedo: true, originalOpId: entry.id },
            },
            workflowId: activeWorkflowId,
            userId,
          })
          workflowStore.removeEdge(edgeId)
          // Removes the edge locally.
        } else {
          logger.debug('Redo remove-edge skipped; edge missing', {
            edgeId,
          })
        }
        break
      }

      case 'move-block': {
        // This case handles redoing a 'move-block' operation, meaning we need to move the block to its `after` position.
        const moveOp = entry.operation as MoveBlockOperation
        const currentBlocks = useWorkflowStore.getState().blocks
        if (currentBlocks[moveOp.data.blockId]) {
          // Checks if block exists.
          addToQueue({
            // Adds server operation to update position.
            id: opId,
            operation: {
              operation: 'update-position',
              target: 'block',
              payload: {
                id: moveOp.data.blockId,
                position: { x: moveOp.data.after.x, y: moveOp.data.after.y }, // Applies the original `after` position.
                parentId: moveOp.data.after.parentId,
                isRedo: true,
                originalOpId: entry.id,
              },
            },
            workflowId: activeWorkflowId,
            userId,
          })
          // Use the store from the hook context for React re-renders
          workflowStore.updateBlockPosition(moveOp.data.blockId, {
            x: moveOp.data.after.x,
            y: moveOp.data.after.y,
          })
          // Updates local position.
          if (moveOp.data.after.parentId !== moveOp.data.before.parentId) {
            // If parent ID changed in the move, re-apply that.
            workflowStore.updateParentId(
              moveOp.data.blockId,
              moveOp.data.after.parentId || '',
              'parent'
            )
          }
        } else {
          logger.debug('Redo move-block skipped; block missing', {
            blockId: moveOp.data.blockId,
          })
        }
        break
      }

      case 'duplicate-block': {
        // Redo duplicate means re-adding the duplicated block.
        const dupOp = entry.operation as DuplicateBlockOperation
        const { duplicatedBlockSnapshot, autoConnectEdge } = dupOp.data

        if (!duplicatedBlockSnapshot || workflowStore.blocks[duplicatedBlockSnapshot.id]) {
          // Skips if snapshot missing or duplicated block already exists.
          logger.debug('Redo duplicate-block skipped', {
            hasSnapshot: Boolean(duplicatedBlockSnapshot),
            exists: Boolean(
              duplicatedBlockSnapshot && workflowStore.blocks[duplicatedBlockSnapshot.id]
            ),
          })
          break
        }

        const currentBlocks = useWorkflowStore.getState().blocks
        const uniqueName = getUniqueBlockName(duplicatedBlockSnapshot.name, currentBlocks)

        // Add the duplicated block
        addToQueue({
          // Adds server operation to duplicate (add) the block.
          id: opId,
          operation: {
            operation: 'duplicate',
            target: 'block',
            payload: {
              ...duplicatedBlockSnapshot,
              name: uniqueName,
              subBlocks: duplicatedBlockSnapshot.subBlocks || {},
              autoConnectEdge,
              isRedo: true,
              originalOpId: entry.id,
            },
          },
          workflowId: activeWorkflowId,
          userId,
        })

        workflowStore.addBlock(
          // Adds the duplicated block locally.
          duplicatedBlockSnapshot.id,
          duplicatedBlockSnapshot.type,
          uniqueName,
          duplicatedBlockSnapshot.position,
          duplicatedBlockSnapshot.data,
          duplicatedBlockSnapshot.data?.parentId,
          duplicatedBlockSnapshot.data?.extent,
          {
            enabled: duplicatedBlockSnapshot.enabled,
            horizontalHandles: duplicatedBlockSnapshot.horizontalHandles,
            isWide: duplicatedBlockSnapshot.isWide,
            advancedMode: duplicatedBlockSnapshot.advancedMode,
            triggerMode: duplicatedBlockSnapshot.triggerMode,
            height: duplicatedBlockSnapshot.height,
          }
        )

        // Restore subblock values
        if (duplicatedBlockSnapshot.subBlocks && activeWorkflowId) {
          const subblockValues: Record<string, any> = {}
          Object.entries(duplicatedBlockSnapshot.subBlocks).forEach(
            ([subBlockId, subBlock]: [string, any]) => {
              if (subBlock.value !== null && subBlock.value !== undefined) {
                subblockValues[subBlockId] = subBlock.value
              }
            }
          )

          if (Object.keys(subblockValues).length > 0) {
            useSubBlockStore.setState((state) => ({
              workflowValues: {
                ...state.workflowValues,
                [activeWorkflowId]: {
                  ...state.workflowValues[activeWorkflowId],
                  [duplicatedBlockSnapshot.id]: subblockValues,
                },
              },
            }))
          }
        }

        // Add auto-connect edge if present
        if (autoConnectEdge && !workflowStore.edges.find((e) => e.id === autoConnectEdge.id)) {
          // If there was an auto-connect edge, re-add it locally and send to server.
          workflowStore.addEdge(autoConnectEdge)
          addToQueue({
            id: crypto.randomUUID(),
            operation: {
              operation: 'add',
              target: 'edge',
              payload: { ...autoConnectEdge, isRedo: true, originalOpId: entry.id },
            },
            workflowId: activeWorkflowId,
            userId,
          })
        }
        break
      }

      case 'update-parent': {
        // Redo parent update means applying the new parent and position.
        const updateOp = entry.operation as UpdateParentOperation
        // Accesses the original operation.
        const { blockId, newParentId, newPosition, affectedEdges } = updateOp.data

        if (workflowStore.blocks[blockId]) {
          // If we're removing FROM a subflow, remove edges first
          if (!newParentId && affectedEdges && affectedEdges.length > 0) {
            // If the block is moving *out of* a parent (i.e., `newParentId` is undefined), remove affected edges.
            affectedEdges.forEach((edge) => {
              if (workflowStore.edges.find((e) => e.id === edge.id)) {
                workflowStore.removeEdge(edge.id)
                addToQueue({
                  id: crypto.randomUUID(),
                  operation: {
                    operation: 'remove',
                    target: 'edge',
                    payload: { id: edge.id, isRedo: true },
                  },
                  workflowId: activeWorkflowId,
                  userId,
                })
              }
            })
          }

          // Send position update to server
          addToQueue({
            id: crypto.randomUUID(),
            operation: {
              operation: 'update-position',
              target: 'block',
              payload: {
                id: blockId,
                position: newPosition, // Applies the original `newPosition`.
                isRedo: true,
                originalOpId: entry.id,
              },
            },
            workflowId: activeWorkflowId,
            userId,
          })

          // Update position locally
          workflowStore.updateBlockPosition(blockId, newPosition)

          // Send parent update to server
          addToQueue({
            id: opId,
            operation: {
              operation: 'update-parent',
              target: 'block',
              payload: {
                id: blockId,
                parentId: newParentId || '', // Applies the original `newParentId`.
                extent: 'parent',
                isRedo: true,
                originalOpId: entry.id,
              },
            },
            workflowId: activeWorkflowId,
            userId,
          })

          // Update parent locally
          workflowStore.updateParentId(blockId, newParentId || '', 'parent')

          // If we're adding TO a subflow, restore edges after
          if (newParentId && affectedEdges && affectedEdges.length > 0) {
            // If the block is moving *into* a parent (i.e., `newParentId` is defined), re-add affected edges.
            affectedEdges.forEach((edge) => {
              if (!workflowStore.edges.find((e) => e.id === edge.id)) {
                workflowStore.addEdge(edge)
                addToQueue({
                  id: crypto.randomUUID(),
                  operation: {
                    operation: 'add',
                    target: 'edge',
                    payload: { ...edge, isRedo: true },
                  },
                  workflowId: activeWorkflowId,
                  userId,
                })
              }
            })
          }
        } else {
          logger.debug('Redo update-parent skipped; block missing', { blockId })
        }
        break
      }
    }

    logger.info('Redo operation completed', {
      type: entry.operation.type,
      workflowId: activeWorkflowId,
      userId,
    })
  }, [activeWorkflowId, userId, undoRedoStore, addToQueue, workflowStore])
  // Dependencies for `redo`: `workflowStore` and `addToQueue` are needed to interact with local state and send server operations.

  const getStackSizes = useCallback(() => {
    // Memoized function to get the current number of operations in the undo and redo stacks.
    if (!activeWorkflowId) return { undoSize: 0, redoSize: 0 }
    return undoRedoStore.getStackSizes(activeWorkflowId, userId)
    // Delegates to the `undoRedoStore` to get the sizes specific to the active workflow and user.
  }, [activeWorkflowId, userId, undoRedoStore])

  const clearStacks = useCallback(() => {
    // Memoized function to clear both the undo and redo stacks.
    if (!activeWorkflowId) return
    undoRedoStore.clear(activeWorkflowId, userId)
    // Delegates to the `undoRedoStore` to clear stacks for the active workflow and user.
  }, [activeWorkflowId, userId, undoRedoStore])

  return {
    // Returns an object containing all the functions provided by this hook, making them available to components that use `useUndoRedo`.
    recordAddBlock,
    recordRemoveBlock,
    recordAddEdge,
    recordRemoveEdge,
    recordMove,
    recordDuplicateBlock,
    recordUpdateParent,
    undo,
    redo,
    getStackSizes,
    clearStacks,
  }
}
```