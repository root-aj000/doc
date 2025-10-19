```typescript
import type { Edge } from 'reactflow'
import { create } from 'zustand'
import { persist } from 'zustand/middleware'
import { createLogger } from '@/lib/logs/console/logger'
import type { BlockState } from '@/stores/workflows/workflow/types'
import type {
  MoveBlockOperation,
  Operation,
  OperationEntry,
  RemoveBlockOperation,
  RemoveEdgeOperation,
  UndoRedoState,
} from './types'

// Purpose of this file:
//
// This file defines a Zustand store for managing undo/redo functionality within a workflow editor.
// It utilizes the `zustand` library for state management and `zustand/middleware` for persisting the state to local storage.
// The store keeps track of operations performed on the workflow, allowing users to undo and redo these operations.
// It also includes logic for coalescing consecutive move operations and pruning invalid operations to maintain a clean and efficient undo/redo history.

// Explanation:

// 1. Imports:
//   - `Edge` from 'reactflow':  Imports the `Edge` type from the `reactflow` library, representing a connection between two nodes in the workflow.
//   - `create` from 'zustand': Imports the `create` function from the `zustand` library, used to create a Zustand store.
//   - `persist` from 'zustand/middleware': Imports the `persist` middleware from `zustand/middleware`, used to persist the store's state to local storage.
//   - `createLogger` from '@/lib/logs/console/logger': Imports a custom logger function, presumably for logging events within the store.
//   - `BlockState` from '@/stores/workflows/workflow/types': Imports the `BlockState` type, representing the state of a block in the workflow.
//   - Types from './types': Imports several types related to undo/redo operations, including:
//     - `MoveBlockOperation`: Represents an operation that moves a block.
//     - `Operation`:  A generic type representing any operation.
//     - `OperationEntry`: Represents a single entry in the undo/redo stack, containing the operation and its inverse.
//     - `RemoveBlockOperation`: Represents an operation that removes a block.
//     - `RemoveEdgeOperation`: Represents an operation that removes an edge.
//     - `UndoRedoState`: Defines the structure of the undo/redo store's state.

// 2. Logger Instance:
//   - `const logger = createLogger('UndoRedoStore')`: Creates a logger instance specifically for this store, using the `createLogger` function.  This allows for filtering and identifying logs originating from this store.

// 3. Default Capacity:
//   - `const DEFAULT_CAPACITY = 15`: Defines the default maximum number of operations to store in the undo/redo stacks.

// 4. `getStackKey` Function:
//   - `function getStackKey(workflowId: string, userId: string): string { return `${workflowId}:${userId}` }`:
//     - This function generates a unique key for each workflow and user combination.
//     - It's used to store and retrieve the undo/redo stack for a specific workflow and user.  This ensures that each user has their own undo/redo history for each workflow they are working on.

// 5. `isOperationApplicable` Function:
//   - `function isOperationApplicable(...)`: This function determines whether an operation can still be applied to the current state of the workflow.
//   - `graph: { blocksById: Record<string, BlockState>; edgesById: Record<string, Edge> }`: Takes the current graph state as input, containing the blocks and edges by their IDs.
//   - The function uses a `switch` statement to check the operation type and determine its applicability:
//     - `'remove-block'`: Checks if the block being removed still exists in the graph.
//     - `'add-block'`: Checks if the block being added *does not* already exist in the graph.
//     - `'move-block'`: Checks if the block being moved still exists.
//     - `'update-parent'`: Checks if the block being updated still exists.
//     - `'duplicate-block'`: Checks if the duplicated block exists.
//     - `'remove-edge'`: Checks if the edge being removed still exists.
//     - `'add-edge'`: Checks if the edge being added *does not* already exist.
//     - `'add-subflow'` / `'remove-subflow'`: Checks if the subflow exists when removing it, and doesn't exist when adding it.
//     - `default`: Returns `true` for any other operation type, assuming it's always applicable.  This might need to be adjusted as more operation types are added.
//   - This function is crucial for ensuring that undo/redo operations don't cause errors when the workflow state has changed since the operation was initially performed.  For example, if a user deletes a block and then tries to redo an operation that moves that block, the operation should be skipped.

// 6. `useUndoRedoStore` Zustand Store:
//   - `export const useUndoRedoStore = create<UndoRedoState>()(...)`: Creates the Zustand store using the `create` function.
//   - `persist(...)`: Applies the `persist` middleware, which automatically saves the store's state to local storage.  This ensures that the undo/redo history is preserved even when the user closes and reopens the browser.
//   - The store's state is defined by the `UndoRedoState` type, which likely includes properties for:
//     - `stacks`: An object that stores the undo and redo stacks for each workflow and user combination. The key is obtained by the function `getStackKey`.
//     - `capacity`: The maximum capacity of the undo/redo stacks.
//   - The `persist` middleware is configured with:
//     - `name: 'workflow-undo-redo'`: The key used to store the state in local storage.
//     - `partialize`:  A function that specifies which parts of the state should be persisted.  In this case, it only persists the `stacks` and `capacity` properties.
//   - The store defines the following actions:

//     - `stacks: {}`: Initializes an empty object to store undo/redo stacks for different workflows and users.
//     - `capacity: DEFAULT_CAPACITY`: Sets the initial capacity of the undo/redo stacks to the default value (15).

//     - `push(workflowId: string, userId: string, entry: OperationEntry)`: Adds a new operation to the undo stack.
//       - Generates a unique key for the workflow and user.
//       - Retrieves the existing undo/redo stack for that key, or creates a new one if it doesn't exist.
//       - **Move Operation Coalescing:**
//         - Checks if the incoming operation is a `move-block` operation.
//         - If it is, it checks if the last operation in the undo stack is also a `move-block` operation for the *same* block.
//         - If both conditions are true, it *coalesces* the two operations into a single operation that represents the cumulative movement. This prevents the undo stack from being filled with many small move operations.
//         - **Skipping No-Op Moves:** The `push` function also includes a check to prevent no-op move operations (where the block's position or parent hasn't actually changed) from being added to the undo stack.
//       - Adds the new operation to the undo stack.
//       - If the undo stack exceeds its capacity, it removes the oldest entry.
//       - Resets the redo stack (since a new operation invalidates any previously redone operations).
//       - Logs the event using the logger.

//     - `undo(workflowId: string, userId: string)`: Undoes the last operation.
//       - Retrieves the undo/redo stack for the given workflow and user.
//       - If the undo stack is empty, it returns `null`.
//       - Removes the last operation from the undo stack.
//       - Adds the operation to the redo stack.
//       - If the redo stack exceeds its capacity, it removes the oldest entry.
//       - Returns the undone operation entry.
//       - Logs the event.

//     - `redo(workflowId: string, userId: string)`: Redoes the last undone operation.
//       - Retrieves the undo/redo stack.
//       - If the redo stack is empty, it returns `null`.
//       - Removes the last operation from the redo stack.
//       - Adds the operation to the undo stack.
//       - If the undo stack exceeds its capacity, it removes the oldest entry.
//       - Returns the redone operation entry.
//       - Logs the event.

//     - `clear(workflowId: string, userId: string)`: Clears both the undo and redo stacks for a specific workflow and user.
//       - Removes the stack from the `stacks` object.
//       - Logs the event.

//     - `clearRedo(workflowId: string, userId: string)`: Clears only the redo stack.

//     - `getStackSizes(workflowId: string, userId: string)`: Returns the sizes of the undo and redo stacks for a given workflow and user.

//     - `setCapacity(capacity: number)`: Changes the maximum capacity of the undo/redo stacks.
//       - Truncates the existing undo and redo stacks to the new capacity.
//       - Logs the event.

//     - `pruneInvalidEntries(workflowId: string, userId: string, graph: { blocksById: Record<string, BlockState>; edgesById: Record<string, Edge> })`:  Removes invalid entries from the undo and redo stacks.
//       - Filters the undo stack, keeping only the entries whose *inverse* operations are still applicable (using `isOperationApplicable`). The inverse is checked because the undo stack contains operations that have already been performed, and we want to make sure we can undo them correctly by applying the inverse.
//       - Filters the redo stack, keeping only the entries whose operations are still applicable.  The redo stack contains operations that have been undone, so we want to make sure we can redo them correctly by applying the original operation.
//       - Updates the store with the filtered stacks.
//       - Logs the event.

// Simplifications:
// 1.  **Readability**: The code is reasonably readable.  Variable names are descriptive.
// 2.  **Complexity**:  The most complex part is the `push` function, specifically the move operation coalescing logic.  This logic could be extracted into a separate helper function for improved readability.

// Overall:
// This code provides a robust and well-structured undo/redo implementation for a workflow editor. It utilizes Zustand and its middleware effectively for state management and persistence. The inclusion of move operation coalescing and invalid entry pruning enhances the user experience and ensures the integrity of the undo/redo history.  The use of logging helps in debugging and understanding the store's behavior.
```