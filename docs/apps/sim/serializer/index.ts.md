This TypeScript file defines a `Serializer` class responsible for converting a visual workflow representation (used in a UI, e.g., ReactFlow) into a simplified, "serialized" format for storage or execution, and vice-versa. It also includes robust validation logic to ensure the workflow is correctly configured before execution.

Essentially, it acts as a bridge between the rich UI-centric data model of a workflow and a more compact, execution-ready data model.

---

### **Core Concepts & Purpose**

*   **Workflow Serialization**: Converts `BlockState` objects (representing nodes in a UI), `Edge` objects (connections), `Loop` and `Parallel` configurations into a `SerializedWorkflow` structure. This is often needed when saving a workflow to a database, sending it to a backend API for execution, or sharing it.
*   **Workflow Deserialization**: Reverses the process, taking a `SerializedWorkflow` and reconstructing the `BlockState` and `Edge` objects needed to render the workflow in a UI.
*   **Input Validation**: Before a workflow can be executed, it needs to be validated. This file includes logic to check for:
    *   Missing required inputs for blocks, especially those marked as "user-only."
    *   Correct configuration of "loop" and "parallel" blocks, ensuring they have valid collections to iterate or distribute over.
*   **Mode Handling (Advanced/Basic)**: The serializer intelligently handles different UI modes (e.g., "advanced mode" vs. "basic mode"), ensuring that only relevant parameters are included in the serialized output and validated.
*   **Canonical Parameter Consolidation**: It resolves cases where a single conceptual parameter might be represented by multiple UI fields (e.g., a dropdown for common values and a text input for custom values) into a single, canonical parameter for the backend.
*   **Accessibility Calculation**: Determines which blocks are "ancestors" or "accessible" from any given block, useful for UI features like variable selection.

---

### **Detailed Explanation**

#### **1. Imports**

This section brings in necessary types and utility functions from other parts of the application.

```typescript
import type { Edge } from 'reactflow' // Type definition for edges in a ReactFlow diagram.
import { BlockPathCalculator } from '@/lib/block-path-calculator' // Utility to calculate paths/ancestors between blocks.
import { createLogger } from '@/lib/logs/console/logger' // Function to create a console logger.
import { getBlock } from '@/blocks' // Function to retrieve a block's configuration by its type.
import type { SubBlockConfig } from '@/blocks/types' // Type definition for the configuration of a sub-block (an input field within a block).
import type { SerializedBlock, SerializedWorkflow } from '@/serializer/types' // Type definitions for the serialized block and workflow formats.
import type { BlockState, Loop, Parallel } from '@/stores/workflows/workflow/types' // Type definitions for the UI-state of a block, loop, and parallel construct.
import { getTool } from '@/tools/utils' // Function to retrieve a tool's configuration by its ID.
```

*   **`Edge`**: From `reactflow`, this is the type for the connections between nodes (blocks) in the visual editor.
*   **`BlockPathCalculator`**: A utility likely used to traverse the workflow graph, e.g., to find all blocks that precede a given block.
*   **`createLogger`**: Initializes a logger instance for debugging and error reporting.
*   **`getBlock`**: A helper to fetch the *definition* or *schema* of a block based on its `type` (e.g., 'text-input', 'llm-call').
*   **`SubBlockConfig`**: Defines the structure for configuration of individual input fields or components *within* a block.
*   **`SerializedBlock`, `SerializedWorkflow`**: These are the target types for the serialization process – the flat, execution-ready representation of blocks and the overall workflow.
*   **`BlockState`, `Loop`, `Parallel`**: These represent the *current state* of blocks, loops, and parallel constructs in the UI/editor. They contain all the information, including UI-specific details, that need to be transformed.
*   **`getTool`**: A helper to fetch the *definition* or *schema* of a tool (which a block might wrap) based on its `id`.

#### **2. Logger Initialization**

```typescript
const logger = createLogger('Serializer')
```

This creates a logger instance named 'Serializer'. This logger will be used throughout the class to output information, warnings, and errors to the console, aiding in debugging.

#### **3. `WorkflowValidationError` Class**

This custom error class is used for specific validation failures during the pre-execution checks.

```typescript
/**
 * Structured validation error for pre-execution workflow validation
 */
export class WorkflowValidationError extends Error {
  constructor(
    message: string,
    public blockId?: string, // Optional ID of the block where the error occurred
    public blockType?: string, // Optional type of the block (e.g., 'loop', 'llm-call')
    public blockName?: string // Optional display name of the block
  ) {
    super(message) // Call the parent Error class constructor with the message
    this.name = 'WorkflowValidationError' // Set a custom name for the error
  }
}
```

*   It extends the built-in `Error` class, making it a standard JavaScript error.
*   The constructor takes a `message` and optional `blockId`, `blockType`, and `blockName`. These additional properties provide more context about *where* in the workflow the validation failed, which is extremely useful for displaying user-friendly error messages.
*   `this.name = 'WorkflowValidationError'` ensures that `error.name` will return this specific string, allowing for easier identification of this error type.

#### **4. `shouldIncludeField` Function**

This helper determines whether a particular sub-block (an input field) should be included based on the current UI mode.

```typescript
/**
 * Helper function to check if a subblock should be included in serialization based on current mode
 */
function shouldIncludeField(subBlockConfig: SubBlockConfig, isAdvancedMode: boolean): boolean {
  const fieldMode = subBlockConfig.mode // Get the mode specified in the sub-block's configuration (e.g., 'basic', 'advanced')

  if (fieldMode === 'advanced' && !isAdvancedMode) {
    return false // If the field is 'advanced' mode only, and we're NOT in advanced mode, skip it.
  }

  return true // Otherwise (field is 'basic', or field is 'advanced' and we ARE in advanced mode), include it.
}
```

