```typescript
import type { Operation, OperationEntry } from './types'

/**
 * Purpose of this file:
 *
 * This file defines utility functions for working with `Operation` objects.  `Operation` objects likely represent changes made within a visual editor or diagramming tool.
 * The functions in this file provide the following capabilities:
 *
 * 1. **Creating `OperationEntry` objects:**  Wrapping an `Operation` and its inverse with additional metadata like a unique ID and timestamp.
 * 2. **Generating inverse `Operation` objects:**  Creating the opposite operation that would undo a given operation (e.g., the inverse of adding a block is removing it). This is crucial for implementing undo/redo functionality.
 * 3. **Transforming `Operation` objects into a collaborative payload:** Converts the Operation into a standard format that can be used to communicate changes over a network to keep multiple clients in sync.
 */

/**
 * Creates an `OperationEntry` object.
 *
 * An `OperationEntry` likely represents a single action performed by the user
 * and is used to keep track of operations and their inverses for implementing
 * features like undo/redo.
 *
 * @param {Operation} operation - The primary operation being performed.
 * @param {Operation} inverse - The inverse operation that would undo the primary operation.
 * @returns {OperationEntry} A new `OperationEntry` object.
 */
export function createOperationEntry(operation: Operation, inverse: Operation): OperationEntry {
  return {
    // Generate a unique ID for the operation entry using the crypto API.  This ensures each entry has a distinct identifier.
    id: crypto.randomUUID(),
    // Store the primary operation.
    operation,
    // Store the inverse operation.
    inverse,
    // Record the timestamp when the operation entry was created using `Date.now()`, which returns the number of milliseconds since the Unix epoch.
    createdAt: Date.now(),
  }
}

/**
 * Creates the inverse of a given `Operation`.
 *
 * This function is the heart of the undo/redo functionality. It takes an operation
 * and returns a new operation that, when executed, will reverse the effect of the original operation.
 *
 * The logic is based on a `switch` statement that handles different `operation.type` values.
 * For each operation type, it creates a new operation object with the appropriate changes to reverse the action.
 *
 * @param {Operation} operation - The original operation to invert.
 * @returns {Operation} The inverse operation.
 */
export function createInverseOperation(operation: Operation): Operation {
  switch (operation.type) {
    // Case: Adding a block. The inverse is to remove the block.
    case 'add-block':
      return {
        // Copy all properties from the original operation.
        ...operation,
        // Change the operation type to 'remove-block'.
        type: 'remove-block',
        // The data for removing a block includes the blockId and potentially snapshots of the block and its edges.
        data: {
          // The blockId to remove is the same as the one added.
          blockId: operation.data.blockId,
          // `blockSnapshot: null` is present here because when removing the block, we remove the snapshot as well
          blockSnapshot: null,
          // Edge snapshots would be removed together with the block.
          edgeSnapshots: [],
        },
      }

    // Case: Removing a block. The inverse is to add the block back.
    case 'remove-block':
      return {
        // Copy all properties from the original operation.
        ...operation,
        // Change the operation type to 'add-block'.
        type: 'add-block',
        // The data for adding a block back is just the blockId.  The block data itself would ideally be retrieved from a snapshot.
        data: {
          blockId: operation.data.blockId,
        },
      }

    // Case: Adding an edge.  The inverse is to remove the edge.
    case 'add-edge':
      return {
        // Copy all properties from the original operation.
        ...operation,
        // Change the operation type to 'remove-edge'.
        type: 'remove-edge',
        // The data for removing an edge includes the edgeId.
        data: {
          edgeId: operation.data.edgeId,
          edgeSnapshot: null,
        },
      }

    // Case: Removing an edge.  The inverse is to add the edge back.
    case 'remove-edge':
      return {
        // Copy all properties from the original operation.
        ...operation,
        // Change the operation type to 'add-edge'.
        type: 'add-edge',
        // The data for adding an edge back is the edgeId. The edge data itself would be retrieved from a snapshot.
        data: {
          edgeId: operation.data.edgeId,
        },
      }

    // Case: Adding a subflow. The inverse is to remove the subflow.
    case 'add-subflow':
      return {
        // Copy all properties from the original operation.
        ...operation,
        // Change the operation type to 'remove-subflow'.
        type: 'remove-subflow',
        // The data for removing an subflow includes the subflowId and potentially snapshots of the subflow.
        data: {
          subflowId: operation.data.subflowId,
          subflowSnapshot: null,
        },
      }

    // Case: Removing a subflow.  The inverse is to add the subflow back.
    case 'remove-subflow':
      return {
        // Copy all properties from the original operation.
        ...operation,
        // Change the operation type to 'add-subflow'.
        type: 'add-subflow',
        // The data for adding a subflow back is the subflowId. The subflow data itself would be retrieved from a snapshot.
        data: {
          subflowId: operation.data.subflowId,
        },
      }

    // Case: Moving a block. The inverse is to move it back to its original position.
    case 'move-block':
      return {
        // Copy all properties from the original operation.
        ...operation,
        // Switch the 'before' and 'after' positions in the data.  This effectively moves the block back to its previous location.
        data: {
          blockId: operation.data.blockId,
          before: operation.data.after,
          after: operation.data.before,
        },
      }

    // Case: Moving a subflow. The inverse is to move it back to its original position.
    case 'move-subflow':
      return {
        // Copy all properties from the original operation.
        ...operation,
        // Switch the 'before' and 'after' positions in the data.  This effectively moves the subflow back to its previous location.
        data: {
          subflowId: operation.data.subflowId,
          before: operation.data.after,
          after: operation.data.before,
        },
      }

    // Case: Duplicating a block. The inverse is to remove the duplicated block.
    case 'duplicate-block':
      return {
        // Copy all properties from the original operation.
        ...operation,
        // Change the operation type to 'remove-block'.
        type: 'remove-block',
        data: {
          //Remove the duplicated block
          blockId: operation.data.duplicatedBlockId,
          blockSnapshot: operation.data.duplicatedBlockSnapshot,
          edgeSnapshots: [],
        },
      }

    // Case: Updating the parent of the block. The inverse is to update the block to its previous parent
    case 'update-parent':
      return {
        ...operation,
        data: {
          blockId: operation.data.blockId,
          oldParentId: operation.data.newParentId,
          newParentId: operation.data.oldParentId,
          oldPosition: operation.data.newPosition,
          newPosition: operation.data.oldPosition,
          affectedEdges: operation.data.affectedEdges,
        },
      }

    // Default case: If the operation type is not handled, throw an error.  This is important for ensuring that all operation types are accounted for.
    default: {
      // This is an exhaustive check to ensure all cases are handled.
      const exhaustiveCheck: never = operation
      // Throw an error indicating the unhandled operation type.
      throw new Error(`Unhandled operation type: ${(exhaustiveCheck as any).type}`)
    }
  }
}

/**
 * Transforms an `Operation` object into a collaborative payload.
 *
 * This function translates operations into a format suitable for collaborative environments.
 * It defines the type of operation and the target of the operation (e.g. block, edge, subflow)
 * to facilitate syncing changes between multiple clients.
 *
 * @param {Operation} operation - The original operation to transform.
 * @returns {{ operation: string; target: string; payload: any }} The collaborative payload.
 */
export function operationToCollaborativePayload(operation: Operation): {
  operation: string
  target: string
  payload: any
} {
  switch (operation.type) {
    // Case: Adding a block.
    case 'add-block':
      return {
        // The collaboration operation is 'add'.
        operation: 'add',
        // The target is 'block'.
        target: 'block',
        // The payload contains the block's ID.
        payload: { id: operation.data.blockId },
      }

    // Case: Removing a block.
    case 'remove-block':
      return {
        // The collaboration operation is 'remove'.
        operation: 'remove',
        // The target is 'block'.
        target: 'block',
        // The payload contains the block's ID.
        payload: { id: operation.data.blockId },
      }

    // Case: Adding an edge.
    case 'add-edge':
      return {
        // The collaboration operation is 'add'.
        operation: 'add',
        // The target is 'edge'.
        target: 'edge',
        // The payload contains the edge's ID.
        payload: { id: operation.data.edgeId },
      }

    // Case: Removing an edge.
    case 'remove-edge':
      return {
        // The collaboration operation is 'remove'.
        operation: 'remove',
        // The target is 'edge'.
        target: 'edge',
        // The payload contains the edge's ID.
        payload: { id: operation.data.edgeId },
      }

    // Case: Adding a subflow.
    case 'add-subflow':
      return {
        // The collaboration operation is 'add'.
        operation: 'add',
        // The target is 'subflow'.
        target: 'subflow',
        // The payload contains the subflow's ID.
        payload: { id: operation.data.subflowId },
      }

    // Case: Removing a subflow.
    case 'remove-subflow':
      return {
        // The collaboration operation is 'remove'.
        operation: 'remove',
        // The target is 'subflow'.
        target: 'subflow',
        // The payload contains the subflow's ID.
        payload: { id: operation.data.subflowId },
      }

    // Case: Moving a block.
    case 'move-block':
      return {
        // The collaboration operation is 'update-position'.
        operation: 'update-position',
        // The target is 'block'.
        target: 'block',
        // The payload contains the block's ID and new coordinates.
        payload: {
          id: operation.data.blockId,
          x: operation.data.after.x,
          y: operation.data.after.y,
          parentId: operation.data.after.parentId,
        },
      }

    // Case: Moving a subflow.
    case 'move-subflow':
      return {
        // The collaboration operation is 'update-position'.
        operation: 'update-position',
        // The target is 'subflow'.
        target: 'subflow',
        // The payload contains the subflow's ID and new coordinates.
        payload: {
          id: operation.data.subflowId,
          x: operation.data.after.x,
          y: operation.data.after.y,
        },
      }

    // Case: Duplicating a block.
    case 'duplicate-block':
      return {
        // The collaboration operation is 'duplicate'.
        operation: 'duplicate',
        // The target is 'block'.
        target: 'block',
        // The payload contains the source block id and the duplicated block id.
        payload: {
          sourceId: operation.data.sourceBlockId,
          duplicatedId: operation.data.duplicatedBlockId,
        },
      }

    // Case: Updating the parent of a block.
    case 'update-parent':
      return {
        // The collaboration operation is 'update-parent'.
        operation: 'update-parent',
        // The target is 'block'.
        target: 'block',
        // The payload contains the block's ID, new parent id, and new coordinates.
        payload: {
          id: operation.data.blockId,
          parentId: operation.data.newParentId,
          x: operation.data.newPosition.x,
          y: operation.data.newPosition.y,
        },
      }

    // Default case: If the operation type is not handled, throw an error.  This is important for ensuring that all operation types are accounted for.
    default: {
      // This is an exhaustive check to ensure all cases are handled.
      const exhaustiveCheck: never = operation
      // Throw an error indicating the unhandled operation type.
      throw new Error(`Unhandled operation type: ${(exhaustiveCheck as any).type}`)
    }
  }
}
```