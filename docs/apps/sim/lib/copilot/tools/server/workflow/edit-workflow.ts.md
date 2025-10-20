This TypeScript file defines a core server-side tool called `edit_workflow` for modifying the state of a workflow. It's designed to accept a series of operations (like adding, editing, or deleting blocks) and apply them to a workflow's internal JSON representation. The goal is to provide a programmatic and robust way to update complex workflow structures, ensuring data integrity and handling dependencies correctly.

---

### Purpose of this File and What it Does

This file's primary purpose is to **manage and modify workflow definitions**. Workflows in this system are represented as a complex JSON object containing various "blocks" (nodes) and "edges" (connections between blocks). This `edit_workflow` server tool acts as an API endpoint or a service function that allows external systems (like a user interface or an AI agent) to make structured changes to a workflow.

Here's a breakdown of what it does:

1.  **Receives Operations:** It takes an array of `EditWorkflowOperation` objects, each describing a specific change to be made to the workflow (e.g., adding a new block, changing a block's parameters, or deleting a block).
2.  **Loads Workflow State:** It fetches the current state of the workflow. This can either be provided directly (useful for client-side previews or chained operations) or loaded from the database.
3.  **Applies Operations Systematically:** It processes these operations in a carefully determined order to avoid conflicts and ensure dependencies are met. For instance, a parent block must exist before a child block can be inserted into it.
4.  **Transforms and Normalizes Data:** It handles the conversion of incoming parameters into the internal format expected by the workflow system. This includes normalizing inputs, outputs, connections, and special block types like custom tools or response formats.
5.  **Maintains Workflow Structure:** It correctly updates the `blocks` and `edges` collections, and also regenerates specialized structures like `loops` and `parallels` that represent sub-workflows.
6.  **Validates Workflow State:** After applying changes, it validates the integrity and correctness of the modified workflow state.
7.  **Persists Custom Tools:** If the workflow contains custom tools, it extracts and saves their definitions to the database, making them available for future use.
8.  **Returns Updated State:** It returns the fully modified and validated workflow state, which can then be saved back to the database or used for further processing.

In essence, it's a transactional engine for workflow mutations, ensuring that changes are applied correctly and the resulting workflow remains valid.

---

### Simplified Complex Logic: `topologicalSortInserts` and Operation Order

The most complex piece of logic here is ensuring that workflow operations are applied in the correct order, especially when dealing with nested blocks (like blocks inside a "loop" or "parallel" block, which are called "subflows" in this context).

**The Problem:**
Imagine you want to add a "Loop" block and, within the same set of operations, add a "Generate Text" block *inside* that loop. If the "Generate Text" block tries to assign itself to the "Loop" block before the "Loop" block itself has been created, it will fail because its parent doesn't exist yet.

**The Solution: Operation Reordering & `topologicalSortInserts`**

This file solves this by:

1.  **Prioritizing Deletions:** First, it performs all `delete` operations. This cleans up any blocks that are no longer needed before other operations try to interact with them.
2.  **Handling Extractions:** Then, `extract_from_subflow` operations are processed. These essentially "un-parent" a block, making it a top-level block.
3.  **Adding Top-Level Blocks:** Next, `add` operations create new blocks that are not nested within any existing subflow. This ensures that potential parent blocks (like a new "Loop" block) are created before anything tries to insert into them.
4.  **Sorting and Inserting into Subflows (`topologicalSortInserts`):** This is where the core dependency logic comes in.
    *   It specifically focuses on `insert_into_subflow` operations, where a block is placed *inside* another (parent) block.
    *   It builds a "dependency graph": if block `A` is inserted into block `B`, then `A` depends on `B`.
    *   It uses a standard algorithm called **Kahn's Algorithm for Topological Sort**. This algorithm finds an order for processing items where no item is processed before its dependencies are met.
    *   **Simplified:** It finds all blocks that don't depend on any *other block currently being inserted*, processes them first, and then iteratively processes blocks whose dependencies are now met. This guarantees that parent blocks are handled before their children when both are part of the `insert_into_subflow` group.
5.  **Processing Edits:** Finally, `edit` operations are applied. These modify existing blocks and their connections, which can only be done reliably once all additions and insertions are complete.

By following this precise order, the system ensures that changes are applied logically and consistently, preventing errors related to missing dependencies or inconsistent states.

---

### Explanation of Each Line of Code (Deep Dive)

Let's go through the code section by section.

#### Imports

```typescript
import crypto from 'crypto' // Provides cryptographic functionality, used here for generating unique IDs (UUIDs).
import { db } from '@sim/db' // Imports the database connection instance from a local `@sim/db` module.
import { workflow as workflowTable } from '@sim/db/schema' // Imports the 'workflow' table schema definition from the database schema. 'workflow' is aliased to 'workflowTable' for clarity.
import { eq } from 'drizzle-orm' // Imports the 'eq' (equals) function from Drizzle ORM, used for database query comparisons.
import type { BaseServerTool } from '@/lib/copilot/tools/server/base-tool' // Imports the TypeScript type definition for a server tool, likely defining the structure of tools that can be executed on the server.
import { createLogger } from '@/lib/logs/console/logger' // Imports a utility to create a logger instance for structured logging.
import { getBlockOutputs } from '@/lib/workflows/block-outputs' // Imports a function to determine the output structure of a given block type.
import { extractAndPersistCustomTools } from '@/lib/workflows/custom-tools-persistence' // Imports a function to extract and save custom tool definitions found in a workflow to the database.
import { loadWorkflowFromNormalizedTables } from '@/lib/workflows/db-helpers' // Imports a helper function to load a workflow's full state from its normalized database tables.
import { validateWorkflowState } from '@/lib/workflows/validation' // Imports a function to validate the structural and logical integrity of a workflow's state.
import { getAllBlocks } from '@/blocks/registry' // Imports a function to get a list of all available block configurations from a central registry.
import { resolveOutputType } from '@/blocks/utils' // Imports a utility function to resolve the actual output type of a block based on its configuration.
import { generateLoopBlocks, generateParallelBlocks } from '@/stores/workflows/workflow/utils' // Imports functions to regenerate the derived 'loop' and 'parallel' structures within a workflow state based on its blocks.
```

#### Interfaces

```typescript
// Defines the structure for a single operation to modify the workflow.
interface EditWorkflowOperation {
  operation_type: 'add' | 'edit' | 'delete' | 'insert_into_subflow' | 'extract_from_subflow' // The type of modification: add a new block, edit an existing one, delete, insert into a subflow (like a loop), or extract from a subflow.
  block_id: string // The unique identifier of the block affected by this operation.
  params?: Record<string, any> // Optional parameters for the operation (e.g., new block type, input values, connections). 'Record<string, any>' means an object with string keys and any type of value.
}

// Defines the parameters expected by the 'edit_workflow' server tool.
interface EditWorkflowParams {
  operations: EditWorkflowOperation[] // An array of operations to apply to the workflow.
  workflowId: string // The ID of the workflow being edited.
  currentUserWorkflow?: string // Optional: A JSON string of the current workflow state. If provided, it bypasses fetching from the database.
}
```

#### `topologicalSortInserts` Function

This function sorts `insert_into_subflow` operations to ensure that parent blocks (e.g., a loop) are processed before their children, which are being inserted into them.

```typescript
/**
 * Topologically sort insert operations to ensure parents are created before children
 * Returns sorted array where parent inserts always come before child inserts
 */
function topologicalSortInserts(
  inserts: EditWorkflowOperation[], // An array of 'insert_into_subflow' operations.
  adds: EditWorkflowOperation[] // An array of 'add' operations (which create top-level blocks and can be parents).
): EditWorkflowOperation[] {
  if (inserts.length === 0) return [] // If no inserts, return an empty array.

  // Build a map of blockId -> operation for quick lookup
  const insertMap = new Map<string, EditWorkflowOperation>() // Map to quickly find an insert operation by its block_id.
  inserts.forEach((op) => insertMap.set(op.block_id, op)) // Populate the map.

  // Build a set of blocks being added (potential parents)
  const addedBlocks = new Set(adds.map((op) => op.block_id)) // Set of block_ids that are being added (these are processed earlier).

  // Build dependency graph: block -> blocks that depend on it
  const dependents = new Map<string, Set<string>>() // Maps a parent block_id to a set of child block_ids that depend on it.
  const dependencies = new Map<string, Set<string>>() // Maps a child block_id to a set of parent block_ids it depends on.

  inserts.forEach((op) => {
    const blockId = op.block_id // The ID of the block being inserted.
    const parentId = op.params?.subflowId // The ID of the subflow (parent) it's being inserted into.

    dependencies.set(blockId, new Set()) // Initialize dependencies for the current block.

    if (parentId) { // If there's a parent specified for this block.
      // Track dependency if parent is being inserted OR being added
      // This ensures children wait for parents regardless of operation type
      const parentBeingCreated = insertMap.has(parentId) || addedBlocks.has(parentId) // Check if the parent is part of the current insert batch or was part of the 'add' batch.

      if (parentBeingCreated) { // If the parent block is also being created (either added or inserted).
        // Only add dependency if parent is also being inserted (not added)
        // Because adds run before inserts, added parents are already created
        if (insertMap.has(parentId)) { // If the parent is also an 'insert' operation (not an 'add' which is handled separately).
          dependencies.get(blockId)!.add(parentId) // Add parentId to this block's dependencies.
          if (!dependents.has(parentId)) { // If parentId is not yet in dependents map, initialize its set.
            dependents.set(parentId, new Set())
          }
          dependents.get(parentId)!.add(blockId) // Add this blockId to the parent's dependents set.
        }
      }
    }
  })

  // Topological sort using Kahn's algorithm
  const sorted: EditWorkflowOperation[] = [] // Array to store the topologically sorted operations.
  const queue: string[] = [] // Queue for Kahn's algorithm, storing block_ids with no remaining dependencies.

  // Start with nodes that have no dependencies (or depend only on added blocks)
  inserts.forEach((op) => {
    const deps = dependencies.get(op.block_id)! // Get the dependencies for the current insert operation.
    if (deps.size === 0) { // If a block has no dependencies on other *insert* operations.
      queue.push(op.block_id) // Add it to the queue to be processed first.
    }
  })

  while (queue.length > 0) { // While there are blocks in the queue.
    const blockId = queue.shift()! // Dequeue a block_id.
    const op = insertMap.get(blockId) // Get its corresponding operation.
    if (op) {
      sorted.push(op) // Add the operation to the sorted list.
    }

    // Remove this node from dependencies of others
    const children = dependents.get(blockId) // Get all blocks that depend on the current block.
    if (children) {
      children.forEach((childId) => {
        const childDeps = dependencies.get(childId)! // Get dependencies of the child.
        childDeps.delete(blockId) // Remove the current block from the child's dependencies.
        if (childDeps.size === 0) { // If the child now has no remaining dependencies.
          queue.push(childId) // Add the child to the queue.
        }
      })
    }
  }

  // If sorted length doesn't match input, there's a cycle (shouldn't happen with valid operations)
  // Just append remaining operations - this is a fallback for malformed input or edge cases.
  if (sorted.length < inserts.length) {
    inserts.forEach((op) => {
      if (!sorted.includes(op)) { // If an operation was not added to the sorted list.
        sorted.push(op) // Add it at the end.
      }
    })
  }

  return sorted // Return the topologically sorted array of insert operations.
}
```

#### `createBlockFromParams` Function

This helper function generates a new block object (in the workflow's internal state format) based on the provided parameters.

```typescript
/**
 * Helper to create a block state from operation params
 */
function createBlockFromParams(blockId: string, params: any, parentId?: string): any {
  // Find the block's configuration from the registry based on its type.
  const blockConfig = getAllBlocks().find((b) => b.type === params.type)

  // Determine outputs based on trigger mode
  const triggerMode = params.triggerMode || false // Determine if the block is in trigger mode.
  let outputs: Record<string, any> // Variable to hold the block's outputs.

  if (params.outputs) { // If outputs are explicitly provided in params.
    outputs = params.outputs // Use them directly.
  } else if (blockConfig) { // If there's a block configuration.
    const subBlocks: Record<string, any> = {} // Initialize subBlocks for output resolution.
    if (params.inputs) { // If inputs are provided.
      Object.entries(params.inputs).forEach(([key, value]) => {
        subBlocks[key] = { id: key, type: 'short-input', value: value } // Convert inputs to temporary subBlock-like structure.
      })
    }
    // Resolve outputs: if in trigger mode, use getBlockOutputs; otherwise, use resolveOutputType from blockConfig.
    outputs = triggerMode
      ? getBlockOutputs(params.type, subBlocks, triggerMode)
      : resolveOutputType(blockConfig.outputs)
  } else {
    outputs = {} // Default to empty outputs if no config or explicit outputs.
  }

  // Construct the initial block state object.
  const blockState: any = {
    id: blockId, // The unique ID of the block.
    type: params.type, // The type of the block (e.g., 'text-generator', 'loop').
    name: params.name, // The display name of the block.
    position: { x: 0, y: 0 }, // Default position (might be updated later by client).
    enabled: params.enabled !== undefined ? params.enabled : true, // Block enabled state, defaults to true.
    horizontalHandles: true, // UI property.
    isWide: false, // UI property.
    advancedMode: params.advancedMode || false, // Advanced mode flag.
    height: 0, // UI property.
    triggerMode: triggerMode, // Whether the block is in trigger mode.
    subBlocks: {}, // Container for sub-blocks (e.g., input fields, configuration options).
    outputs: outputs, // The resolved outputs of the block.
    data: parentId ? { parentId, extent: 'parent' as const } : {}, // Data object; includes parentId and extent if it's a child block.
  }

  // Add inputs as subBlocks
  if (params.inputs) { // If inputs are provided in the operation params.
    Object.entries(params.inputs).forEach(([key, value]) => {
      let sanitizedValue = value // Start with the raw value.

      // Special handling for inputFormat - ensure it's an array
      if (key === 'inputFormat' && value !== null && value !== undefined) {
        if (!Array.isArray(value)) { // If inputFormat is not an array.
          // Invalid format, default to empty array
          sanitizedValue = [] // Set to an empty array.
        }
      }

      // Special handling for tools - normalize to restore sanitized fields
      if (key === 'tools' && Array.isArray(value)) {
        sanitizedValue = normalizeTools(value) // Normalize tools data.
      }

      // Special handling for responseFormat - normalize to ensure consistent format
      if (key === 'responseFormat' && value) {
        sanitizedValue = normalizeResponseFormat(value) // Normalize response format data.
      }

      // Add the input as a subBlock.
      blockState.subBlocks[key] = {
        id: key,
        type: 'short-input', // Type of the sub-block.
        value: sanitizedValue, // The (sanitized) value of the input.
      }
    })
  }

  // Set up subBlocks from block configuration (default values/structure)
  if (blockConfig) { // If a block configuration exists.
    blockConfig.subBlocks.forEach((subBlock) => {
      if (!blockState.subBlocks[subBlock.id]) { // If a subBlock from config is not already defined from params.inputs.
        blockState.subBlocks[subBlock.id] = { // Add it with its default structure.
          id: subBlock.id,
          type: subBlock.type,
          value: null, // Default value is null.
        }
      }
    })
  }

  return blockState // Return the constructed block state.
}
```

#### `normalizeTools` Function

This function ensures custom tool data is consistently formatted, especially after potential sanitization for other purposes (like AI model training).

```typescript
/**
 * Normalize tools array by adding back fields that were sanitized for training
 */
function normalizeTools(tools: any[]): any[] {
  return tools.map((tool) => { // Iterate over each tool in the array.
    if (tool.type === 'custom-tool') { // If it's a custom tool.
      // Reconstruct sanitized custom tool fields
      const normalized: any = {
        ...tool, // Copy existing tool properties.
        params: tool.params || {}, // Ensure 'params' exists, defaulting to an empty object.
        isExpanded: tool.isExpanded ?? true, // Ensure 'isExpanded' exists, defaulting to true.
      }

      // Ensure schema has proper structure
      if (normalized.schema?.function) { // If a function schema exists.
        normalized.schema = { // Reformat the schema.
          type: 'function',
          function: {
            name: tool.title, // Derive function name from the tool's title.
            description: normalized.schema.function.description,
            parameters: normalized.schema.function.parameters,
          },
        }
      }

      return normalized // Return the normalized custom tool.
    }

    // For other tool types, just ensure isExpanded exists
    return {
      ...tool, // Copy existing tool properties.
      isExpanded: tool.isExpanded ?? true, // Ensure 'isExpanded' exists, defaulting to true.
    }
  })
}
```

#### `normalizeResponseFormat` Function

This function standardizes the format of `responseFormat` values, typically into a pretty-printed JSON string.

```typescript
/**
 * Normalize responseFormat to ensure consistent storage
 * Handles both string (JSON) and object formats
 * Returns pretty-printed JSON for better UI readability
 */
function normalizeResponseFormat(value: any): string {
  try {
    let obj = value // Start with the raw value.

    // If it's already a string, parse it first
    if (typeof value === 'string') {
      const trimmed = value.trim() // Trim whitespace.
      if (!trimmed) { // If string is empty after trimming.
        return '' // Return empty string.
      }
      obj = JSON.parse(trimmed) // Attempt to parse the string as JSON.
    }

    // If it's an object, stringify it with consistent formatting
    if (obj && typeof obj === 'object') {
      // Sort keys recursively for consistent comparison
      const sortKeys = (item: any): any => { // Helper function to recursively sort keys of objects.
        if (Array.isArray(item)) {
          return item.map(sortKeys) // If array, sort each item.
        }
        if (item !== null && typeof item === 'object') {
          return Object.keys(item) // Get keys.
            .sort() // Sort keys alphabetically.
            .reduce((result: any, key: string) => {
              result[key] = sortKeys(item[key]) // Recursively sort values for each key.
              return result
            }, {}) // Build new object with sorted keys.
        }
        return item // Return non-object/array items as-is.
      }

      // Return pretty-printed with 2-space indentation for UI readability
      // The sanitizer will normalize it to minified format for comparison
      return JSON.stringify(sortKeys(obj), null, 2) // Stringify the sorted object with 2-space indentation.
    }

    return String(value) // If not an object or valid JSON string, convert to string.
  } catch (error) {
    // If parsing fails, return the original value as string
    return String(value) // In case of any error during parsing/stringifying, return original value as string.
  }
}
```

#### `addConnectionsAsEdges` Function

This helper function converts conceptual `connections` (defined in operation parameters) into concrete `edges` for the workflow graph.

```typescript
/**
 * Helper to add connections as edges for a block
 */
function addConnectionsAsEdges(
  modifiedState: any, // The current modified workflow state.
  blockId: string, // The ID of the block from which connections originate.
  connections: Record<string, any> // An object mapping source handles (e.g., 'success') to target block IDs.
): void {
  Object.entries(connections).forEach(([sourceHandle, targets]) => { // Iterate over each connection type and its targets.
    const targetArray = Array.isArray(targets) ? targets : [targets] // Ensure targets is an array.
    targetArray.forEach((targetId: string) => { // For each target block ID.
      modifiedState.edges.push({ // Add a new edge to the workflow state.
        id: crypto.randomUUID(), // Generate a unique ID for the edge.
        source: blockId, // The source block of the edge.
        sourceHandle, // The handle on the source block (e.g., 'source', 'error').
        target: targetId, // The target block of the edge.
        targetHandle: 'target', // Default handle on the target block.
        type: 'default', // Default edge type.
      })
    })
  })
}
```

#### `applyOperationsToWorkflowState` Function

This is the central function where all workflow modification logic resides.

```typescript
/**
 * Apply operations directly to the workflow JSON state
 */
function applyOperationsToWorkflowState(
  workflowState: any, // The initial workflow state.
  operations: EditWorkflowOperation[] // The list of operations to apply.
): any {
  // Deep clone the workflow state to avoid mutations on the original object.
  const modifiedState = JSON.parse(JSON.stringify(workflowState))

  // Log initial state for debugging and auditing.
  const logger = createLogger('EditWorkflowServerTool') // Create a logger instance.
  logger.info('Applying operations to workflow:', {
    totalOperations: operations.length, // Total number of operations.
    // Count operations by type.
    operationTypes: operations.reduce((acc: any, op) => {
      acc[op.operation_type] = (acc[op.operation_type] || 0) + 1
      return acc
    }, {}),
    initialBlockCount: Object.keys(modifiedState.blocks || {}).length, // Number of blocks before operations.
  })

  // Reorder operations: delete -> extract -> add -> insert -> edit
  // This order is crucial for correct application of changes.
  const deletes = operations.filter((op) => op.operation_type === 'delete')
  const extracts = operations.filter((op) => op.operation_type === 'extract_from_subflow')
  const adds = operations.filter((op) => op.operation_type === 'add')
  const inserts = operations.filter((op) => op.operation_type === 'insert_into_subflow')
  const edits = operations.filter((op) => op.operation_type === 'edit')

  // Sort insert operations to ensure parents are inserted before children.
  // This handles cases where a loop/parallel is being added along with its children.
  const sortedInserts = topologicalSortInserts(inserts, adds) // Call the topological sort function.

  // Combine operations into the final processing order.
  const orderedOperations: EditWorkflowOperation[] = [
    ...deletes,
    ...extracts,
    ...adds,
    ...sortedInserts,
    ...edits,
  ]

  logger.info('Operations after reordering:', {
    // Log the order of operations for debugging.
    order: orderedOperations.map(
      (op) =>
        `${op.operation_type}:${op.block_id}${op.params?.subflowId ? `(parent:${op.params.subflowId})` : ''}`
    ),
  })

  // Iterate and apply each ordered operation.
  for (const operation of orderedOperations) {
    const { operation_type, block_id, params } = operation // Destructure operation properties.

    logger.debug(`Executing operation: ${operation_type} for block ${block_id}`, {
      params: params ? Object.keys(params) : [], // Log keys of params, not full object for brevity.
      currentBlockCount: Object.keys(modifiedState.blocks).length, // Log current block count.
    })

    switch (operation_type) {
      case 'delete': {
        if (modifiedState.blocks[block_id]) { // If the block exists.
          // Find all child blocks to remove (recursively, for nested subflows).
          const blocksToRemove = new Set<string>([block_id]) // Start with the block itself.
          const findChildren = (parentId: string) => { // Recursive helper to find all descendants.
            Object.entries(modifiedState.blocks).forEach(([childId, child]: [string, any]) => {
              if (child.data?.parentId === parentId) { // If a block is a child of parentId.
                blocksToRemove.add(childId) // Add it to the set.
                findChildren(childId) // Recursively find its children.
              }
            })
          }
          findChildren(block_id) // Start finding children from the target block.

          // Remove blocks from the state.
          blocksToRemove.forEach((id) => delete modifiedState.blocks[id])

          // Remove edges connected to deleted blocks.
          modifiedState.edges = modifiedState.edges.filter(
            (edge: any) => !blocksToRemove.has(edge.source) && !blocksToRemove.has(edge.target) // Keep only edges whose source and target are not among the deleted blocks.
          )
        }
        break
      }

      case 'edit': {
        if (modifiedState.blocks[block_id]) { // If the block to edit exists.
          const block = modifiedState.blocks[block_id] // Get a reference to the block.

          // Ensure block has essential properties before editing.
          if (!block.type) {
            logger.warn(`Block ${block_id} missing type property, skipping edit`, {
              blockKeys: Object.keys(block),
              blockData: JSON.stringify(block),
            })
            break // Skip this edit if block type is missing (critical for determining behavior).
          }

          // Update inputs (convert to subBlocks format).
          if (params?.inputs) {
            if (!block.subBlocks) block.subBlocks = {} // Initialize subBlocks if not present.
            Object.entries(params.inputs).forEach(([key, value]) => {
              let sanitizedValue = value // Start with raw value.

              // Special handling for inputFormat - ensure it's an array.
              if (key === 'inputFormat' && value !== null && value !== undefined) {
                if (!Array.isArray(value)) {
                  sanitizedValue = []
                }
              }

              // Special handling for tools - normalize.
              if (key === 'tools' && Array.isArray(value)) {
                sanitizedValue = normalizeTools(value)
              }

              // Special handling for responseFormat - normalize.
              if (key === 'responseFormat' && value) {
                sanitizedValue = normalizeResponseFormat(value)
              }

              // Update or create the subBlock.
              if (!block.subBlocks[key]) {
                block.subBlocks[key] = {
                  id: key,
                  type: 'short-input',
                  value: sanitizedValue,
                }
              } else {
                block.subBlocks[key].value = sanitizedValue
              }
            })

            // Update loop/parallel configuration in block.data.
            if (block.type === 'loop') {
              block.data = block.data || {} // Ensure data object exists.
              if (params.inputs.loopType !== undefined) block.data.loopType = params.inputs.loopType
              if (params.inputs.iterations !== undefined) block.data.count = params.inputs.iterations
              if (params.inputs.collection !== undefined) block.data.collection = params.inputs.collection
            } else if (block.type === 'parallel') {
              block.data = block.data || {} // Ensure data object exists.
              if (params.inputs.parallelType !== undefined) block.data.parallelType = params.inputs.parallelType
              if (params.inputs.count !== undefined) block.data.count = params.inputs.count
              if (params.inputs.collection !== undefined) block.data.collection = params.inputs.collection
            }
          }

          // Update basic properties.
          if (params?.type !== undefined) block.type = params.type
          if (params?.name !== undefined) block.name = params.name

          // Handle trigger mode toggle.
          if (typeof params?.triggerMode === 'boolean') {
            block.triggerMode = params.triggerMode

            if (params.triggerMode === true) {
              // Remove all incoming edges when enabling trigger mode (as trigger blocks don't typically have incoming connections).
              modifiedState.edges = modifiedState.edges.filter(
                (edge: any) => edge.target !== block_id
              )
            }
          }

          // Handle advanced mode toggle.
          if (typeof params?.advancedMode === 'boolean') {
            block.advancedMode = params.advancedMode
          }

          // Handle nested nodes update (for loops/parallels).
          if (params?.nestedNodes) {
            // Remove all existing child blocks for this parent.
            const existingChildren = Object.keys(modifiedState.blocks).filter(
              (id) => modifiedState.blocks[id].data?.parentId === block_id
            )
            existingChildren.forEach((childId) => delete modifiedState.blocks[childId])

            // Remove edges to/from removed children.
            modifiedState.edges = modifiedState.edges.filter(
              (edge: any) =>
                !existingChildren.includes(edge.source) && !existingChildren.includes(edge.target)
            )

            // Add new nested blocks (children).
            Object.entries(params.nestedNodes).forEach(([childId, childBlock]: [string, any]) => {
              const childBlockState = createBlockFromParams(childId, childBlock, block_id) // Create child block.
              modifiedState.blocks[childId] = childBlockState // Add to state.

              // Add connections for the new child block.
              if (childBlock.connections) {
                addConnectionsAsEdges(modifiedState, childId, childBlock.connections)
              }
            })

            // Update loop/parallel configuration based on type (redundant with block.data update above, but ensures consistency).
            if (block.type === 'loop') {
              block.data = block.data || {}
              if (params.inputs?.loopType) block.data.loopType = params.inputs.loopType
              if (params.inputs?.iterations) block.data.count = params.inputs.iterations
              if (params.inputs?.collection) block.data.collection = params.inputs.collection
            } else if (block.type === 'parallel') {
              block.data = block.data || {}
              if (params.inputs?.parallelType) block.data.parallelType = params.inputs.parallelType
              if (params.inputs?.count) block.data.count = params.inputs.count
              if (params.inputs?.collection) block.data.collection = params.inputs.collection
            }
          }

          // Handle connections update (convert to edges).
          if (params?.connections) {
            // Remove existing outgoing edges from this block.
            modifiedState.edges = modifiedState.edges.filter(
              (edge: any) => edge.source !== block_id
            )

            // Add new edges based on connections.
            Object.entries(params.connections).forEach(([connectionType, targets]) => {
              if (targets === null) return // Skip if targets are null.

              // Map semantic connection names to actual React Flow handle IDs.
              const mapConnectionTypeToHandle = (type: string): string => {
                if (type === 'success') return 'source' // 'success' maps to 'source' handle.
                if (type === 'error') return 'error'   // 'error' maps to 'error' handle.
                return type // Other types pass through as-is.
              }

              const actualSourceHandle = mapConnectionTypeToHandle(connectionType) // Get the actual handle name.

              const addEdge = (targetBlock: string, targetHandle?: string) => { // Helper to add an individual edge.
                modifiedState.edges.push({
                  id: crypto.randomUUID(),
                  source: block_id,
                  sourceHandle: actualSourceHandle,
                  target: targetBlock,
                  targetHandle: targetHandle || 'target', // Default target handle.
                  type: 'default',
                })
              }

              // Handle different formats of targets (string, array, object).
              if (typeof targets === 'string') {
                addEdge(targets)
              } else if (Array.isArray(targets)) {
                targets.forEach((target: any) => {
                  if (typeof target === 'string') {
                    addEdge(target)
                  } else if (target?.block) { // If target is an object with a 'block' property.
                    addEdge(target.block, target.handle)
                  }
                })
              } else if (typeof targets === 'object' && (targets as any)?.block) { // If target is a single object with a 'block' property.
                addEdge((targets as any).block, (targets as any).handle)
              }
            })
          }

          // Handle specific edge removal.
          if (params?.removeEdges && Array.isArray(params.removeEdges)) {
            params.removeEdges.forEach(({ targetBlockId, sourceHandle = 'source' }) => {
              modifiedState.edges = modifiedState.edges.filter(
                (edge: any) =>
                  !( // Remove edge if it matches source, target, and sourceHandle.
                    edge.source === block_id &&
                    edge.target === targetBlockId &&
                    edge.sourceHandle === sourceHandle
                  )
              )
            })
          }
        }
        break
      }

      case 'add': {
        if (params?.type && params?.name) { // Ensure required params are present.
          // Create new block with proper structure.
          const newBlock = createBlockFromParams(block_id, params)

          // Set loop/parallel data on parent block BEFORE adding to blocks.
          if (params.nestedNodes) { // If this block is a loop/parallel with nested nodes.
            if (params.type === 'loop') {
              newBlock.data = {
                ...newBlock.data,
                loopType: params.inputs?.loopType || 'for', // Default loopType.
                ...(params.inputs?.collection && { collection: params.inputs.collection }), // Add collection if present.
                ...(params.inputs?.iterations && { count: params.inputs.iterations }), // Add iterations/count if present.
              }
            } else if (params.type === 'parallel') {
              newBlock.data = {
                ...newBlock.data,
                parallelType: params.inputs?.parallelType || 'count', // Default parallelType.
                ...(params.inputs?.collection && { collection: params.inputs.collection }),
                ...(params.inputs?.count && { count: params.inputs.count }),
              }
            }
          }

          // Add parent block FIRST before adding children.
          // This ensures children can reference valid parentId.
          modifiedState.blocks[block_id] = newBlock // Add the new block to the state.

          // Handle nested nodes (for loops/parallels created from scratch).
          if (params.nestedNodes) {
            Object.entries(params.nestedNodes).forEach(([childId, childBlock]: [string, any]) => {
              const childBlockState = createBlockFromParams(childId, childBlock, block_id) // Create child block, linking to parent.
              modifiedState.blocks[childId] = childBlockState // Add child to state.

              if (childBlock.connections) { // Add connections for the child.
                addConnectionsAsEdges(modifiedState, childId, childBlock.connections)
              }
            })
          }

          // Add connections as edges for the new parent block.
          if (params.connections) {
            addConnectionsAsEdges(modifiedState, block_id, params.connections)
          }
        }
        break
      }

      case 'insert_into_subflow': {
        const subflowId = params?.subflowId // ID of the parent subflow.
        if (!subflowId || !params?.type || !params?.name) {
          logger.error('Missing required params for insert_into_subflow', { block_id, params })
          break // Skip if required parameters are missing.
        }

        const subflowBlock = modifiedState.blocks[subflowId] // Get the parent subflow block.
        if (!subflowBlock) {
          logger.error('Subflow block not found - parent must be created first', {
            subflowId,
            block_id,
            existingBlocks: Object.keys(modifiedState.blocks),
            operationType: 'insert_into_subflow',
          })
          // This indicates an ordering issue in operations or invalid input.
          break // Skip this operation if parent doesn't exist.
        }

        if (subflowBlock.type !== 'loop' && subflowBlock.type !== 'parallel') {
          logger.error('Subflow block has invalid type', {
            subflowId,
            type: subflowBlock.type,
            block_id,
          })
          break // Subflow must be a 'loop' or 'parallel' type.
        }

        const blockConfig = getAllBlocks().find((block) => block.type === params.type) // Get block config.

        const existingBlock = modifiedState.blocks[block_id] // Check if the block already exists (meaning it's being moved).

        if (existingBlock) {
          // Moving existing block into subflow - just update parent.
          existingBlock.data = {
            ...existingBlock.data,
            parentId: subflowId, // Set new parentId.
            extent: 'parent' as const, // Set extent to 'parent'.
          }

          // Update inputs if provided (with normalization).
          if (params.inputs) {
            Object.entries(params.inputs).forEach(([key, value]) => {
              let sanitizedValue = value
              if (key === 'inputFormat' && value !== null && value !== undefined) {
                if (!Array.isArray(value)) {
                  sanitizedValue = []
                }
              }
              if (key === 'tools' && Array.isArray(value)) {
                sanitizedValue = normalizeTools(value)
              }
              if (key === 'responseFormat' && value) {
                sanitizedValue = normalizeResponseFormat(value)
              }

              if (!existingBlock.subBlocks[key]) {
                existingBlock.subBlocks[key] = {
                  id: key,
                  type: 'short-input',
                  value: sanitizedValue,
                }
              } else {
                existingBlock.subBlocks[key].value = sanitizedValue
              }
            })
          }
        } else {
          // Create new block as child of subflow.
          const newBlock = createBlockFromParams(block_id, params, subflowId) // Create new block with parentId.
          modifiedState.blocks[block_id] = newBlock // Add to state.
        }

        // Add/update connections as edges.
        if (params.connections) {
          // Remove existing outgoing edges from this block.
          modifiedState.edges = modifiedState.edges.filter((edge: any) => edge.source !== block_id)
          // Add new connections.
          addConnectionsAsEdges(modifiedState, block_id, params.connections)
        }
        break
      }

      case 'extract_from_subflow': {
        const subflowId = params?.subflowId // ID of the parent subflow it's being extracted from.
        if (!subflowId) {
          logger.warn('Missing subflowId for extract_from_subflow', { block_id })
          break // Skip if subflowId is missing.
        }

        const block = modifiedState.blocks[block_id] // Get the block to extract.
        if (!block) {
          logger.warn('Block not found for extraction', { block_id })
          break // Skip if block not found.
        }

        // Verify it's actually a child of this subflow (optional check for robustness).
        if (block.data?.parentId !== subflowId) {
          logger.warn('Block is not a child of specified subflow', {
            block_id,
            actualParent: block.data?.parentId,
            specifiedParent: subflowId,
          })
        }

        // Remove parent relationship.
        if (block.data) {
          block.data.parentId = undefined // Clear parentId.
          block.data.extent = undefined // Clear extent.
        }

        // Note: We keep the block and its edges, just remove parent relationship.
        // The block becomes a root-level block.
        break
      }
    }
  }

  // Regenerate loops and parallels after all modifications are applied.
  modifiedState.loops = generateLoopBlocks(modifiedState.blocks)
  modifiedState.parallels = generateParallelBlocks(modifiedState.blocks)

  // Validate all blocks have types before returning. This is a final cleanup/validation pass.
  const blocksWithoutType = Object.entries(modifiedState.blocks)
    .filter(([_, block]: [string, any]) => !block.type || block.type === undefined) // Find blocks without a type.
    .map(([id, block]: [string, any]) => ({ id, block }))

  if (blocksWithoutType.length > 0) {
    logger.error('Blocks without type after operations:', {
      blocksWithoutType: blocksWithoutType.map(({ id, block }) => ({
        id,
        type: block.type,
        name: block.name,
        keys: Object.keys(block),
      })),
    })

    // Attempt to fix by removing type-less blocks.
    blocksWithoutType.forEach(({ id }) => {
      delete modifiedState.blocks[id]
    })

    // Remove edges connected to removed blocks.
    const removedIds = new Set(blocksWithoutType.map(({ id }) => id))
    modifiedState.edges = modifiedState.edges.filter(
      (edge: any) => !removedIds.has(edge.source) && !removedIds.has(edge.target)
    )
  }

  return modifiedState // Return the final modified workflow state.
}
```

#### `getCurrentWorkflowStateFromDb` Function

This function retrieves a workflow's state from the database.

```typescript
async function getCurrentWorkflowStateFromDb(
  workflowId: string
): Promise<{ workflowState: any; subBlockValues: Record<string, Record<string, any>> }> {
  const logger = createLogger('EditWorkflowServerTool') // Create a logger instance.
  // Query the database for the workflow record.
  const [workflowRecord] = await db
    .select()
    .from(workflowTable)
    .where(eq(workflowTable.id, workflowId))
    .limit(1)
  if (!workflowRecord) throw new Error(`Workflow ${workflowId} not found in database`) // Throw if not found.

  // Load the full workflow state (blocks, edges, loops, parallels) from normalized tables.
  const normalized = await loadWorkflowFromNormalizedTables(workflowId)
  if (!normalized) throw new Error('Workflow has no normalized data') // Throw if no normalized data.

  // Validate and fix blocks without types (defense against corrupted data from DB).
  const blocks = { ...normalized.blocks } // Create a mutable copy of blocks.
  const invalidBlocks: string[] = [] // Array to store IDs of invalid blocks.

  Object.entries(blocks).forEach(([id, block]: [string, any]) => {
    if (!block.type) { // If a block is missing its type.
      logger.warn(`Block ${id} loaded without type from database`, {
        blockKeys: Object.keys(block),
        blockName: block.name,
      })
      invalidBlocks.push(id) // Mark as invalid.
    }
  })

  // Remove invalid blocks from the 'blocks' object.
  invalidBlocks.forEach((id) => delete blocks[id])

  // Remove edges connected to invalid blocks.
  const edges = normalized.edges.filter(
    (edge: any) => !invalidBlocks.includes(edge.source) && !invalidBlocks.includes(edge.target)
  )

  // Construct the workflow state object.
  const workflowState: any = {
    blocks,
    edges,
    loops: normalized.loops || {}, // Ensure loops object exists.
    parallels: normalized.parallels || {}, // Ensure parallels object exists.
  }
  // Extract subBlock values for potentially separate usage (e.g., in forms).
  const subBlockValues: Record<string, Record<string, any>> = {}
  Object.entries(normalized.blocks).forEach(([blockId, block]) => {
    subBlockValues[blockId] = {}
    Object.entries((block as any).subBlocks || {}).forEach(([subId, sub]) => {
      if ((sub as any).value !== undefined) subBlockValues[blockId][subId] = (sub as any).value
    })
  })
  return { workflowState, subBlockValues } // Return the state and subBlock values.
}
```

#### `editWorkflowServerTool` Export

This is the main export of the file, defining the server tool itself.

```typescript
export const editWorkflowServerTool: BaseServerTool<EditWorkflowParams, any> = {
  name: 'edit_workflow', // The unique name of this server tool.

  // The asynchronous execution function for the tool.
  async execute(params: EditWorkflowParams, context?: { userId: string }): Promise<any> {
    const logger = createLogger('EditWorkflowServerTool') // Create a logger instance.
    const { operations, workflowId, currentUserWorkflow } = params // Destructure input parameters.
    if (!operations || operations.length === 0) throw new Error('operations are required') // Validate operations.
    if (!workflowId) throw new Error('workflowId is required') // Validate workflowId.

    logger.info('Executing edit_workflow', { // Log execution start.
      operationCount: operations.length,
      workflowId,
      hasCurrentUserWorkflow: !!currentUserWorkflow, // Indicate if current workflow state was provided.
    })

    // Get current workflow state.
    let workflowState: any
    if (currentUserWorkflow) { // If workflow state is provided as a JSON string.
      try {
        workflowState = JSON.parse(currentUserWorkflow) // Parse it.
      } catch (error) {
        logger.error('Failed to parse currentUserWorkflow', error)
        throw new Error('Invalid currentUserWorkflow format') // Error if parsing fails.
      }
    } else { // Otherwise, load from the database.
      const fromDb = await getCurrentWorkflowStateFromDb(workflowId)
      workflowState = fromDb.workflowState
    }

    // Apply operations directly to the workflow state.
    const modifiedWorkflowState = applyOperationsToWorkflowState(workflowState, operations)

    // Validate the workflow state after modifications.
    const validation = validateWorkflowState(modifiedWorkflowState, { sanitize: true })

    if (!validation.valid) { // If validation fails.
      logger.error('Edited workflow state is invalid', { // Log errors.
        errors: validation.errors,
        warnings: validation.warnings,
      })
      throw new Error(`Invalid edited workflow: ${validation.errors.join('; ')}`) // Throw an error.
    }

    if (validation.warnings.length > 0) { // Log any warnings.
      logger.warn('Edited workflow validation warnings', {
        warnings: validation.warnings,
      })
    }

    // Extract and persist custom tools to database.
    if (context?.userId) { // If a userId is available in context.
      try {
        const finalWorkflowState = validation.sanitizedState || modifiedWorkflowState // Use sanitized state if available.
        const { saved, errors } = await extractAndPersistCustomTools( // Persist custom tools.
          finalWorkflowState,
          context.userId
        )

        if (saved > 0) {
          logger.info(`Persisted ${saved} custom tool(s) to database`, { workflowId })
        }

        if (errors.length > 0) {
          logger.warn('Some custom tools failed to persist', { errors, workflowId })
        }
      } catch (error) {
        logger.error('Failed to persist custom tools', { error, workflowId })
      }
    } else {
      logger.warn('No userId in context - skipping custom tools persistence', { workflowId }) // Warn if no userId for persistence.
    }

    logger.info('edit_workflow successfully applied operations', { // Log success metrics.
      operationCount: operations.length,
      blocksCount: Object.keys(modifiedWorkflowState.blocks).length,
      edgesCount: modifiedWorkflowState.edges.length,
      validationErrors: validation.errors.length,
      validationWarnings: validation.warnings.length,
    })

    // Return the modified workflow state for the client to convert to YAML if needed.
    return {
      success: true,
      workflowState: validation.sanitizedState || modifiedWorkflowState, // Return the final, validated workflow state.
    }
  },
}
```