*   **`subBlockConfig.mode`**: Each sub-block (input field) can be configured to appear in `'basic'` or `'advanced'` mode.
*   **`isAdvancedMode`**: This boolean indicates the current mode of the *overall block* in the UI.
*   The function returns `false` only if a field is explicitly for `'advanced'` mode, but the current context is *not* `isAdvancedMode`. In all other cases (basic field in any mode, or advanced field in advanced mode), it returns `true`.

#### **5. `Serializer` Class**

This is the main class containing all the serialization, deserialization, and validation logic.

##### **`serializeWorkflow` Method**

This is the primary method for converting an entire workflow from its UI state to a serialized format.

```typescript
export class Serializer {
  serializeWorkflow(
    blocks: Record<string, BlockState>, // Map of block IDs to their UI state
    edges: Edge[], // Array of connections between blocks
    loops: Record<string, Loop>, // Map of loop IDs to their configurations
    parallels?: Record<string, Parallel>, // Optional map of parallel IDs to their configurations
    validateRequired = false // Flag to enable/disable pre-execution validation
  ): SerializedWorkflow {
    const safeLoops = loops || {} // Ensure loops is an object, even if undefined
    const safeParallels = parallels || {} // Ensure parallels is an object, even if undefined
    const accessibleBlocksMap = this.computeAccessibleBlockIds(
      blocks,
      edges,
      safeLoops,
      safeParallels
    ) // Calculate which blocks are accessible from each block

    if (validateRequired) {
      this.validateSubflowsBeforeExecution(blocks, safeLoops, safeParallels) // Perform validation checks for loops and parallels
    }

    return {
      version: '1.0', // Current version of the serialization format
      blocks: Object.values(blocks).map((block) =>
        this.serializeBlock(block, {
          validateRequired, // Pass validation flag to individual block serialization
          allBlocks: blocks, // Pass all blocks for potential internal lookups
          accessibleBlocksMap, // Pass the map of accessible blocks
        })
      ),
      connections: edges.map((edge) => ({
        source: edge.source, // ID of the source block
        target: edge.target, // ID of the target block
        sourceHandle: edge.sourceHandle || undefined, // Optional source handle ID
        targetHandle: edge.targetHandle || undefined, // Optional target handle ID
      })),
      loops: safeLoops, // Include loop configurations directly
      parallels: safeParallels, // Include parallel configurations directly
    }
  }
  // ... (rest of the class)
}
```

*   **Inputs**:
    *   `blocks`: An object mapping block IDs to their `BlockState` (the UI representation of each block).
    *   `edges`: An array of `Edge` objects representing connections between blocks.
    *   `loops`: An object mapping loop IDs to their `Loop` configuration.
    *   `parallels?`: An optional object mapping parallel IDs to their `Parallel` configuration.
    *   `validateRequired`: A boolean flag; if `true`, it triggers a validation pass for the workflow.
*   **`safeLoops` and `safeParallels`**: Ensures that `loops` and `parallels` are always treated as objects, even if they are `undefined` or `null` when passed in. This avoids potential runtime errors.
*   **`accessibleBlocksMap`**: Calls `computeAccessibleBlockIds` to build a map indicating for each block, which other blocks (itself, its ancestors, and blocks within subflows) are "accessible." This map is likely used for features like dynamic variable selection in the UI.
*   **`if (validateRequired)`**: If validation is requested, it calls `validateSubflowsBeforeExecution` to check specific conditions for loops and parallels. This will throw a `WorkflowValidationError` if issues are found.
*   **Return Value (`SerializedWorkflow`)**:
    *   `version: '1.0'`: A hardcoded version number for the serialization format. Useful for future compatibility and migrations.
    *   `blocks`: Iterates over all `BlockState` objects, calling `this.serializeBlock` for each one. This recursively serializes individual blocks. It passes `validateRequired`, `allBlocks`, and `accessibleBlocksMap` to `serializeBlock` as context.
    *   `connections`: Maps the `Edge` objects to a simpler format for serialization, primarily containing `source`, `target`, and their respective `Handle` IDs.
    *   `loops`, `parallels`: The `safeLoops` and `safeParallels` objects are included directly as they are already in a serialized-friendly format.

##### **`validateSubflowsBeforeExecution` Method**

This private method performs specific validation checks for `Loop` and `Parallel` blocks, ensuring they have valid inputs for their operational modes.

