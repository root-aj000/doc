```typescript
import { createLogger } from '@/lib/logs/console/logger'
import { useWorkflowRegistry } from '@/stores/workflows/registry/store'
import { mergeSubblockState } from '@/stores/workflows/utils'
import { useWorkflowStore } from '@/stores/workflows/workflow/store'
import type { BlockState, WorkflowState } from '@/stores/workflows/workflow/types'

const logger = createLogger('Workflows')

/**
 * Get a workflow with its state merged in by ID
 * Note: Since localStorage has been removed, this only works for the active workflow
 * @param workflowId ID of the workflow to retrieve
 * @returns The workflow with merged state values or null if not found/not active
 */
export function getWorkflowWithValues(workflowId: string) {
  // Get the workflow registry store
  const { workflows } = useWorkflowRegistry.getState()
  // Get the ID of the currently active workflow
  const activeWorkflowId = useWorkflowRegistry.getState().activeWorkflowId
  // Get the current state of the workflow store
  const currentState = useWorkflowStore.getState()

  // Check if a workflow with the specified ID exists in the registry.
  if (!workflows[workflowId]) {
    logger.warn(`Workflow ${workflowId} not found`)
    return null
  }

  // Due to removal of localStorage, only the active workflow's state is available.
  if (workflowId !== activeWorkflowId) {
    logger.warn(`Cannot get state for non-active workflow ${workflowId} - localStorage removed`)
    return null
  }

  // Get the metadata of the workflow from the registry
  const metadata = workflows[workflowId]

  // Get the deployment status of the workflow from the registry.
  const deploymentStatus = useWorkflowRegistry.getState().getWorkflowDeploymentStatus(workflowId)

  // Create a `WorkflowState` object with data from the workflow store and registry.
  const workflowState: WorkflowState = {
    // Get the base workflow state from the workflow store. This fetches the basic workflow structure (blocks, edges, loops, parallels) and default values.
    ...useWorkflowStore.getState().getWorkflowState(),
    // Override the `isDeployed` flag using the workflow registry data, using `false` as the default
    isDeployed: deploymentStatus?.isDeployed || false,
    // Override the `deployedAt` timestamp with the registry value, which could be undefined
    deployedAt: deploymentStatus?.deployedAt,
  }

  // Merge the subblock state (e.g., values of input fields within blocks) into the main block state using the `mergeSubblockState` function. This combines the static block definition with the user-entered values.
  const mergedBlocks = mergeSubblockState(workflowState.blocks, workflowId)

  // Return a new object representing the workflow with its merged state.
  return {
    id: workflowId,
    name: metadata.name,
    description: metadata.description,
    color: metadata.color || '#3972F6',
    marketplaceData: metadata.marketplaceData || null,
    workspaceId: metadata.workspaceId,
    folderId: metadata.folderId,
    state: {
      // Include the merged blocks with their up-to-date values
      blocks: mergedBlocks,
      // Get the connection between the blocks
      edges: workflowState.edges,
      // Looping configurations of the workflow
      loops: workflowState.loops,
      // Parallel configurations of the workflow
      parallels: workflowState.parallels,
      // Track when it was last saved
      lastSaved: workflowState.lastSaved,
      // Flag if it is deployed
      isDeployed: workflowState.isDeployed,
      // When it was deployed
      deployedAt: workflowState.deployedAt,
    },
  }
}

/**
 * Get a specific block with its subblock values merged in
 * @param blockId ID of the block to retrieve
 * @returns The block with merged subblock values or null if not found
 */
export function getBlockWithValues(blockId: string): BlockState | null {
  // Get the current workflow state from the store
  const workflowState = useWorkflowStore.getState()
  // Get the ID of the currently active workflow
  const activeWorkflowId = useWorkflowRegistry.getState().activeWorkflowId

  // If there's no active workflow or the requested block doesn't exist in the current state, return null.
  if (!activeWorkflowId || !workflowState.blocks[blockId]) return null

  // Merge the subblock values into the block state for the specified block and active workflow.
  const mergedBlocks = mergeSubblockState(workflowState.blocks, activeWorkflowId, blockId)
  // Return the merged block state, or null if the block is not found after merging.
  return mergedBlocks[blockId] || null
}

/**
 * Get all workflows with their values merged
 * Note: Since localStorage has been removed, this only includes the active workflow state
 * @returns An object containing workflows, with state only for the active workflow
 */
export function getAllWorkflowsWithValues() {
  // Get the workflow definitions from the registry
  const { workflows } = useWorkflowRegistry.getState()
  // Initialize an empty object to store the result
  const result: Record<string, any> = {}
  // Get the ID of the currently active workflow
  const activeWorkflowId = useWorkflowRegistry.getState().activeWorkflowId
  // Get the current state of the workflow from the store
  const currentState = useWorkflowStore.getState()

  // Only sync the active workflow to ensure we always send valid state data
  if (activeWorkflowId && workflows[activeWorkflowId]) {
    // Get the metadata for the active workflow
    const metadata = workflows[activeWorkflowId]

    // Get the deployment status for the active workflow from the registry
    const deploymentStatus = useWorkflowRegistry
      .getState()
      .getWorkflowDeploymentStatus(activeWorkflowId)

    // Ensure state has all required fields for Zod validation
    const workflowState: WorkflowState = {
      // Use the main store's method to get the base workflow state with fallback values
      ...useWorkflowStore.getState().getWorkflowState(),
      // Ensure fallback values for safer handling
      blocks: currentState.blocks || {},
      edges: currentState.edges || [],
      loops: currentState.loops || {},
      parallels: currentState.parallels || {},
      lastSaved: currentState.lastSaved || Date.now(),
      // Override deployment fields with registry-specific deployment status
      isDeployed: deploymentStatus?.isDeployed || false,
      deployedAt: deploymentStatus?.deployedAt,
    }

    // Merge the subblock values for this specific workflow
    const mergedBlocks = mergeSubblockState(workflowState.blocks, activeWorkflowId)

    // Include the API key in the state if it exists in the deployment status
    const apiKey = deploymentStatus?.apiKey

    // Add the active workflow to the result object, including its ID, metadata, and merged state.
    result[activeWorkflowId] = {
      id: activeWorkflowId,
      name: metadata.name,
      description: metadata.description,
      color: metadata.color || '#3972F6',
      marketplaceData: metadata.marketplaceData || null,
      folderId: metadata.folderId,
      state: {
        blocks: mergedBlocks,
        edges: workflowState.edges,
        loops: workflowState.loops,
        parallels: workflowState.parallels,
        lastSaved: workflowState.lastSaved,
        isDeployed: workflowState.isDeployed,
        deployedAt: workflowState.deployedAt,
        marketplaceData: metadata.marketplaceData || null,
      },
      // Include API key if available
      apiKey,
    }

    // Only include workspaceId if it's not null/undefined
    if (metadata.workspaceId) {
      result[activeWorkflowId].workspaceId = metadata.workspaceId
    }
  }

  // Return the result object, which contains the active workflow and its merged state
  return result
}

export { useWorkflowRegistry } from '@/stores/workflows/registry/store'
export type { WorkflowMetadata } from '@/stores/workflows/registry/types'
export { useSubBlockStore } from '@/stores/workflows/subblock/store'
export type { SubBlockStore } from '@/stores/workflows/subblock/types'
export { mergeSubblockState } from '@/stores/workflows/utils'
export { useWorkflowStore } from '@/stores/workflows/workflow/store'
export type { WorkflowState } from '@/stores/workflows/workflow/types'
```

