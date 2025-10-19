```typescript
import type { Edge } from 'reactflow'
import type { BlockState } from '@/stores/workflows/workflow/types'

export type OperationType =
  | 'add-block'
  | 'remove-block'
  | 'add-edge'
  | 'remove-edge'
  | 'add-subflow'
  | 'remove-subflow'
  | 'move-block'
  | 'move-subflow'
  | 'duplicate-block'
  | 'update-parent'

export interface BaseOperation {
  id: string
  type: OperationType
  timestamp: number
  workflowId: string
  userId: string
}

export interface AddBlockOperation extends BaseOperation {
  type: 'add-block'
  data: {
    blockId: string
  }
}

export interface RemoveBlockOperation extends BaseOperation {
  type: 'remove-block'
  data: {
    blockId: string
    blockSnapshot: BlockState | null
    edgeSnapshots?: Edge[]
    allBlockSnapshots?: Record<string, BlockState>
  }
}

export interface AddEdgeOperation extends BaseOperation {
  type: 'add-edge'
  data: {
    edgeId: string
  }
}

export interface RemoveEdgeOperation extends BaseOperation {
  type: 'remove-edge'
  data: {
    edgeId: string
    edgeSnapshot: Edge | null
  }
}

export interface AddSubflowOperation extends BaseOperation {
  type: 'add-subflow'
  data: {
    subflowId: string
  }
}

export interface RemoveSubflowOperation extends BaseOperation {
  type: 'remove-subflow'
  data: {
    subflowId: string
    subflowSnapshot: BlockState | null
  }
}

export interface MoveBlockOperation extends BaseOperation {
  type: 'move-block'
  data: {
    blockId: string
    before: {
      x: number
      y: number
      parentId?: string
    }
    after: {
      x: number
      y: number
      parentId?: string
    }
  }
}

export interface MoveSubflowOperation extends BaseOperation {
  type: 'move-subflow'
  data: {
    subflowId: string
    before: {
      x: number
      y: number
    }
    after: {
      x: number
      y: number
    }
  }
}

export interface DuplicateBlockOperation extends BaseOperation {
  type: 'duplicate-block'
  data: {
    sourceBlockId: string
    duplicatedBlockId: string
    duplicatedBlockSnapshot: BlockState
    autoConnectEdge?: Edge
  }
}

export interface UpdateParentOperation extends BaseOperation {
  type: 'update-parent'
  data: {
    blockId: string
    oldParentId?: string
    newParentId?: string
    oldPosition: { x: number; y: number }
    newPosition: { x: number; y: number }
    affectedEdges?: Edge[]
  }
}

export type Operation =
  | AddBlockOperation
  | RemoveBlockOperation
  | AddEdgeOperation
  | RemoveEdgeOperation
  | AddSubflowOperation
  | RemoveSubflowOperation
  | MoveBlockOperation
  | MoveSubflowOperation
  | DuplicateBlockOperation
  | UpdateParentOperation

export interface OperationEntry {
  id: string
  operation: Operation
  inverse: Operation
  createdAt: number
}

export interface UndoRedoState {
  stacks: Record<
    string,
    {
      undo: OperationEntry[]
      redo: OperationEntry[]
    }
  >
  capacity: number
  push: (workflowId: string, userId: string, entry: OperationEntry) => void
  undo: (workflowId: string, userId: string) => OperationEntry | null
  redo: (workflowId: string, userId: string) => OperationEntry | null
  clear: (workflowId: string, userId: string) => void
  clearRedo: (workflowId: string, userId: string) => void
  getStackSizes: (workflowId: string, userId: string) => { undoSize: number; redoSize: number }
  setCapacity: (capacity: number) => void
  pruneInvalidEntries: (
    workflowId: string,
    userId: string,
    graph: { blocksById: Record<string, BlockState>; edgesById: Record<string, Edge> }
  ) => void
}
```

### Purpose of this file

This TypeScript file defines the data structures and types necessary for implementing an undo/redo functionality within a workflow editor application, likely built with ReactFlow. It specifies the types of operations that can be performed on the workflow (e.g., adding a block, removing an edge), the data associated with each operation, and the structure for managing the undo/redo stacks. It also provides a structure to define the methods available to control and manage the state of the undo/redo stacks.

### Explanation of Code

Let's break down each section of the code:

**1. Imports:**

```typescript
import type { Edge } from 'reactflow'
import type { BlockState } from '@/stores/workflows/workflow/types'
```