```typescript
  /**
   * Validate loop and parallel subflows for required inputs when running in "each/collection" modes
   */
  private validateSubflowsBeforeExecution(
    blocks: Record<string, BlockState>,
    loops: Record<string, Loop>,
    parallels: Record<string, Parallel>
  ): void {
    // Validate loops in forEach mode
    Object.values(loops || {}).forEach((loop) => {
      if (!loop) return // Skip if loop object is null/undefined
      if (loop.loopType === 'forEach') { // Only validate loops configured for 'forEach' iteration
        const items = (loop as any).forEachItems // Get the collection/items configured for the loop

        // IIFE (Immediately Invoked Function Expression) to check if 'items' represents a non-empty collection
        const hasNonEmptyCollection = (() => {
          if (items === undefined || items === null) return false // No items provided
          if (Array.isArray(items)) return items.length > 0 // Non-empty array
          if (typeof items === 'object') return Object.keys(items).length > 0 // Non-empty object
          if (typeof items === 'string') { // If it's a string, it might be a variable reference or JSON
            const trimmed = items.trim()
            if (trimmed.length === 0) return false // Empty string
            // If it looks like JSON (starts with '[' or '{'), try to parse it
            if (trimmed.startsWith('[') || trimmed.startsWith('{')) {
              try {
                const parsed = JSON.parse(trimmed) // Attempt to parse as JSON
                if (Array.isArray(parsed)) return parsed.length > 0 // Parsed to non-empty array
                if (parsed && typeof parsed === 'object') return Object.keys(parsed).length > 0 // Parsed to non-empty object
              } catch {
                // If JSON parsing fails, it's not valid JSON, but could still be a variable reference.
                // In this case, we consider it a non-empty string.
                return true
              }
            }
            // Non-JSON string – allow (may be a variable reference/expression like <start.items>)
            return true
          }
          return false // Fallback for other types
        })()

        if (!hasNonEmptyCollection) {
          const blockName = blocks[loop.id]?.name || 'Loop' // Get the loop's display name
          const error = new WorkflowValidationError(
            `${blockName} requires a collection for forEach mode. Provide a non-empty array/object or a variable reference.`,
            loop.id,
            'loop',
            blockName
          )
          throw error // Throw validation error
        }
      }
    })

    // Validate parallels in collection mode (similar logic to loops)
    Object.values(parallels || {}).forEach((parallel) => {
      if (!parallel) return
      if ((parallel as any).parallelType === 'collection') { // Only validate parallels configured for 'collection' mode
        const distribution = (parallel as any).distribution // Get the collection/distribution configured

        const hasNonEmptyDistribution = (() => {
          if (distribution === undefined || distribution === null) return false
          if (Array.isArray(distribution)) return distribution.length > 0
          if (typeof distribution === 'object') return Object.keys(distribution).length > 0
          if (typeof distribution === 'string') {
            const trimmed = distribution.trim()
            if (trimmed.length === 0) return false
            if (trimmed.startsWith('[') || trimmed.startsWith('{')) {
              try {
                const parsed = JSON.parse(trimmed)
                if (Array.isArray(parsed)) return parsed.length > 0
                if (parsed && typeof parsed === 'object') return Object.keys(parsed).length > 0
              } catch {
                return true
              }
            }
            return true
          }
          return false
        })()

        if (!hasNonEmptyDistribution) {
          const blockName = blocks[parallel.id]?.name || 'Parallel'
          const error = new WorkflowValidationError(
            `${blockName} requires a collection for collection mode. Provide a non-empty array/object or a variable reference.`,
            parallel.id,
            'parallel',
            blockName
          )
          throw error
        }
      }
    })
  }
```

*   **Purpose**: Ensures that `forEach` loops have a non-empty collection of items to iterate over, and `collection` parallels have a non-empty distribution. This prevents workflows from failing at runtime due to missing essential inputs for these constructs.
*   **Loop Validation**:
    *   It iterates through all `loops`.
    *   If a `loop` is of `loopType === 'forEach'`, it checks its `forEachItems` property.
    *   **`hasNonEmptyCollection` IIFE**: This is a robust check:
        *   Returns `false` if `items` is `undefined`, `null`, or an empty array/object.
        *   If `items` is a string, it's more complex:
            *   An empty trimmed string results in `false`.
            *   If it *looks like* JSON (starts with `[` or `{`), it attempts `JSON.parse`. If parsing succeeds and results in a non-empty array/object, it's valid.
            *   If JSON parsing fails (e.g., it's not valid JSON, or it's a variable reference like `<start.items>`), it's still considered `true` because it could be a dynamic reference that will resolve to a collection at runtime. This is a conservative approach.
    *   If `hasNonEmptyCollection` is `false`, it throws a `WorkflowValidationError` with a user-friendly message, including the block's details.
*   **Parallel Validation**: The logic for `parallels` in `collection` mode is almost identical, checking the `distribution` property.

##### **`serializeBlock` Method**

This private method converts a single `BlockState` (UI representation) into a `SerializedBlock` (execution-ready format).