### Purpose of this file:

This file provides functions for accessing and merging workflow data from different stores in a workflow management system.  It focuses on retrieving workflows and their associated state, including the values of individual blocks and sub-blocks within the workflow.  Critically, due to the removal of `localStorage` persistence, it primarily works with the *active* workflow, meaning the workflow currently being edited or viewed by the user. The file also re-exports types and store accessors from other modules.

### Explanation of each section:

1.  **Imports:**
    *   `createLogger`:  A function to create a logger instance for this module, allowing for categorized logging.
    *   `useWorkflowRegistry`: A function to access the Zustand store that holds information about registered workflows (metadata like name, description, deployment status).
    *   `mergeSubblockState`: A utility function (presumably) that merges the state of "sub-blocks" (e.g., individual input fields within a block) into the main block state. This is crucial for capturing user-entered values.
    *   `useWorkflowStore`:  A function to access the Zustand store containing the main workflow state (blocks, edges, loops, parallels).
    *   `WorkflowState`, `BlockState`: Types defining the structure of the workflow and block states, respectively.

2.  **`logger`:**
    *   `const logger = createLogger('Workflows')`
        *   Creates a logger instance specifically for the "Workflows" module.  This allows filtering logs based on the source.

3.  **`getWorkflowWithValues` function:**
    *   **Purpose:** Retrieves a workflow by its ID and merges its state (including sub-block values) into a single object.  It only works for the active workflow due to the removal of `localStorage` persistence.
    *   **Logic:**
        *   `const { workflows } = useWorkflowRegistry.getState()`: Gets all workflows from the `workflowRegistry` store. This provides the metadata of the workflow (name, description, etc.).
        *   `const activeWorkflowId = useWorkflowRegistry.getState().activeWorkflowId`: Gets the ID of the currently active workflow from `workflowRegistry`.
        *   `const currentState = useWorkflowStore.getState()`: Gets the current workflow state, which stores data about the individual blocks, edges, etc.
        *   `if (!workflows[workflowId])`:  Checks if the requested workflow exists in the registry. If not, it logs a warning and returns `null`.
        *   `if (workflowId !== activeWorkflowId)`: Checks if the requested workflow is the active workflow.  If not, it logs a warning and returns `null`, as state is only available for the active workflow.
        *   `const metadata = workflows[workflowId]`:  Gets the metadata of the workflow from the registry.
        *   `const deploymentStatus = useWorkflowRegistry.getState().getWorkflowDeploymentStatus(workflowId)`: Retrieves deployment information from the `WorkflowRegistry` store for a specified workflow ID.
        *   The `workflowState` object is constructed by:
            *   Copying the base workflow state from `useWorkflowStore.getState().getWorkflowState()` (e.g., the basic structure of the workflow).
            *   Overriding the `isDeployed` and `deployedAt` properties with values from the `deploymentStatus`.
        *   `const mergedBlocks = mergeSubblockState(workflowState.blocks, workflowId)`:  This is the critical step of merging the sub-block values into the main block state.  It uses the `mergeSubblockState` function to combine the static block definitions with the user-entered values (presumably stored separately in a sub-block store).
        *   The function then returns a new object containing:
            *   The workflow's `id`, `name`, `description`, `color`, `marketplaceData`, `workspaceId`, and `folderId` from the registry.
            *   A `state` object containing the merged `blocks`, `edges`, `loops`, `parallels`, `lastSaved`, `isDeployed`, and `deployedAt`.