*   `import type { Edge } from 'reactflow'`: Imports the `Edge` type from the `reactflow` library.  `Edge` likely represents a connection between two nodes in the workflow diagram.  The `type` keyword indicates that this is a type-only import, meaning it won't import any actual values, only the type definition.
*   `import type { BlockState } from '@/stores/workflows/workflow/types'`: Imports the `BlockState` type from a local file.  `BlockState` presumably represents the state of a block (or node) within the workflow.  It could include properties like its position, configuration, or other relevant data.  The `@/stores/workflows/workflow/types` path suggests this type is defined within the application's state management system (e.g., Redux, Zustand, or a custom store).

**2. `OperationType` Type:**

```typescript
export type OperationType =
  | 'add-block'
  | 'remove-block'
  | 'add-edge'
  | 'remove-edge'
  | 'add-subflow'
  | 'remove-subflow'
  | 'move-block'
  | 'move-subflow'
  | 'duplicate-block'
  | 'update-parent'
```

*   `export type OperationType = ...`: Defines a type called `OperationType` using a union of string literals.  This type represents all the possible types of operations that can be performed within the workflow editor.
*   Each string literal (e.g., `'add-block'`, `'remove-edge'`) represents a specific operation, such as adding a new block, removing an existing edge, adding a subflow, moving a block and so on.

**3. `BaseOperation` Interface:**

```typescript
export interface BaseOperation {
  id: string
  type: OperationType
  timestamp: number
  workflowId: string
  userId: string
}
```

*   `export interface BaseOperation { ... }`: Defines an interface called `BaseOperation`.  This interface serves as the base type for all specific operation types.  It contains common properties that are shared by all operations.
*   `id: string`: A unique identifier for the operation.
*   `type: OperationType`: The type of operation (one of the values defined in the `OperationType` type).
*   `timestamp: number`: A timestamp indicating when the operation was performed.
*   `workflowId: string`:  The ID of the workflow the operation belongs to.
*   `userId: string`: The ID of the user who performed the operation.

**4. Specific Operation Interfaces (e.g., `AddBlockOperation`, `RemoveBlockOperation`):**

These interfaces define the structure of data for each specific `OperationType`.  They all extend the `BaseOperation` interface and add a `data` property that holds operation-specific information.

*   **`AddBlockOperation`:**

```typescript
export interface AddBlockOperation extends BaseOperation {
  type: 'add-block'
  data: {
    blockId: string
  }
}
```

    *   `type: 'add-block'`:  Specifies the operation type as `'add-block'`.  This is a type literal, ensuring that only the correct type is assigned.
    *   `data: { blockId: string }`:  Contains the ID of the block that was added.

*   **`RemoveBlockOperation`:**

```typescript
export interface RemoveBlockOperation extends BaseOperation {
  type: 'remove-block'
  data: {
    blockId: string
    blockSnapshot: BlockState | null
    edgeSnapshots?: Edge[]
    allBlockSnapshots?: Record<string, BlockState>
  }
}
```

    *   `type: 'remove-block'`: Specifies the operation type as `'remove-block'`.
    *   `data`: Contains information needed to potentially *undo* the removal of the block.
        *   `blockId: string`: The ID of the removed block.
        *   `blockSnapshot: BlockState | null`: A snapshot of the block's state *before* it was removed. This is crucial for restoring the block during an undo operation.  It can be `null` if the block's state couldn't be captured.
        *   `edgeSnapshots?: Edge[]`: Optional array of edges that were connected to the removed block.  These edges would also need to be restored during an undo. The `?` indicates that the property is optional.
        *  `allBlockSnapshots?: Record<string, BlockState>`: Optional record of all blocks snapshots.

*   **`AddEdgeOperation`:**

```typescript
export interface AddEdgeOperation extends BaseOperation {
  type: 'add-edge'
  data: {
    edgeId: string
  }
}
```

    *   `type: 'add-edge'`: Specifies the operation type as `'add-edge'`.
    *   `data: { edgeId: string }`: Contains the ID of the edge that was added.

*   **`RemoveEdgeOperation`:**

```typescript
export interface RemoveEdgeOperation extends BaseOperation {
  type: 'remove-edge'
  data: {
    edgeId: string
    edgeSnapshot: Edge | null
  }
}
```

    *   `type: 'remove-edge'`: Specifies the operation type as `'remove-edge'`.
    *   `data`: Contains information needed to potentially *undo* the removal of the edge.
        *   `edgeId: string`: The ID of the removed edge.
        *   `edgeSnapshot: Edge | null`: A snapshot of the edge's state *before* it was removed. This is crucial for restoring the edge during an undo operation. It can be `null` if the edge's state couldn't be captured.

*   **`AddSubflowOperation`:**