```typescript
  private serializeBlock(
    block: BlockState, // The UI state of the block
    options: {
      validateRequired: boolean // Whether to perform validation
      allBlocks: Record<string, BlockState> // All blocks in the workflow (for context)
      accessibleBlocksMap: Map<string, Set<string>> // Map of accessible blocks (for context)
    }
  ): SerializedBlock {
    // Special handling for subflow blocks (loops, parallels, etc.)
    if (block.type === 'loop' || block.type === 'parallel') {
      return {
        id: block.id,
        position: block.position, // Keep position for UI rendering (even if not strictly needed for execution)
        config: {
          tool: '', // Loop/Parallel blocks don't execute a specific tool
          params: block.data || {}, // Preserve the block's internal data (e.g., parallelType, count) as parameters
        },
        inputs: {}, // No direct inputs as a subflow container
        outputs: block.outputs, // Keep outputs for potential connections
        metadata: {
          id: block.type,
          name: block.name,
          description: block.type === 'loop' ? 'Loop container' : 'Parallel container',
          category: 'subflow',
          color: block.type === 'loop' ? '#3b82f6' : '#8b5cf6',
        },
        enabled: block.enabled, // Keep enabled state
      }
    }

    const blockConfig = getBlock(block.type) // Get the block's configuration schema
    if (!blockConfig) {
      throw new Error(`Invalid block type: ${block.type}`) // Error if block type is unknown
    }

    // Extract parameters from UI state, consolidating basic/advanced fields
    const params = this.extractParams(block)

    // Add triggerMode and advancedMode to params if enabled on the block
    try {
      const isTriggerCategory = blockConfig.category === 'triggers'
      if (block.triggerMode === true || isTriggerCategory) {
        params.triggerMode = true
      }
      if (block.advancedMode === true) {
        params.advancedMode = true
      }
    } catch (_) {
      // no-op: conservative, avoid blocking serialization if blockConfig is unexpected
    }

    // Validate required fields that only users can provide (before execution starts)
    if (options.validateRequired) {
      this.validateRequiredFieldsBeforeExecution(block, blockConfig, params)
    }

    let toolId = ''

    if (block.type === 'agent' && params.tools) {
      // Special handling for agent blocks that can use multiple tools
      try {
        // Tools can be an array or a JSON string that needs parsing
        const tools = Array.isArray(params.tools) ? params.tools : JSON.parse(params.tools)

        // If there are custom tools, we just keep them as is. They'll be handled at runtime.
        // For non-custom tools, determine the specific tool ID to use
        const nonCustomTools = tools.filter((tool: any) => tool.type !== 'custom-tool')
        if (nonCustomTools.length > 0) {
          try {
            // Use block config's tool function (if present) to dynamically choose, otherwise take the first available
            toolId = blockConfig.tools.config?.tool
              ? blockConfig.tools.config.tool(params)
              : blockConfig.tools.access[0]
          } catch (error) {
            logger.warn('Tool selection failed during serialization, using default:', {
              error: error instanceof Error ? error.message : String(error),
            })
            toolId = blockConfig.tools.access[0] // Fallback to first tool if selection fails
          }
        }
      } catch (error) {
        logger.error('Error processing tools in agent block:', { error })
        toolId = blockConfig.tools.access[0] // Fallback to first tool on error
      }
    } else {
      // For non-agent blocks, get tool ID from block config as usual
      try {
        // Use block config's tool function (if present) to dynamically choose, otherwise take the first available
        toolId = blockConfig.tools.config?.tool
          ? blockConfig.tools.config.tool(params)
          : blockConfig.tools.access[0]
      } catch (error) {
        logger.warn('Tool selection failed during serialization, using default:', {
          error: error instanceof Error ? error.message : String(error),
        })
        toolId = blockConfig.tools.access[0] // Fallback to first tool if selection fails
      }
    }

    // Get inputs from block config (their types, not values)
    const inputs: Record<string, any> = {}
    if (blockConfig.inputs) {
      Object.entries(blockConfig.inputs).forEach(([key, config]) => {
        inputs[key] = config.type // Store input types (e.g., 'text', 'array')
      })
    }

    return {
      id: block.id,
      position: block.position,
      config: {
        tool: toolId, // The ID of the specific tool to be executed
        params, // The processed parameters for the tool
      },
      inputs, // Expected input types for the block
      outputs: {
        ...block.outputs, // Existing outputs from the UI state
        // Include response format fields if available, parsed safely
        ...(params.responseFormat
          ? {
              responseFormat: this.parseResponseFormatSafely(params.responseFormat),
            }
          : {}),
      },
      metadata: {
        id: block.type, // Block type
        name: block.name, // Display name
        description: blockConfig.description, // Description from config
        category: blockConfig.category, // Category from config
        color: blockConfig.bgColor, // Background color from config
      },
      enabled: block.enabled, // Enabled state
    }
  }
```

*   **Special Handling for `loop` / `parallel` blocks**: These blocks are structural containers, not execution units that run tools. Their serialization is simpler: they mainly retain their ID, position, configuration data (`block.data` becomes `params`), outputs, and metadata. `tool` is explicitly set to an empty string.
*   **`getBlock(block.type)`**: Fetches the blueprint for this block type.
*   **`extractParams(block)`**: This crucial call gathers all the parameters from the block's `subBlocks` (its input fields in the UI), handles basic/advanced mode, and consolidates canonical parameters. The result (`params`) is the set of key-value pairs the backend expects for execution.
*   **`triggerMode` and `advancedMode`**: If these flags are set on the `BlockState` or if the block is a `trigger` category, they are added to the `params`. This information might influence how the block behaves during execution.
*   **`validateRequiredFieldsBeforeExecution`**: If `options.validateRequired` is `true`, this method is called to check for missing essential user inputs.
*   **`toolId` Determination**: This is one of the more complex parts:
    *   **Agent Blocks**: If `block.type` is `'agent'` and it has `params.tools`, it implies the agent can dynamically choose tools. It first tries to parse `params.tools` (which might be a JSON string). It then filters for non-custom tools and attempts to use a `tool` function from `blockConfig.tools.config` (if available) to select the specific tool ID. If that fails, it falls back to the first tool listed in `blockConfig.tools.access`. Errors are logged.
    *   **Non-Agent Blocks**: For other block types, it follows a similar pattern, using `blockConfig.tools.config?.tool(params)` for dynamic selection or falling back to `blockConfig.tools.access[0]`.
*   **`inputs`**: This section populates the `inputs` object with the *types* of inputs expected by the block (e.g., `'text'`, `'boolean'`), derived from `blockConfig.inputs`. The actual *values* of these inputs come from connections, not from `params`.
*   **Return Value (`SerializedBlock`)**: Constructs the `SerializedBlock` object, including the determined `toolId`, `params`, `inputs`, `outputs` (with `responseFormat` parsed), and `metadata` (description, category, color).

##### **`parseResponseFormatSafely` Method**

This private method handles the parsing of the `responseFormat` parameter, which might be a complex JSON string or a simple variable reference.

```typescript
  private parseResponseFormatSafely(responseFormat: any): any {
    if (!responseFormat) {
      return undefined // If no response format is provided, return undefined
    }

    // If already an object, return as-is (already parsed)
    if (typeof responseFormat === 'object' && responseFormat !== null) {
      return responseFormat
    }

    // Handle string values
    if (typeof responseFormat === 'string') {
      const trimmedValue = responseFormat.trim()

      // Check for variable references like <start.input>
      if (trimmedValue.startsWith('<') && trimmedValue.includes('>')) {
        // Keep variable references as-is; they will be resolved at runtime
        return trimmedValue
      }

      if (trimmedValue === '') {
        return undefined // Empty string means no format
      }

      // Try to parse as JSON (e.g., '{ "type": "object", "properties": {} }')
      try {
        return JSON.parse(trimmedValue)
      } catch (error) {
        // If parsing fails, it's not valid JSON. Log a warning and return undefined
        // This avoids crashing the workflow if an invalid string is provided.
        logger.warn('Failed to parse response format as JSON in serializer, using undefined:', {
          value: trimmedValue,
          error: error instanceof Error ? error.message : String(error),
        })
        return undefined
      }
    }

    // For any other type (number, boolean, etc.), return undefined as it's not a valid format
    return undefined
  }
```