4.  **`getBlockWithValues` function:**
    *   **Purpose:** Retrieves a single block from the active workflow by its ID and merges its sub-block values.
    *   **Logic:**
        *   `const workflowState = useWorkflowStore.getState()`: Gets the current workflow state.
        *   `const activeWorkflowId = useWorkflowRegistry.getState().activeWorkflowId`: Gets the active workflow ID.
        *   `if (!activeWorkflowId || !workflowState.blocks[blockId]) return null`:  Returns `null` if there's no active workflow or the block doesn't exist in the current state.
        *   `const mergedBlocks = mergeSubblockState(workflowState.blocks, activeWorkflowId, blockId)`: Merges the sub-block values for the specified block.  The `blockId` parameter likely restricts the merge to only the specified block.
        *   `return mergedBlocks[blockId] || null`: Returns the merged block state, or `null` if the block is not found after merging.

5.  **`getAllWorkflowsWithValues` function:**
    *   **Purpose:** Retrieves all workflows with their merged state, but only returns the state for the active workflow.
    *   **Logic:**
        *   `const { workflows } = useWorkflowRegistry.getState()`: Gets all workflows from the registry.
        *   `const result: Record<string, any> = {}`: Initializes an empty object to store the result.
        *   `const activeWorkflowId = useWorkflowRegistry.getState().activeWorkflowId`: Gets the active workflow ID.
        *   `const currentState = useWorkflowStore.getState()`: Gets the current state from the `useWorkflowStore`.
        *   The main logic is wrapped in a condition `if (activeWorkflowId && workflows[activeWorkflowId])`, meaning it only processes if there is an active workflow and that workflow exists in the registry.
        *   Inside the `if` block:
            *   `const metadata = workflows[activeWorkflowId]`: Gets the metadata for the active workflow.
            *   `const deploymentStatus = useWorkflowRegistry.getState().getWorkflowDeploymentStatus(activeWorkflowId)`: Gets the deployment status of the workflow from the registry.
            *   A `workflowState` object is constructed similarly to `getWorkflowWithValues`. It uses fallback values in case the data is missing. It copies data from workflow store, then overrides `isDeployed` and `deployedAt` with values from the `deploymentStatus`.
            *   `const mergedBlocks = mergeSubblockState(workflowState.blocks, activeWorkflowId)`: Merges the sub-block values for the active workflow.
            *   `const apiKey = deploymentStatus?.apiKey`: Gets the API key from the deployment status if available.
            *   The code assigns the active workflow to the `result` object, including the merged blocks, deployment status, and API key.
            *  If the workflow has `workspaceId` metadata, it gets added to the result.
        *   `return result`: Returns the `result` object, which will either be empty or contain only the active workflow with its merged state.

6.  **Exports:**
    *   Re-exports several items to make them accessible from other modules:
        *   `useWorkflowRegistry` (from its store file)
        *   `WorkflowMetadata` (from its types file)
        *   `useSubBlockStore` (from its store file)
        *   `SubBlockStore` (from its types file)
        *   `mergeSubblockState` (from its utils file)
        *   `useWorkflowStore` (from its store file)
        *   `WorkflowState` (from its types file)

### Summary and Key Concepts

*   **State Management:** The code heavily relies on Zustand stores for managing workflow data and metadata.  There are two primary stores: `useWorkflowRegistry` (for workflow metadata and deployment status) and `useWorkflowStore` (for the actual workflow structure and state).
*   **Data Merging:** The core logic involves merging data from different stores, particularly the `mergeSubblockState` function. This is necessary to combine the static definition of a workflow block with the dynamic, user-entered values within that block.
*   **Active Workflow Focus:** Due to the removal of `localStorage`, the code is primarily designed to work with the currently active workflow. This constraint significantly simplifies the state management but limits the ability to access and manage the state of inactive workflows.
*   **Deployment Status:** The code retrieves deployment status information from the `WorkflowRegistry` store, allowing it to display whether a workflow is deployed and when it was deployed.
*   **Logging:**  The code uses a custom logger (`createLogger`) to provide categorized logging output, which is helpful for debugging and monitoring the workflow system.
*   **Types:** TypeScript types are used extensively to ensure type safety and improve code maintainability.

In essence, this file acts as a central point for accessing and combining workflow data from different sources. It handles the complexities of merging sub-block values and ensures that only valid state data (primarily for the active workflow) is exposed to other parts of the application.