```typescript
export interface AddSubflowOperation extends BaseOperation {
  type: 'add-subflow'
  data: {
    subflowId: string
  }
}
```

    *   `type: 'add-subflow'`: Specifies the operation type as `'add-subflow'`.
    *   `data: { subflowId: string }`: Contains the ID of the subflow that was added.

*   **`RemoveSubflowOperation`:**

```typescript
export interface RemoveSubflowOperation extends BaseOperation {
  type: 'remove-subflow'
  data: {
    subflowId: string
    subflowSnapshot: BlockState | null
  }
}
```

    *   `type: 'remove-subflow'`: Specifies the operation type as `'remove-subflow'`.
    *   `data`: Contains information needed to potentially *undo* the removal of the subflow.
        *   `subflowId: string`: The ID of the removed subflow.
        *   `subflowSnapshot: BlockState | null`: A snapshot of the subflow's state *before* it was removed.  This is crucial for restoring the subflow during an undo operation. It can be `null` if the subflow's state couldn't be captured.

*   **`MoveBlockOperation`:**

```typescript
export interface MoveBlockOperation extends BaseOperation {
  type: 'move-block'
  data: {
    blockId: string
    before: {
      x: number
      y: number
      parentId?: string
    }
    after: {
      x: number
      y: number
      parentId?: string
    }
  }
}
```

    *   `type: 'move-block'`: Specifies the operation type as `'move-block'`.
    *   `data`: Contains information about the block's position *before* and *after* the move.
        *   `blockId: string`: The ID of the moved block.
        *   `before`: The block's position before the move.
            *   `x: number`: The x-coordinate.
            *   `y: number`: The y-coordinate.
            *   `parentId?: string`: The ID of the parent block before the move (if the block was nested within another block).
        *   `after`: The block's position after the move.
            *   `x: number`: The x-coordinate.
            *   `y: number`: The y-coordinate.
            *   `parentId?: string`: The ID of the parent block after the move (if the block is now nested within another block).

*   **`MoveSubflowOperation`:**

```typescript
export interface MoveSubflowOperation extends BaseOperation {
  type: 'move-subflow'
  data: {
    subflowId: string
    before: {
      x: number
      y: number
    }
    after: {
      x: number
      y: number
    }
  }
}
```

    *   `type: 'move-subflow'`: Specifies the operation type as `'move-subflow'`.
    *   `data`: Contains information about the subflow's position *before* and *after* the move.
        *   `subflowId: string`: The ID of the moved subflow.
        *   `before`: The subflow's position before the move.
            *   `x: number`: The x-coordinate.
            *   `y: number`: The y-coordinate.
        *   `after`: The subflow's position after the move.
            *   `x: number`: The x-coordinate.
            *   `y: number.
*   **`DuplicateBlockOperation`:**

```typescript
export interface DuplicateBlockOperation extends BaseOperation {
  type: 'duplicate-block'
  data: {
    sourceBlockId: string
    duplicatedBlockId: string
    duplicatedBlockSnapshot: BlockState
    autoConnectEdge?: Edge
  }
}
```

    *   `type: 'duplicate-block'`: Specifies the operation type as `'duplicate-block'`.
    *   `data`: Contains information about the duplicated block.
        *   `sourceBlockId: string`: The ID of the original block that was duplicated.
        *   `duplicatedBlockId: string`: The ID of the newly created, duplicated block.
        *   `duplicatedBlockSnapshot: BlockState`: A snapshot of the duplicated block's state.
        *   `autoConnectEdge?: Edge`: An optional edge that was automatically created to connect the duplicated block.

*   **`UpdateParentOperation`:**

```typescript
export interface UpdateParentOperation extends BaseOperation {
  type: 'update-parent'
  data: {
    blockId: string
    oldParentId?: string
    newParentId?: string
    oldPosition: { x: number; y: number }
    newPosition: { x: number; y: number }
    affectedEdges?: Edge[]
  }
}
```

    *   `type: 'update-parent'`: Specifies the operation type as `'update-parent'`.
    *   `data`: Contains information about the parent update.
        *   `blockId: string`: The ID of the block which parent was updated.
        *   `oldParentId?: string`: The ID of the previous parent.
        *   `newParentId?: string`: The ID of the new parent.
        *   `oldPosition: { x: number; y: number }`: The position of the block in the previous parent.
        *   `newPosition: { x: number; y: number }`: The position of the block in the new parent.
        *   `affectedEdges?: Edge[]`: An optional edge that was affected by the parent update.

**5. `Operation` Type:**

```typescript
export type Operation =
  | AddBlockOperation
  | RemoveBlockOperation
  | AddEdgeOperation
  | RemoveEdgeOperation
  | AddSubflowOperation
  | RemoveSubflowOperation
  | MoveBlockOperation
  | MoveSubflowOperation
  | DuplicateBlockOperation
  | UpdateParentOperation