*   **Purpose**: To robustly convert the `responseFormat` value (which might come from a text input) into a structured object suitable for execution, or keep it as a variable reference.
*   **Handles various types**:
    *   If `undefined` or `null`, returns `undefined`.
    *   If already an `object` (meaning it's already parsed), returns it as-is.
    *   **If a `string`**:
        *   Checks for variable references (`<...>`) and returns them directly.
        *   If an empty string, returns `undefined`.
        *   Otherwise, it attempts to `JSON.parse` the string. If successful, it returns the parsed object.
        *   If `JSON.parse` fails (meaning it's not valid JSON), it logs a warning and returns `undefined`, preventing crashes.
    *   For any other data type, it returns `undefined`.

##### **`extractParams` Method**

This private method is responsible for taking all values from a block's `subBlocks` (UI input fields) and converting them into a clean `params` object, respecting modes and consolidating canonical parameters.

```typescript
  private extractParams(block: BlockState): Record<string, any> {
    // Special handling for subflow blocks (loops, parallels, etc.)
    if (block.type === 'loop' || block.type === 'parallel') {
      return {} // Loop and parallel blocks don't have traditional tool parameters
    }

    const blockConfig = getBlock(block.type)
    if (!blockConfig) {
      throw new Error(`Invalid block type: ${block.type}`)
    }

    const params: Record<string, any> = {} // The object to store the extracted parameters
    const isAdvancedMode = block.advancedMode ?? false // Determine if the block is in advanced mode
    const isStarterBlock = block.type === 'starter' // Check if it's the special 'starter' block

    // 1. Collect all current values from subBlocks, filtering by mode
    Object.entries(block.subBlocks).forEach(([id, subBlock]) => {
      // Find the corresponding subblock config to check its mode
      const subBlockConfig = blockConfig.subBlocks.find((config) => config.id === id)

      // Special case: include starter block's 'inputFormat' even if it's empty, if it had values originally
      const hasStarterInputFormatValues =
        isStarterBlock &&
        id === 'inputFormat' &&
        Array.isArray(subBlock.value) &&
        subBlock.value.length > 0

      // Include the field if its mode matches the current block mode, or if it's the special starter inputFormat
      if (
        subBlockConfig &&
        (shouldIncludeField(subBlockConfig, isAdvancedMode) || hasStarterInputFormatValues)
      ) {
        params[id] = subBlock.value
      }
    })

    // 2. Then check for any subBlocks with default values that were not explicitly set
    blockConfig.subBlocks.forEach((subBlockConfig) => {
      const id = subBlockConfig.id
      // If the parameter is missing (null/undefined) AND there's a default value function AND it should be included by mode
      if (
        (params[id] === null || params[id] === undefined) &&
        subBlockConfig.value &&
        shouldIncludeField(subBlockConfig, isAdvancedMode)
      ) {
        // Use the default value function, passing current params as context
        params[id] = subBlockConfig.value(params)
      }
    })

    // 3. Finally, consolidate canonical parameters (e.g., selector and manual ID into a single param)
    // This handles cases where a single logical parameter has multiple UI fields (e.g., a dropdown for common choices and a text input for custom values)
    const canonicalGroups: Record<string, { basic?: string; advanced: string[] }> = {}
    blockConfig.subBlocks.forEach((sb) => {
      if (!sb.canonicalParamId) return // Skip if this subBlock doesn't belong to a canonical group
      const key = sb.canonicalParamId // The ID of the canonical parameter (e.g., 'modelId')
      if (!canonicalGroups[key]) canonicalGroups[key] = { basic: undefined, advanced: [] } // Initialize group if not exists
      if (sb.mode === 'advanced') canonicalGroups[key].advanced.push(sb.id) // Add advanced fields
      else canonicalGroups[key].basic = sb.id // Set basic field
    })

    Object.entries(canonicalGroups).forEach(([canonicalKey, group]) => {
      const basicId = group.basic // ID of the basic mode field
      const advancedIds = group.advanced // IDs of advanced mode fields
      const basicVal = basicId ? params[basicId] : undefined // Value of the basic field
      // Find the first non-empty value from advanced fields
      const advancedVal = advancedIds
        .map((id) => params[id])
        .find(
          (v) => v !== undefined && v !== null && (typeof v !== 'string' || v.trim().length > 0)
        )

      let chosen: any
      // Logic to decide which value to pick based on mode and availability
      if (advancedVal !== undefined && basicVal !== undefined) {
        // If both exist, prioritize advanced in advanced mode, basic in basic mode
        chosen = isAdvancedMode ? advancedVal : basicVal
      } else if (advancedVal !== undefined) {
        chosen = advancedVal // If only advanced exists, use it
      } else if (basicVal !== undefined) {
        // If only basic exists, use it only if in basic mode (otherwise it's effectively hidden/ignored)
        chosen = isAdvancedMode ? undefined : basicVal
      } else {
        chosen = undefined // Neither exists
      }

      // Clean up the original sub-block parameters, keeping only the canonical one
      const sourceIds = [basicId, ...advancedIds].filter(Boolean) as string[]
      sourceIds.forEach((id) => {
        if (id !== canonicalKey) delete params[id] // Delete the individual fields
      })
      if (chosen !== undefined) params[canonicalKey] = chosen // Set the canonical parameter
      else delete params[canonicalKey] // Delete canonical if no value was chosen
    })

    return params
  }
```

*   **Purpose**: To aggregate all relevant values from the block's `subBlocks` (which are the UI's internal representation of input fields) into a flat `params` object. It handles three main steps:
    1.  **Collect Current Values**: Iterates through `block.subBlocks` (the actual values entered by the user). For each sub-block, it checks its `subBlockConfig.mode` and the `isAdvancedMode` flag using `shouldIncludeField`. Only fields matching the current mode are initially added to `params`. A special case is made for the `starter` block's `inputFormat` if it has existing values.
    2.  **Apply Default Values**: After collecting user-provided values, it iterates through *all* `blockConfig.subBlocks`. If a parameter is still `null` or `undefined` in `params` and its `subBlockConfig` defines a `value` (which is typically a function that computes a default), and `shouldIncludeField` is true, it calls that function to set the default.
    3.  **Consolidate Canonical Parameters**: This is the most complex part.
        *   **`canonicalGroups`**: It identifies groups of sub-blocks that represent the *same logical parameter* but are displayed differently (e.g., a "Model Selector" dropdown and a "Custom Model ID" text input might both map to a `modelId` canonical parameter). `basic` holds the ID of the field for basic mode, `advanced` holds IDs for advanced mode fields.
        *   For each `canonicalKey` (e.g., `'modelId'`):
            *   It retrieves the values for the `basic` and `advanced` fields.
            *   **`chosen` Logic**: It then applies logic to decide which value (`basicVal` or `advancedVal`) should be `chosen` for the `canonicalKey`.
                *   If both exist, it picks `advancedVal` in `advancedMode` and `basicVal` otherwise.
                *   If only `advancedVal` exists, it's chosen.
                *   If only `basicVal` exists, it's chosen *only if not in advanced mode* (since the basic field would typically be hidden in advanced mode).
            *   **Cleanup**: It deletes the original `basic` and `advanced` sub-block IDs from `params` and adds the `chosen` value under the `canonicalKey` instead. This ensures the serialized `params` object has a single, unambiguous entry for that logical parameter.

##### **`validateRequiredFieldsBeforeExecution` Method**

This private method performs validation on required input fields *before* execution, specifically for fields designated as "user-only."

```typescript
  private validateRequiredFieldsBeforeExecution(
    block: BlockState,
    blockConfig: any, // Block configuration schema
    params: Record<string, any> // The already extracted parameters
  ) {
    // Skip validation if the block is used as a trigger (triggers don't have required "inputs" from user)
    if (
      block.triggerMode === true ||
      blockConfig.category === 'triggers' ||
      params.triggerMode === true
    ) {
      logger.info('Skipping validation for block in trigger mode', {
        blockId: block.id,
        blockType: block.type,
      })
      return
    }

    // Get the tool configuration to check parameter visibility
    const toolAccess = blockConfig.tools?.access
    if (!toolAccess || toolAccess.length === 0) {
      return // No tools associated, nothing to validate
    }

    // Determine the current tool ID using the same logic as the serializer
    let currentToolId = ''
    try {
      currentToolId = blockConfig.tools.config?.tool
        ? blockConfig.tools.config.tool(params)
        : blockConfig.tools.access[0]
    } catch (error) {
      logger.warn('Tool selection failed during validation, using default:', {
        error: error instanceof Error ? error.message : String(error),
      })
      currentToolId = blockConfig.tools.access[0]
    }

    // Get the specific tool configuration to validate against its parameters
    const currentTool = getTool(currentToolId)
    if (!currentTool) {
      return // Tool not found, cannot validate
    }

    // Check required user-only parameters for the current tool
    const missingFields: string[] = []

    // Iterate through the tool's parameters (from the tool definition), not the block's subBlocks
    Object.entries(currentTool.params || {}).forEach(([paramId, paramConfig]) => {
      // We only care about parameters that are 'required' and meant for 'user-only' input
      if (paramConfig.required && paramConfig.visibility === 'user-only') {
        // Find the corresponding subBlock configuration to check its mode and conditions
        const subBlockConfig = blockConfig.subBlocks?.find((sb: any) => sb.id === paramId)

        let shouldValidateParam = true // Assume we should validate unless conditions say otherwise

        if (subBlockConfig) {
          const isAdvancedMode = block.advancedMode ?? false
          const includedByMode = shouldIncludeField(subBlockConfig, isAdvancedMode) // Check if visible based on advanced mode

          // Complex IIFE to evaluate conditional visibility of the parameter
          const includedByCondition = (() => {
            const evalCond = (
              condition: // The condition object from subBlockConfig
                | {
                    field: string
                    value: any
                    not?: boolean
                    and?: { field: string; value: any; not?: boolean }
                  }
                | (() => {
                    field: string
                    value: any
                    not?: boolean
                    and?: { field: string; value: any; not?: boolean }
                  })
                | undefined,
              values: Record<string, any> // The current `params` to evaluate against
            ): boolean => {
              if (!condition) return true // No condition means it's always included
              const actual = typeof condition === 'function' ? condition() : condition // Resolve condition if it's a function
              const fieldValue = values[actual.field] // Get the value of the field being checked in the condition

              // Check if the field's value matches the condition's value (considering 'not' and array values)
              const valueMatch = Array.isArray(actual.value)
                ? fieldValue != null &&
                  (actual.not
                    ? !actual.value.includes(fieldValue)
                    : actual.value.includes(fieldValue))
                : actual.not
                  ? fieldValue !== actual.value
                  : fieldValue === actual.value

              // Check for an optional 'and' condition
              const andMatch = !actual.and
                ? true // No 'and' condition, so it's true
                : (() => {
                    const andFieldValue = values[actual.and!.field] // Value of the field in the 'and' condition
                    return Array.isArray(actual.and!.value)
                      ? andFieldValue != null &&
                          (actual.and!.not
                            ? !actual.and!.value.includes(andFieldValue)
                            : actual.and!.value.includes(andFieldValue))
                      : actual.and!.not
                        ? andFieldValue !== actual.and!.value
                        : andFieldValue === actual.and!.value
                  })()

              return valueMatch && andMatch // Both conditions must be true
            }

            return evalCond(subBlockConfig.condition, params) // Evaluate the subBlock's condition
          })()

          shouldValidateParam = includedByMode && includedByCondition // Only validate if visible by mode AND condition
        }

        if (!shouldValidateParam) {
          return // Skip validation for this parameter if it's not currently visible/active
        }

        const fieldValue = params[paramId] // Get the parameter's value from the extracted params
        if (fieldValue === undefined || fieldValue === null || fieldValue === '') {
          const displayName = subBlockConfig?.title || paramId // Get a user-friendly name for the missing field
          missingFields.push(displayName) // Add to the list of missing fields
        }
      }
    })

    if (missingFields.length > 0) {
      const blockName = block.name || blockConfig.name || 'Block'
      throw new Error(`${blockName} is missing required fields: ${missingFields.join(', ')}`) // Throw error with missing fields
    }
  }
```

*   **Purpose**: To ensure that all *required* fields that a *user must provide* are filled out before a workflow execution starts. This prevents common runtime errors due to incomplete configuration.
*   **Skip Trigger Blocks**: Blocks configured as `triggerMode` or belonging to the `'triggers'` category are skipped because triggers don't typically have "required" user inputs in the same way regular action blocks do.
*   **Tool ID Determination**: Similar to `serializeBlock`, it determines the `currentToolId` based on the block's `params` and `blockConfig`. Validation is done against the specific tool that will be used.
*   **Iterate Tool Parameters**: It iterates through `currentTool.params` (from the tool's definition), not the block's `subBlocks`. This is because the tool definition specifies what parameters it *actually* needs and if they are required.
    *   It only considers parameters where `paramConfig.required` is `true` and `paramConfig.visibility` is `'user-only'`.
    *   **`shouldValidateParam` Logic**: This determines if a required parameter is *currently relevant* for validation. It depends on:
        *   `includedByMode`: Whether the sub-block is visible based on `isAdvancedMode` (using `shouldIncludeField`).
        *   `includedByCondition`: Whether the sub-block is visible based on its conditional logic (`subBlockConfig.condition`).
            *   **`evalCond` IIFE**: This function evaluates the complex `condition` object (which might also be a function) against the current `params`. It supports `field`, `value`, `not`, and an optional `and` clause to check for conditional visibility. If `evalCond` returns `false`, the field is not considered visible/active, so it shouldn't be validated.
    *   If `shouldValidateParam` is `false`, the parameter is skipped for validation.
    *   **Check Value**: If the parameter *should* be validated, it checks `params[paramId]`. If the `fieldValue` is `undefined`, `null`, or an empty string, it's considered missing, and its display name is added to `missingFields`.
*   **Error Throwing**: If `missingFields` is not empty after checking all parameters, it throws a standard `Error` (not `WorkflowValidationError` here, which is interesting; it could be a `WorkflowValidationError` too for consistency) with a message listing all missing fields.

##### **`computeAccessibleBlockIds` Method**

This private method calculates which blocks are "accessible" from any given block within the workflow.

```typescript
  private computeAccessibleBlockIds(
    blocks: Record<string, BlockState>,
    edges: Edge[],
    loops: Record<string, Loop>,
    parallels: Record<string, Parallel>
  ): Map<string, Set<string>> {
    const accessibleMap = new Map<string, Set<string>>() // Map: blockId -> Set of accessible block IDs
    // Convert ReactFlow edges to a simpler format for BlockPathCalculator
    const simplifiedEdges = edges.map((edge) => ({ source: edge.source, target: edge.target }))

    // Find the starter block (if any)
    const starterBlock = Object.values(blocks).find((block) => block.type === 'starter')

    Object.keys(blocks).forEach((blockId) => {
      // Find all blocks that are ancestors (predecessors) of the current block
      const ancestorIds = BlockPathCalculator.findAllPathNodes(simplifiedEdges, blockId)
      const accessibleIds = new Set<string>(ancestorIds)
      accessibleIds.add(blockId) // A block is always accessible to itself

      // The starter block is generally accessible from anywhere (e.g., for variables)
      if (starterBlock) {
        accessibleIds.add(starterBlock.id)
      }

      // If the current block is inside a loop, all other nodes in that loop are accessible
      Object.values(loops).forEach((loop) => {
        if (!loop?.nodes) return
        if (loop.nodes.includes(blockId)) {
          loop.nodes.forEach((nodeId) => accessibleIds.add(nodeId))
        }
      })

      // If the current block is inside a parallel, all other nodes in that parallel are accessible
      Object.values(parallels).forEach((parallel) => {
        if (!parallel?.nodes) return
        if (parallel.nodes.includes(blockId)) {
          parallel.nodes.forEach((nodeId) => accessibleIds.add(nodeId))
        }
      })

      accessibleMap.set(blockId, accessibleIds) // Store the computed set for this block
    })

    return accessibleMap
  }
```

*   **Purpose**: To create a map where each block ID points to a set of other block IDs that are considered "accessible" from it. This is typically used in the UI to determine which variables or outputs from other blocks can be referenced by the current block.
*   **`simplifiedEdges`**: Transforms ReactFlow `Edge` objects into a simpler `source`/`target` format required by `BlockPathCalculator`.
*   **`starterBlock`**: Finds the special `starter` block, as its outputs are usually globally accessible.
*   **Iterates All Blocks**: For each `blockId`:
    *   **`BlockPathCalculator.findAllPathNodes`**: Uses the utility to find all ancestor blocks (blocks that lead into the current `blockId`).
    *   The `accessibleIds` set is populated with ancestors and the block itself.
    *   The `starterBlock.id` is added if it exists.
    *   **Loop/Parallel Scope**: It checks if the current `blockId` is part of any `loop` or `parallel` construct. If it is, then *all* nodes within that `loop` or `parallel` are also added to `accessibleIds`, reflecting that blocks within the same subflow often have access to each other's outputs.
*   The resulting `accessibleMap` provides a comprehensive view of contextual accessibility for each block.

##### **`deserializeWorkflow` Method**

This method takes a `SerializedWorkflow` and converts it back into the UI-friendly `blocks` and `edges` format.

```typescript
  deserializeWorkflow(workflow: SerializedWorkflow): {
    blocks: Record<string, BlockState> // Reconstructed UI blocks
    edges: Edge[] // Reconstructed UI edges
  } {
    const blocks: Record<string, BlockState> = {}
    const edges: Edge[] = []

    // Deserialize blocks
    workflow.blocks.forEach((serializedBlock) => {
      const block = this.deserializeBlock(serializedBlock) // Convert each serialized block
      blocks[block.id] = block // Store in the blocks map
    })

    // Deserialize connections
    workflow.connections.forEach((connection) => {
      edges.push({
        id: crypto.randomUUID(), // Assign a new random ID for ReactFlow (not stored in serialized form)
        source: connection.source,
        target: connection.target,
        sourceHandle: connection.sourceHandle,
        targetHandle: connection.targetHandle,
      })
    })

    return { blocks, edges }
  }
```

*   **Purpose**: To load a workflow from a serialized format (e.g., fetched from a database) back into the UI's internal data structures.
*   **Deserialize Blocks**: It iterates through `workflow.blocks`, calling `this.deserializeBlock` for each `serializedBlock` to convert it back to `BlockState` and stores it in the `blocks` map.
*   **Deserialize Connections**: It iterates through `workflow.connections`, creating `Edge` objects. A new `id` for each edge is generated using `crypto.randomUUID()` because `Edge` IDs are often dynamic for UI frameworks like ReactFlow and might not be stored directly in the serialized format.

##### **`deserializeBlock` Method**

This private method converts a single `SerializedBlock` back into a `BlockState` (UI representation).

```typescript
  private deserializeBlock(serializedBlock: SerializedBlock): BlockState {
    const blockType = serializedBlock.metadata?.id // Get the block type from metadata
    if (!blockType) {
      throw new Error(`Invalid block type: ${serializedBlock.metadata?.id}`) // Error if type is missing
    }

    // Special handling for subflow blocks (loops, parallels, etc.)
    if (blockType === 'loop' || blockType === 'parallel') {
      return {
        id: serializedBlock.id,
        type: blockType,
        name: serializedBlock.metadata?.name || (blockType === 'loop' ? 'Loop' : 'Parallel'),
        position: serializedBlock.position,
        subBlocks: {}, // Loop/parallel blocks don't have traditional input fields as subBlocks
        outputs: serializedBlock.outputs,
        enabled: serializedBlock.enabled ?? true, // Default to true if not specified
        data: serializedBlock.config.params, // Restore the internal data (e.g., parallelType, count)
      }
    }

    const blockConfig = getBlock(blockType) // Get the block's configuration schema
    if (!blockConfig) {
      throw new Error(`Invalid block type: ${blockType}`)
    }

    const subBlocks: Record<string, any> = {}
    // Populate subBlocks based on block configuration and serialized parameters
    blockConfig.subBlocks.forEach((subBlock) => {
      subBlocks[subBlock.id] = {
        id: subBlock.id,
        type: subBlock.type,
        // Get the value from serializedBlock.config.params, default to null if not found
        value: serializedBlock.config.params[subBlock.id] ?? null,
      }
    })

    return {
      id: serializedBlock.id,
      type: blockType,
      name: serializedBlock.metadata?.name || blockConfig.name,
      position: serializedBlock.position,
      subBlocks,
      outputs: serializedBlock.outputs,
      enabled: true, // Default enabled state for deserialized blocks
      // Restore triggerMode and advancedMode from serialized parameters or metadata category
      triggerMode:
        serializedBlock.config?.params?.triggerMode === true ||
        serializedBlock.metadata?.category === 'triggers',
      advancedMode: serializedBlock.config?.params?.advancedMode === true,
    }
  }
```

*   **Purpose**: To reconstruct a single `BlockState` object suitable for the UI from its `SerializedBlock` representation.
*   **Special Handling for `loop` / `parallel` blocks**: Similar to serialization, these structural blocks are handled simply, restoring their basic properties and the internal `data` (which stored their `parallelType`, `count`, etc.). `subBlocks` are empty.
*   **`getBlock(blockType)`**: Fetches the blueprint for the block type.
*   **`subBlocks` Population**: It iterates through `blockConfig.subBlocks` (the definition of all possible input fields for this block type). For each field, it attempts to find its value in `serializedBlock.config.params`. If found, that value is used; otherwise, `null` is used as a default.
*   **Return Value (`BlockState`)**: Constructs the `BlockState` object, restoring `id`, `type`, `name`, `position`, `subBlocks`, `outputs`, `enabled` (defaulting to `true`), and crucially, `triggerMode` and `advancedMode` from the serialized `params` or metadata.

---

This `Serializer` class is a central piece of infrastructure for any application that manages complex workflows, allowing them to be persisted, transmitted, validated, and rendered in a user interface consistently. The careful handling of modes, canonical parameters, and various data types ensures a robust and flexible system.