```

*   `export type Operation = ...`: Defines a type called `Operation` as a union of all the specific operation interfaces defined previously. This means that a variable of type `Operation` can hold any of the specific operation types. This simplifies working with operations in a generic way.

**6. `OperationEntry` Interface:**

```typescript
export interface OperationEntry {
  id: string
  operation: Operation
  inverse: Operation
  createdAt: number
}
```

*   `export interface OperationEntry { ... }`: Defines an interface called `OperationEntry`.  An `OperationEntry` represents a single entry in the undo/redo stack.  It contains the original operation and its inverse, allowing for easy undoing and redoing.
*   `id: string`: A unique identifier for the operation entry.
*   `operation: Operation`: The operation that was performed.
*   `inverse: Operation`: The inverse operation that would undo the original operation.  For example, if the `operation` is `AddBlockOperation`, the `inverse` would be `RemoveBlockOperation`.
*   `createdAt: number`: A timestamp indicating when the operation entry was created.

**7. `UndoRedoState` Interface:**

```typescript
export interface UndoRedoState {
  stacks: Record<
    string,
    {
      undo: OperationEntry[]
      redo: OperationEntry[]
    }
  >
  capacity: number
  push: (workflowId: string, userId: string, entry: OperationEntry) => void
  undo: (workflowId: string, userId: string) => OperationEntry | null
  redo: (workflowId: string, userId: string) => OperationEntry | null
  clear: (workflowId: string, userId: string) => void
  clearRedo: (workflowId: string, userId: string) => void
  getStackSizes: (workflowId: string, userId: string) => { undoSize: number; redoSize: number }
  setCapacity: (capacity: number) => void
  pruneInvalidEntries: (
    workflowId: string,
    userId: string,
    graph: { blocksById: Record<string, BlockState>; edgesById: Record<string, Edge> }
  ) => void
}
```

*   `export interface UndoRedoState { ... }`: Defines the structure for managing the undo/redo state of the application.
*   `stacks: Record<string, { undo: OperationEntry[]; redo: OperationEntry[] }>`:
    *   This property stores the undo/redo stacks for different workflows, indexed by `workflowId`. The `Record<string, ...>` means that it is an object where keys are `string` (workflowId) and values are the stack state for that workflow.
    *   Each workflow has its own `undo` and `redo` stacks, which are arrays of `OperationEntry` objects.
*   `capacity: number`: The maximum number of operations that can be stored in the undo/redo stacks.  This helps to prevent memory leaks.
*   `push: (workflowId: string, userId: string, entry: OperationEntry) => void`:  A function to push a new `OperationEntry` onto the undo stack.  It takes the `workflowId`, `userId` and the `OperationEntry` to add.
*   `undo: (workflowId: string, userId: string) => OperationEntry | null`: A function to undo the last operation. It retrieves the last `OperationEntry` from the undo stack, performs the inverse operation, and pushes the original operation onto the redo stack. Returns the undone operation entry, or `null` if the undo stack is empty.
*   `redo: (workflowId: string, userId: string) => OperationEntry | null`: A function to redo the last undone operation. It retrieves the last `OperationEntry` from the redo stack, performs the original operation, and pushes the inverse operation onto the undo stack. Returns the redone operation entry, or `null` if the redo stack is empty.
*   `clear: (workflowId: string, userId: string) => void`: A function to clear both the undo and redo stacks.
*   `clearRedo: (workflowId: string, userId: string) => void`: A function to clear only the redo stack.
*   `getStackSizes: (workflowId: string, userId: string) => { undoSize: number; redoSize: number }`: A function that returns the current size of the undo and redo stacks.
*   `setCapacity: (capacity: number) => void`: A function to set the maximum capacity of the undo/redo stacks.
*   `pruneInvalidEntries: (workflowId: string, userId: string, graph: { blocksById: Record<string, BlockState>; edgesById: Record<string, Edge> }) => void`: A function to remove invalid entries from the undo/redo stacks.  This is important to handle cases where a block or edge has been permanently deleted and can no longer be restored.  It takes the current graph state (blocks and edges) as an argument.

### Summary

This file provides a robust type system for implementing undo/redo functionality in a workflow editor. It defines the structure of operations, their associated data, and the overall state management required to track and revert changes. The use of interfaces and type unions makes the code well-organized, maintainable, and type-safe. This approach ensures that all operations are properly tracked and that the undo/redo functionality behaves predictably.
