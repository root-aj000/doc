This TypeScript file defines the `InputResolver` class, a critical component in a workflow execution system. Its primary responsibility is to take a raw block configuration and resolve all dynamic input values, transforming them into concrete data that a block can use during execution.

This involves:
*   **Reference Resolution:** Looking up values from previously executed blocks (e.g., `<blockName.output.field>`), global workflow variables (`<variable.myVar>`), environment variables (`{{ENV_VAR}}`), and loop/parallel specific data (`<loop.currentItem>`, `<parallel.index>`).
*   **Conditional Input Filtering:** Disabling certain input fields based on other input values (e.g., show a 'database name' field only if 'database type' is selected).
*   **Type Coercion and Formatting:** Converting resolved values into the correct data types (e.g., parsing JSON strings, formatting values appropriately for JavaScript code contexts).
*   **Accessibility Checks:** Ensuring that a block can only reference other blocks it is logically "connected" to within the workflow.

In essence, `InputResolver` acts as the "translator" that turns abstract workflow definitions with placeholders into executable instructions with concrete data.

---

### File Overview

```typescript
import { createLogger } from '@/lib/logs/console/logger' // For logging messages
import { VariableManager } from '@/lib/variables/variable-manager' // Manages workflow variables
import { extractReferencePrefixes, SYSTEM_REFERENCE_PREFIXES } from '@/lib/workflows/references' // Helps identify and categorize references
import { TRIGGER_REFERENCE_ALIAS_MAP } from '@/lib/workflows/triggers' // Maps trigger block names to their types
import { getBlock } from '@/blocks/index' // Retrieves block configuration schemas
import type { LoopManager } from '@/executor/loops/loops' // Type definition for loop manager
import type { ExecutionContext } from '@/executor/types' // Type definition for execution context
import type { SerializedBlock, SerializedWorkflow } from '@/serializer/types' // Type definitions for workflow and block structures
import { normalizeBlockName } from '@/stores/workflows/utils' // Utility for normalizing block names

// Initialize a logger specifically for the InputResolver class
const logger = createLogger('InputResolver')

// A simple utility function to access a property on an object.
// Used for navigating object paths in references.
function resolvePropertyAccess(obj: any, property: string): any {
  return obj[property]
}

export class InputResolver {
  // Maps block IDs to their serialized block configurations for quick lookup
  private blockById: Map<string, SerializedBlock>
  // Maps normalized block names (lowercase, no spaces) to their serialized block configurations
  private blockByNormalizedName: Map<string, SerializedBlock>
  // Maps a block's ID to the ID of the loop it belongs to
  private loopsByBlockId: Map<string, string>
  // Maps a block's ID to the ID of the parallel block it belongs to
  private parallelsByBlockId: Map<string, string>

  /**
   * Constructs an instance of InputResolver.
   * Initializes internal maps for efficient lookups of blocks, loops, and parallels.
   *
   * @param workflow - The entire serialized workflow definition.
   * @param environmentVariables - Key-value pairs of environment variables.
   * @param workflowVariables - Key-value pairs of workflow-specific variables.
   * @param loopManager - (Optional) Manages loop-related operations.
   * @param accessibleBlocksMap - (Optional) A pre-calculated map showing which blocks are accessible from each block.
   */
  constructor(
    private workflow: SerializedWorkflow, // The workflow structure
    private environmentVariables: Record<string, string>, // Env vars from execution context
    private workflowVariables: Record<string, any> = {}, // Workflow variables
    private loopManager?: LoopManager, // Optional loop manager
    public accessibleBlocksMap?: Map<string, Set<string>> // Optional pre-calculated accessibility map
  ) {
    // Create a Map for quick lookup of blocks by their ID
    this.blockById = new Map(workflow.blocks.map((block) => [block.id, block]))

    // Create a Map for quick lookup of blocks by their normalized name.
    // If a block has a name, it's normalized; otherwise, its ID is used as the key.
    this.blockByNormalizedName = new Map(
      workflow.blocks.map((block) => [
        block.metadata?.name ? this.normalizeBlockName(block.metadata.name) : block.id,
        block,
      ])
    )

    // Special handling for the 'starter' block: it can always be referenced as "start".
    const starterBlock = workflow.blocks.find((block) => block.metadata?.id === 'starter')
    if (starterBlock) {
      this.blockByNormalizedName.set('start', starterBlock)
      // Also add its actual normalized name if it has one.
      if (starterBlock.metadata?.name) {
        this.blockByNormalizedName.set(
          this.normalizeBlockName(starterBlock.metadata.name),
          starterBlock
        )
      }
    }

    // Populate the loopsByBlockId map: iterate through all loops in the workflow
    // and for each block within a loop, map its ID to the parent loop's ID.
    this.loopsByBlockId = new Map()
    for (const [loopId, loop] of Object.entries(workflow.loops || {})) {
      for (const blockId of loop.nodes) {
        this.loopsByBlockId.set(blockId, loopId)
      }
    }

    // Populate the parallelsByBlockId map: iterate through all parallels in the workflow
    // and for each block within a parallel, map its ID to the parent parallel's ID.
    this.parallelsByBlockId = new Map()
    for (const [parallelId, parallel] of Object.entries(workflow.parallels || {})) {
      for (const blockId of parallel.nodes) {
        this.parallelsByBlockId.set(blockId, parallelId)
      }
    }
  }

  /**
   * Evaluates if a sub-block (an input field that can be conditionally shown/hidden) should be active
   * based on its defined condition and the current values of other inputs.
   *
   * @param condition - The condition object (or a function returning one) to evaluate.
   * @param currentValues - An object containing the current values of all input parameters for the block.
   * @returns True if the sub-block should be active (condition is met or no condition), false otherwise.
   */
  private evaluateSubBlockCondition(
    condition:
      | {
          field: string // The input field whose value is checked
          value: any // The target value to compare against
          not?: boolean // If true, the condition is "field value is NOT target value"
          and?: { field: string; value: any; not?: boolean } // An optional secondary condition
        }
      | (() => {
          field: string
          value: any
          not?: boolean
          and?: { field: string; value: any; not?: boolean }
        })
      | undefined,
    currentValues: Record<string, any>
  ): boolean {
    // If no condition is defined, the sub-block is always active.
    if (!condition) return true

    // If the condition is a function, call it to get the actual condition object.
    const actualCondition = typeof condition === 'function' ? condition() : condition

    // Get the value of the input field specified in the condition.
    const fieldValue = currentValues[actualCondition.field]

    // Determine if the field value matches the condition's primary value.
    // Handles both single values and array of values.
    const isValueMatch = Array.isArray(actualCondition.value) // If target value is an array...
      ? fieldValue != null && // ...ensure field value is not null/undefined...
        (actualCondition.not // ...then check if it's NOT in the array or IS in the array.
          ? !actualCondition.value.includes(fieldValue)
          : actualCondition.value.includes(fieldValue))
      : actualCondition.not // If target value is not an array, do a direct comparison (not equal or equal).
        ? fieldValue !== actualCondition.value
        : fieldValue === actualCondition.value

    // Evaluate the optional 'and' condition if it exists.
    const isAndValueMatch =
      !actualCondition.and || // If no 'and' condition, it's considered true
      (() => {
        // Get the field value for the 'and' condition.
        const andFieldValue = currentValues[actualCondition.and!.field]
        // Similar logic as above for checking array or direct match for the 'and' condition.
        return Array.isArray(actualCondition.and!.value)
          ? andFieldValue != null &&
              (actualCondition.and!.not
                ? !actualCondition.and!.value.includes(andFieldValue)
                : actualCondition.and!.value.includes(andFieldValue))
          : actualCondition.and!.not
            ? andFieldValue !== actualCondition.and!.value
            : andFieldValue === actualCondition.and!.value
      })() // Immediately invoked function expression for encapsulation

    // Both conditions must be true for the sub-block to be active.
    return isValueMatch && isAndValueMatch
  }

  /**
   * Filters the block's input parameters based on the conditions defined for its sub-blocks.
   * Only inputs whose sub-block conditions are met will be included.
   *
   * @param block - The serialized block configuration.
   * @param inputs - A record of all input parameters for the block.
   * @returns A new record containing only the input parameters that should be processed.
   */
  private filterInputsByConditions(
    block: SerializedBlock,
    inputs: Record<string, any>
  ): Record<string, any> {
    const blockType = block.metadata?.id
    // If the block has no type, or cannot retrieve its configuration, return all inputs.
    if (!blockType) return inputs

    const blockConfig = getBlock(blockType)
    // If no block configuration or no sub-blocks are defined, return all inputs.
    if (!blockConfig || !blockConfig.subBlocks) return inputs

    const filteredInputs: Record<string, any> = {}
    // Iterate over all provided inputs for the block.
    for (const [key, value] of Object.entries(inputs)) {
      let shouldInclude = false

      // Find all sub-block configurations that match the current input key (ID).
      const matchingSubBlocks = blockConfig.subBlocks.filter((sb) => sb.id === key)

      if (matchingSubBlocks.length === 0) {
        // If no sub-block config is found for this input, it's a standard input and should always be included.
        shouldInclude = true
      } else {
        // If matching sub-blocks are found, check if ANY of them should be active.
        // An input is included if at least one of its corresponding sub-block conditions is met.
        for (const subBlock of matchingSubBlocks) {
          if (!subBlock.condition || this.evaluateSubBlockCondition(subBlock.condition, inputs)) {
            shouldInclude = true
            break // Found an active sub-block, no need to check others for this key.
          }
        }
      }

      // If the input should be included, add it to the filtered results.
      if (shouldInclude) {
        filteredInputs[key] = value
      }
    }

    return filteredInputs
  }

  /**
   * The core method to resolve all input parameters for a given block.
   * It handles various types of references (block, variable, environment, loop, parallel)
   * and ensures values are properly formatted for their destination.
   *
   * @param block - The serialized block whose inputs need to be resolved.
   * @param context - The current execution context, containing block states, loop information, etc.
   * @returns A record of resolved input parameters.
   */
  resolveInputs(block: SerializedBlock, context: ExecutionContext): Record<string, any> {
    // Start with all parameters from the block's configuration.
    const allInputs = { ...block.config.params }
    // Filter these inputs based on sub-block conditions to only process active fields.
    const inputs = this.filterInputsByConditions(block, allInputs)
    const result: Record<string, any> = {}

    // Process each input parameter.
    for (const [key, value] of Object.entries(inputs)) {
      // Skip null or undefined values, assigning them directly.
      if (value === null || value === undefined) {
        result[key] = value
        continue
      }

      // *** Special early check for 'conditions' key in 'condition' blocks ***
      // Condition blocks' 'conditions' field expects a raw string, not resolved references or JSON parsing.
      const isConditionBlock = block.metadata?.id === 'condition'
      const isConditionsKey = key === 'conditions'

      if (isConditionBlock && isConditionsKey && typeof value === 'string') {
        result[key] = value // Assign the raw string directly.
        continue // Skip all further processing for this key.
      }
      // *** End of early check ***

      // Handle string values that may contain references or interpolations.
      if (typeof value === 'string') {
        const trimmedValue = value.trim()

        // 1. Check for direct variable reference pattern: `<variable.name>`
        const directVariableMatch = trimmedValue.match(/^<variable\.([^>]+)>$/)
        if (directVariableMatch) {
          const variableName = directVariableMatch[1]
          const variable = this.findVariableByName(variableName)

          if (variable) {
            result[key] = this.getTypedVariableValue(variable) // Get the correctly typed value.
            continue
          }
          logger.warn(
            `Direct variable reference <variable.${variableName}> not found. Treating as literal.`
          )
          result[key] = value // If not found, use the original string literally.
          continue
        }

        // 2. Check for direct loop reference pattern: `<loop.property>`
        const directLoopMatch = trimmedValue.match(/^<loop\.([^>]+)>$/)
        if (directLoopMatch) {
          const containingLoopId = this.loopsByBlockId.get(block.id) // Find the loop this block belongs to.

          if (containingLoopId) {
            const pathParts = directLoopMatch[1].split('.')
            // Resolve the loop reference, not in a template literal context initially.
            const loopValue = this.resolveLoopReference(
              containingLoopId,
              pathParts,
              context,
              block,
              false
            )

            if (loopValue !== null) {
              // Attempt to parse the value as JSON; otherwise, use it as a string.
              try {
                result[key] = JSON.parse(loopValue)
              } catch {
                result[key] = loopValue
              }
              continue
            }
          }

          logger.warn(`Direct loop reference <loop.${directLoopMatch[1]}> could not be resolved.`)
          result[key] = value // If not found, use the original string literally.
          continue
        }

        // 3. Check for direct parallel reference pattern: `<parallel.property>`
        const directParallelMatch = trimmedValue.match(/^<parallel\.([^>]+)>$/)
        if (directParallelMatch) {
          // Find the parallel block this current block is part of.
          let containingParallelId: string | undefined
          for (const [parallelId, parallel] of Object.entries(context.workflow?.parallels || {})) {
            if (parallel.nodes.includes(block.id)) {
              containingParallelId = parallelId
              break
            }
          }

          if (containingParallelId) {
            const pathParts = directParallelMatch[1].split('.')
            // Resolve the parallel reference, not in a template literal context initially.
            const parallelValue = this.resolveParallelReference(
              containingParallelId,
              pathParts,
              context,
              block,
              false
            )

            if (parallelValue !== null) {
              // Attempt to parse the value as JSON; otherwise, use it as a string.
              try {
                result[key] = JSON.parse(parallelValue)
              } catch {
                result[key] = parallelValue
              }
              continue
            }
          }

          logger.warn(
            `Direct parallel reference <parallel.${directParallelMatch[1]}> could not be resolved.`
          )
          result[key] = value // If not found, use the original string literally.
          continue
        }

        // 4. For all other string values, process them with the full resolution pipeline.
        result[key] = this.processStringValue(value, key, context, block)
      }
      // Handle objects and arrays recursively by calling a helper method.
      else if (typeof value === 'object') {
        result[key] = this.processObjectValue(value, context, block)
      }
      // For all other primitive types (numbers, booleans), pass them through directly.
      else {
        result[key] = value
      }
    }

    return result
  }

  /**
   * Retrieves the correctly typed value of a workflow variable.
   * It leverages the `VariableManager` for consistent type handling (e.g., "plain" for string, "number" for numbers).
   *
   * @param variable - The variable object (e.g., from `workflowVariables`).
   * @returns The actual typed value of the variable.
   */
  private getTypedVariableValue(variable: any): any {
    // Return null/undefined as is.
    if (!variable || variable.value === undefined || variable.value === null) {
      return variable?.value
    }

    try {
      // Normalize 'string' type to 'plain' for consistency with `VariableManager`.
      const type = variable.type === 'string' ? 'plain' : variable.type

      // Use `VariableManager.resolveForExecution` to get the value in its correct type.
      return VariableManager.resolveForExecution(variable.value, type)
    } catch (error) {
      // Log an error if resolution fails and return the original raw value.
      logger.error(`Error processing variable ${variable.name} (type: ${variable.type}):`, error)
      return variable.value
    }
  }

  /**
   * Formats a typed variable value into a string suitable for interpolation.
   * It considers the variable's type and the context of the consuming block
   * (e.g., if it's a code block, it might need different formatting).
   *
   * @param value - The typed value obtained from `getTypedVariableValue`.
   * @param type - The original variable type (e.g., 'string', 'number', 'plain').
   * @param currentBlock - The block where this value will be used, to determine context.
   * @returns A string representation suitable for insertion into another string.
   */
  private formatValueForInterpolation(
    value: any,
    type: string,
    currentBlock?: SerializedBlock
  ): string {
    try {
      // Normalize 'string' type to 'plain'.
      const normalizedType = type === 'string' ? 'plain' : type

      // For 'plain' text variables that are already strings, use them as-is without modification.
      if (normalizedType === 'plain' && typeof value === 'string') {
        return value
      }

      // Determine if the consuming block requires the value to be formatted for a code context.
      const needsCodeStringLiteral = this.needsCodeStringLiteral(currentBlock, String(value))
      const isFunctionBlock = currentBlock?.metadata?.id === 'function'

      // If it's a 'function' block or needs code literal formatting, use `formatForCodeContext`.
      if (isFunctionBlock || needsCodeStringLiteral) {
        return VariableManager.formatForCodeContext(value, normalizedType as any)
      }
      // Otherwise, use `formatForTemplateInterpolation` for general string embedding.
      return VariableManager.formatForTemplateInterpolation(value, normalizedType as any)
    } catch (error) {
      logger.error(`Error formatting value for interpolation (type: ${type}):`, error)
      // Fallback to simple string conversion on error.
      return String(value)
    }
  }

  /**
   * Resolves workflow variable references (e.g., `<variable.name>`) embedded within a string.
   * It finds matching placeholders, retrieves the variable's typed value, and formats it for interpolation.
   *
   * @param value - The input string potentially containing variable references.
   * @param currentBlock - The block currently being processed (context for formatting).
   * @returns The string with all found variable references replaced by their resolved values.
   */
  resolveVariableReferences(value: string, currentBlock?: SerializedBlock): string {
    // If the input isn't a string (e.g., already resolved to an object), return it directly.
    if (typeof value !== 'string') {
      return value as any
    }

    // Find all occurrences of the variable reference pattern.
    const variableMatches = value.match(/<variable\.([^>]+)>/g)
    if (!variableMatches) return value // No matches, return original string.

    let resolvedValue = value

    // Iterate through each matched reference.
    for (const match of variableMatches) {
      const variableName = match.slice('<variable.'.length, -1) // Extract the variable name.

      const variable = this.findVariableByName(variableName) // Find the variable in workflowVariables.

      if (variable) {
        const typedValue = this.getTypedVariableValue(variable) // Get its typed value.
        const formattedValue: string = this.formatValueForInterpolation(
          typedValue,
          variable.type,
          currentBlock
        ) // Format for string interpolation.
        resolvedValue = resolvedValue.replace(match, formattedValue) // Replace the placeholder.
      } else {
        logger.warn(
          `Interpolated variable reference <variable.${variableName}> not found. Leaving as literal.`
        )
        // If not found, the placeholder remains in the string.
      }
    }

    return resolvedValue
  }

  /**
   * Resolves block references (e.g., `<blockId.property>` or `<blockName.property>`) within a string.
   * This is a complex method that handles:
   * - Trigger block aliases (start, api, chat, manual)
   * - Loop and Parallel specific references (`loop.currentItem`, `parallel.index`)
   * - Standard block outputs, including nested paths and array indexing.
   * - Accessibility checks: ensuring the referenced block is connected and in the active execution path.
   * - Formatting values based on the *current block's* type (e.g., for code vs. JSON).
   *
   * @param value - The input string potentially containing block references.
   * @param context - The current execution context.
   * @param currentBlock - The block making the reference.
   * @returns The string with all found block references replaced by their resolved values.
   * @throws Error if a referenced block is not found, not accessible, disabled, or an invalid path is used.
   */
  resolveBlockReferences(
    value: string,
    context: ExecutionContext,
    currentBlock: SerializedBlock
  ): string {
    // Skip resolution for API block body content that looks like XML to prevent issues.
    if (
      currentBlock.metadata?.id === 'api' &&
      typeof value === 'string' &&
      (value.includes('<?xml') || value.includes('xmlns:') || value.includes('</')) &&
      value.includes('<') &&
      value.includes('>')
    ) {
      return value
    }

    // Extract all potential reference prefixes (e.g., `<blockId.`) from the string.
    const blockMatches = extractReferencePrefixes(value)
    if (blockMatches.length === 0) return value // No matches, return original string.

    // Get the set of block prefixes (IDs/normalized names) that are accessible from the current block.
    const accessiblePrefixes = this.getAccessiblePrefixes(currentBlock)

    let resolvedValue = value

    for (const match of blockMatches) {
      const { raw, prefix } = match // `raw` is the full `<prefix.path>` string, `prefix` is `blockId` or `blockName`.

      // If the prefix is not in the accessible set, skip this match.
      if (!accessiblePrefixes.has(prefix)) {
        continue
      }

      // Skip if this is a variable reference; it's handled by `resolveVariableReferences`.
      if (raw.startsWith('<variable.')) {
        continue
      }

      const path = raw.slice(1, -1) // Remove leading/trailing `< >`.
      const [blockRefToken, ...pathParts] = path.split('.') // Split into block reference and path parts.
      const blockRef = blockRefToken.trim()

      // Skip XML-like tags (e.g., `<some:tag>`) that might be mistaken for references.
      if (blockRef.includes(':')) {
        continue
      }

      // Determine if the current context is a JavaScript template literal.
      const isInTemplateLiteral =
        currentBlock.metadata?.id === 'function' &&
        value.includes('${') &&
        value.includes('}') &&
        value.includes('`')

      // --- Special Case: Trigger Block References (start, api, chat, manual) ---
      const blockRefLower = blockRef.toLowerCase()
      const triggerType =
        TRIGGER_REFERENCE_ALIAS_MAP[blockRefLower as keyof typeof TRIGGER_REFERENCE_ALIAS_MAP]
      if (triggerType) {
        // Find the actual trigger block based on its type.
        const triggerBlock = this.workflow.blocks.find(
          (block) => block.metadata?.id === triggerType
        )
        if (triggerBlock) {
          const blockState = context.blockStates.get(triggerBlock.id) // Get its execution state.
          if (blockState) {
            let replacementValue: any = blockState.output // Start with the block's output.

            // Traverse the path parts (e.g., `.input`, `.headers.Authorization`).
            for (const part of pathParts) {
              if (!replacementValue || typeof replacementValue !== 'object') {
                logger.warn(
                  `[resolveBlockReferences] Invalid path "${part}" - replacementValue is not an object:`,
                  replacementValue
                )
                throw new Error(`Invalid path "${part}" in "${path}" for trigger block.`)
              }

              // Handle array indexing like `files[0]` or `items[1]`.
              const arrayMatch = part.match(/^([^[]+)\[(\d+)\]$/)
              if (arrayMatch) {
                const [, arrayName, indexStr] = arrayMatch
                const index = Number.parseInt(indexStr, 10)

                const arrayValue = replacementValue[arrayName]
                if (!Array.isArray(arrayValue)) {
                  throw new Error(
                    `Property "${arrayName}" is not an array in path "${path}" for trigger block.`
                  )
                }

                if (index < 0 || index >= arrayValue.length) {
                  throw new Error(
                    `Array index ${index} is out of bounds for "${arrayName}" (length: ${arrayValue.length}) in path "${path}" for trigger block.`
                  )
                }

                replacementValue = arrayValue[index]
              } else if (/^(?:[^[]+(?:\[\d+\])+|(?:\[\d+\])+)$/.test(part)) {
                // Support multiple indices like `values[0][0]`.
                replacementValue = this.resolvePartWithIndices(
                  replacementValue,
                  part,
                  path,
                  'starter block'
                )
              } else {
                // Regular property access.
                replacementValue = resolvePropertyAccess(replacementValue, part)
              }

              if (replacementValue === undefined) {
                logger.warn(
                  `[resolveBlockReferences] No value found at path "${part}" in trigger block.`
                )
                throw new Error(`No value found at path "${path}" in trigger block.`)
              }
            }

            // Format the resolved value based on the *current block's* type and the path being referenced.
            let formattedValue: string

            // Special formatting for trigger *input* references (e.g., `<start.input>`, `<api.fieldName>`).
            const isTriggerInputRef =
              (blockRefLower === 'start' && pathParts.join('.').includes('input')) ||
              (blockRefLower === 'chat' && pathParts.join('.').includes('input')) ||
              (blockRefLower === 'api' && pathParts.length > 0)
            if (isTriggerInputRef) {
              const blockType = currentBlock.metadata?.id

              if (typeof replacementValue === 'object' && replacementValue !== null) {
                // For function blocks, stringify objects for code.
                if (blockType === 'function') {
                  formattedValue = JSON.stringify(replacementValue)
                }
                // For API blocks, stringify for body.
                else if (blockType === 'api') {
                  formattedValue = JSON.stringify(replacementValue)
                }
                // For condition blocks, use specific stringify logic.
                else if (blockType === 'condition') {
                  formattedValue = this.stringifyForCondition(replacementValue)
                }
                // For response blocks, preserve object structure (it will be JSON.stringified later).
                else if (blockType === 'response') {
                  formattedValue = replacementValue
                }
                // For all other blocks, stringify objects.
                else {
                  formattedValue = JSON.stringify(replacementValue)
                }
              } else {
                // For primitive values.
                if (blockType === 'function') {
                  formattedValue = this.formatValueForCodeContext(
                    replacementValue,
                    currentBlock,
                    isInTemplateLiteral
                  )
                } else if (blockType === 'condition') {
                  formattedValue = this.stringifyForCondition(replacementValue)
                } else if (blockType === 'response' && typeof replacementValue === 'string') {
                  // For response blocks, string values might need quoting
                  formattedValue = JSON.stringify(replacementValue)
                } else {
                  formattedValue = String(replacementValue)
                }
              }
            } else {
              // Standard handling for non-input references from trigger blocks.
              const blockType = currentBlock.metadata?.id
              if (blockType === 'response') {
                if (typeof replacementValue === 'string') {
                  formattedValue = JSON.stringify(replacementValue) // Properly escape for JSON.
                } else {
                  formattedValue = replacementValue
                }
              } else {
                formattedValue =
                  typeof replacementValue === 'object'
                    ? JSON.stringify(replacementValue)
                    : String(replacementValue)
              }
            }

            resolvedValue = resolvedValue.replace(raw, formattedValue) // Replace the placeholder.
            continue
          }
        }
      }

      // --- Special Case: "loop" References ---
      if (blockRef.toLowerCase() === 'loop') {
        const containingLoopId = this.loopsByBlockId.get(currentBlock.id) // Get the ID of the loop the current block is in.

        if (containingLoopId) {
          const formattedValue = this.resolveLoopReference(
            containingLoopId,
            pathParts,
            context,
            currentBlock,
            isInTemplateLiteral
          )

          if (formattedValue !== null) {
            resolvedValue = resolvedValue.replace(raw, formattedValue)
            continue
          }
        }
      }

      // --- Special Case: "parallel" References ---
      if (blockRef.toLowerCase() === 'parallel') {
        const containingParallelId = this.parallelsByBlockId.get(currentBlock.id) // Get the ID of the parallel the current block is in.

        if (containingParallelId) {
          const formattedValue = this.resolveParallelReference(
            containingParallelId,
            pathParts,
            context,
            currentBlock,
            isInTemplateLiteral
          )

          if (formattedValue !== null) {
            resolvedValue = resolvedValue.replace(raw, formattedValue)
            continue
          }
        }
      }

      // --- Standard Block Reference Resolution ---
      // Validate that the referenced block exists and is accessible.
      const validation = this.validateBlockReference(blockRef, currentBlock.id)

      if (!validation.isValid) {
        throw new Error(validation.errorMessage!)
      }

      const sourceBlock = this.blockById.get(validation.resolvedBlockId!)!

      // If the source block is disabled, throw an error.
      if (sourceBlock.enabled === false) {
        throw new Error(
          `Block "${sourceBlock.metadata?.name || sourceBlock.id}" is disabled, and block "${currentBlock.metadata?.name || currentBlock.id}" depends on it.`
        )
      }

      // Check if the source block was part of the active execution path.
      const isInActivePath = context.activeExecutionPath.has(sourceBlock.id)

      if (!isInActivePath) {
        // If not in the active path, it means it was skipped (e.g., by a condition block).
        // Replace the reference with an empty string.
        resolvedValue = resolvedValue.replace(raw, '')
        continue
      }

      // Retrieve the execution state of the source block.
      let blockState = context.blockStates.get(sourceBlock.id)

      // --- Parallel Execution Specific Handling ---
      // If the current block is executing within a virtual block (parallel iteration)
      // and the source block is *also* part of the same parallel,
      // try to retrieve the block state for that specific parallel iteration.
      if (
        context.currentVirtualBlockId &&
        context.parallelBlockMapping?.has(context.currentVirtualBlockId)
      ) {
        const currentParallelInfo = context.parallelBlockMapping.get(context.currentVirtualBlockId)
        if (currentParallelInfo) {
          const parallel = context.workflow?.parallels?.[currentParallelInfo.parallelId]
          if (parallel?.nodes.includes(sourceBlock.id)) {
            const virtualSourceBlockId = `${sourceBlock.id}_parallel_${currentParallelInfo.parallelId}_iteration_${currentParallelInfo.iterationIndex}`
            blockState = context.blockStates.get(virtualSourceBlockId)
          }
        }
      }

      if (!blockState) {
        // If no block state found:
        const isInLoop = this.loopsByBlockId.has(sourceBlock.id)
        if (isInLoop) {
          // If the referenced block is in a loop and no state is found, it means it hasn't run yet or finished.
          // Replace with an empty string.
          resolvedValue = resolvedValue.replace(raw, '')
          continue
        }

        // If not in a loop and not in active path (redundant check, but good for safety), replace with empty string.
        if (!context.activeExecutionPath.has(sourceBlock.id)) {
          resolvedValue = resolvedValue.replace(raw, '')
          continue
        }

        // Otherwise, it's an unexpected state (block should have run but has no state).
        throw new Error(
          `No state found for block "${sourceBlock.metadata?.name || sourceBlock.id}" (ID: ${sourceBlock.id}).`
        )
      }

      let replacementValue: any = blockState.output // Start with the source block's output.

      // Traverse the path parts (e.g., `.data.field`, `.items[0].value`).
      for (const part of pathParts) {
        if (!replacementValue || typeof replacementValue !== 'object') {
          throw new Error(
            `Invalid path "${part}" in "${path}" for block "${currentBlock.metadata?.name || currentBlock.id}".`
          )
        }

        // Handle array indexing syntax (`files[0]`).
        const arrayMatch = part.match(/^([^[]+)\[(\d+)\]$/)
        if (arrayMatch) {
          const [, arrayName, indexStr] = arrayMatch
          const index = Number.parseInt(indexStr, 10)

          const arrayValue = replacementValue[arrayName]
          if (!Array.isArray(arrayValue)) {
            throw new Error(
              `Property "${arrayName}" is not an array in path "${path}" for block "${sourceBlock.metadata?.name || sourceBlock.id}".`
            )
          }

          if (index < 0 || index >= arrayValue.length) {
            throw new Error(
              `Array index ${index} is out of bounds for "${arrayName}" (length: ${arrayValue.length}) in path "${path}" for block "${sourceBlock.metadata?.name || sourceBlock.id}".`
            )
          }

          replacementValue = arrayValue[index]
        } else if (/^(?:[^[]+(?:\[\d+\])+|(?:\[\d+\])+)$/.test(part)) {
          // Handle multiple indices like `values[0][0]`.
          replacementValue = this.resolvePartWithIndices(
            replacementValue,
            part,
            path,
            sourceBlock.metadata?.name || sourceBlock.id
          )
        } else {
          // Regular property access.
          replacementValue = resolvePropertyAccess(replacementValue, part)
        }

        if (replacementValue === undefined) {
          throw new Error(
            `No value found at path "${path}" in block "${sourceBlock.metadata?.name || sourceBlock.id}".`
          )
        }
      }

      let formattedValue: string

      // Format the value based on the *current block's* type where the reference is used.
      if (currentBlock.metadata?.id === 'condition') {
        formattedValue = this.stringifyForCondition(replacementValue) // Specific formatting for condition blocks.
      } else if (
        typeof replacementValue === 'string' &&
        this.needsCodeStringLiteral(currentBlock, value) // Check if code literal formatting is needed.
      ) {
        // For response blocks, strings are quoted for JSON.
        if (currentBlock.metadata?.id === 'response') {
          if (typeof replacementValue === 'string') {
            formattedValue = JSON.stringify(replacementValue)
          } else {
            formattedValue = replacementValue
          }
        } else {
          // For other code-like blocks (e.g., function), use `formatValueForCodeContext`.
          formattedValue = this.formatValueForCodeContext(
            replacementValue,
            currentBlock,
            isInTemplateLiteral
          )
        }
      } else {
        // Default formatting: stringify objects/arrays, convert primitives to string.
        formattedValue =
          typeof replacementValue === 'object'
            ? JSON.stringify(replacementValue)
            : String(replacementValue)
      }

      resolvedValue = resolvedValue.replace(raw, formattedValue) // Replace the placeholder.
    }

    return resolvedValue
  }

  /**
   * Resolves environment variable references (e.g., `{{ENV_VAR}}`) in a value.
   * This method recursively processes strings, arrays, and objects to find and replace
   * environment variable placeholders.
   *
   * @param value - The value (can be string, array, object) potentially containing environment variable references.
   * @returns The value with environment variables resolved.
   * @throws Error if a referenced environment variable is not found.
   */
  resolveEnvVariables(value: any): any {
    if (typeof value === 'string') {
      const envMatches = value.match(/\{\{([^}]+)\}\}/g) // Find all `{{KEY}}` patterns.
      if (envMatches) {
        let resolvedValue = value
        for (const match of envMatches) {
          const envKey = match.slice(2, -2) // Extract the key (e.g., "ENV_VAR").
          const envValue = this.environmentVariables[envKey] // Look up the value.

          if (envValue === undefined) {
            throw new Error(`Environment variable "${envKey}" was not found.`)
          }

          resolvedValue = resolvedValue.replace(match, envValue) // Replace the placeholder.
        }
        return resolvedValue
      }
      return value // No env variables found, return original string.
    }

    // Recursively process arrays.
    if (Array.isArray(value)) {
      return value.map((item) => this.resolveEnvVariables(item))
    }

    // Recursively process objects.
    if (value && typeof value === 'object') {
      return Object.entries(value).reduce(
        (acc, [k, v]) => ({
          ...acc,
          [k]: this.resolveEnvVariables(v),
        }),
        {}
      )
    }

    return value // Return primitives as is.
  }

  /**
   * A generalized recursive method to resolve all types of dynamic values
   * (workflow variables, block references, environment variables) within any nested structure
   * (strings, arrays, objects). It applies the resolution pipeline in order.
   *
   * @param value - The value to resolve (object, array, or primitive).
   * @param context - The current execution context.
   * @param currentBlock - The block making the reference.
   * @returns The fully resolved value.
   */
  private resolveNestedStructure(
    value: any,
    context: ExecutionContext,
    currentBlock: SerializedBlock
  ): any {
    // Handle null or undefined values directly.
    if (value === null || value === undefined) {
      return value
    }

    // For strings, apply the full resolution pipeline: variables -> blocks -> environment.
    if (typeof value === 'string') {
      const resolvedVars = this.resolveVariableReferences(value, currentBlock)
      const resolvedReferences = this.resolveBlockReferences(resolvedVars, context, currentBlock)
      return this.resolveEnvVariables(resolvedReferences)
    }

    // Recursively resolve items in arrays.
    if (Array.isArray(value)) {
      return value.map((item) => this.resolveNestedStructure(item, context, currentBlock))
    }

    // Recursively resolve properties in objects.
    if (typeof value === 'object') {
      const result: Record<string, any> = {}
      for (const [k, v] of Object.entries(value)) {
        result[k] = this.resolveNestedStructure(v, context, currentBlock)
      }
      return result
    }

    // Return primitive types as is.
    return value
  }

  /**
   * Formats a value specifically for use within condition blocks.
   * It ensures strings are properly quoted and escaped, and other types are converted
   * to a string representation suitable for direct comparison in condition expressions.
   *
   * @param value - The value to format.
   * @returns A string representation suitable for a condition expression.
   */
  private stringifyForCondition(value: any): string {
    if (typeof value === 'string') {
      // Escape special characters (backslashes, double quotes, newlines, carriage returns)
      // and wrap the string in double quotes.
      const sanitized = value
        .replace(/\\/g, '\\\\')
        .replace(/"/g, '\\"')
        .replace(/\n/g, '\\n')
        .replace(/\r/g, '\\r')
      return `"${sanitized}"`
    }
    if (value === null) {
      return 'null'
    }
    if (typeof value === 'undefined') {
      return 'undefined'
    }
    if (typeof value === 'object') {
      return JSON.stringify(value) // Stringify objects and arrays.
    }
    return String(value) // Convert numbers, booleans, etc., to string.
  }

  /**
   * Resolves a path part that might include multiple array indices,
   * such as `"values[0][0]"`. It iteratively accesses properties and array elements.
   *
   * @param base - The base object or array to start resolving from.
   * @param part - The path segment (e.g., "prop[0][1]" or "[0][0]").
   * @param fullPath - The complete reference path for error messages.
   * @param sourceName - The name of the source block for error messages.
   * @returns The resolved value at the specified path.
   * @throws Error if the path is invalid, a property is not found, or an index is out of bounds.
   */
  private resolvePartWithIndices(
    base: any,
    part: string,
    fullPath: string,
    sourceName: string
  ): any {
    let value = base

    // Extract leading property name if present (e.g., "values" from "values[0][0]").
    const propMatch = part.match(/^([^[]+)/)
    let rest = part
    if (propMatch) {
      const prop = propMatch[1]
      value = resolvePropertyAccess(value, prop)
      rest = part.slice(prop.length) // Remaining part with indices (e.g., "[0][0]").
      if (value === undefined) {
        throw new Error(`No value found at path "${fullPath}" in block "${sourceName}".`)
      }
    }

    // Iteratively apply each `[index]` part.
    const indexRe = /^\[(\d+)\]/ // Regex to match `[number]`.
    while (rest.length > 0) {
      const m = rest.match(indexRe)
      if (!m) {
        throw new Error(`Invalid path "${part}" in "${fullPath}" for block "${sourceName}".`)
      }
      const idx = Number.parseInt(m[1], 10) // Parse the index.
      if (!Array.isArray(value)) {
        throw new Error(`Invalid path "${part}" in "${fullPath}" for block "${sourceName}".`)
      }
      if (idx < 0 || idx >= value.length) {
        throw new Error(
          `Array index ${idx} is out of bounds in path "${fullPath}" for block "${sourceName}".`
        )
      }
      value = value[idx] // Access the array element.
      rest = rest.slice(m[0].length) // Remove the processed `[index]` part.
    }

    return value
  }

  /**
   * Normalizes a block name by converting it to lowercase and removing all whitespace.
   * This ensures consistent lookup regardless of casing or spacing in the original name.
   *
   * @param name - The block name to normalize.
   * @returns The normalized block name.
   */
  private normalizeBlockName(name: string): string {
    return name.toLowerCase().replace(/\s+/g, '')
  }

  /**
   * Helper method to find a workflow variable by its name.
   * It also normalizes variable names for matching.
   *
   * @param variableName - The name of the variable to find.
   * @returns The found variable object or `undefined` if not found.
   */
  private findVariableByName(variableName: string): any | undefined {
    // Iterate through workflowVariables and compare normalized names.
    const foundVariable = Object.entries(this.workflowVariables).find(
      ([_, variable]) => (variable.name || '').replace(/\s+/g, '') === variableName
    )

    return foundVariable ? foundVariable[1] : undefined
  }

  /**
   * Determines which blocks are accessible (can be referenced) from a given `currentBlockId`.
   * It prioritizes a pre-calculated `accessibleBlocksMap` for performance.
   * If not available, it falls back to a legacy calculation based on connections and special rules for loops/parallels.
   *
   * @param currentBlockId - The ID of the block making the reference.
   * @returns A Set of IDs of accessible blocks.
   */
  private getAccessibleBlocks(currentBlockId: string): Set<string> {
    // Use pre-calculated map if available.
    if (this.accessibleBlocksMap?.has(currentBlockId)) {
      return this.accessibleBlocksMap.get(currentBlockId)!
    }

    // Fallback to legacy calculation for compatibility.
    return this.calculateAccessibleBlocksLegacy(currentBlockId)
  }

  /**
   * Legacy method for calculating accessible blocks. This method is primarily for backward
   * compatibility and is used when the `accessibleBlocksMap` (which is typically pre-computed
   * during workflow compilation) is not provided.
   *
   * Rules:
   * 1. Blocks that have an outgoing connection *to* the current block.
   * 2. The 'starter' block is always accessible.
   * 3. All blocks within the same loop as the current block.
   * 4. All blocks within the same parallel as the current block.
   *
   * @param currentBlockId - The ID of the block requesting accessible references.
   * @returns A Set of IDs of accessible blocks based on legacy rules.
   */
  private calculateAccessibleBlocksLegacy(currentBlockId: string): Set<string> {
    const accessibleBlocks = new Set<string>()

    // Add blocks that directly connect TO the current block.
    for (const connection of this.workflow.connections) {
      if (connection.target === currentBlockId) {
        accessibleBlocks.add(connection.source)
      }
    }

    // Always allow referencing the starter block.
    const starterBlock = this.workflow.blocks.find((block) => block.metadata?.id === 'starter')
    if (starterBlock) {
      accessibleBlocks.add(starterBlock.id)
    }

    // If the current block is in a loop, all other blocks in that same loop are accessible.
    const currentBlockLoop = this.loopsByBlockId.get(currentBlockId)
    if (currentBlockLoop) {
      const loop = this.workflow.loops?.[currentBlockLoop]
      if (loop) {
        for (const nodeId of loop.nodes) {
          accessibleBlocks.add(nodeId)
        }
      }
    }

    // If the current block is in a parallel, all other blocks in that same parallel are accessible.
    const currentBlockParallel = this.parallelsByBlockId.get(currentBlockId)
    if (currentBlockParallel) {
      const parallel = this.workflow.parallels?.[currentBlockParallel]
      if (parallel) {
        for (const nodeId of parallel.nodes) {
          accessibleBlocks.add(nodeId)
        }
      }
    }

    return accessibleBlocks
  }

  /**
   * Retrieves user-friendly names of blocks that are accessible from a given block,
   * primarily for generating informative error messages.
   *
   * @param currentBlockId - The ID of the block making the reference.
   * @returns An array of unique, accessible block names.
   */
  private getAccessibleBlockNamesForError(currentBlockId: string): string[] {
    const accessibleBlockIds = this.getAccessibleBlocks(currentBlockId)
    const names: string[] = []

    for (const blockId of accessibleBlockIds) {
      const block = this.blockById.get(blockId)
      if (block?.metadata?.name) {
        names.push(block.metadata.name)
      }
    }

    return [...new Set(names)] // Remove duplicates.
  }

  /**
   * Validates if a block reference (`blockRef`) is valid and accessible from the `currentBlockId`.
   * It checks for block existence (by ID or normalized name) and connection-based access rules.
   *
   * @param blockRef - The name or ID of the block being referenced.
   * @param currentBlockId - The ID of the block making the reference.
   * @returns An object indicating `isValid`, the `resolvedBlockId` if valid, or an `errorMessage`.
   */
  private validateBlockReference(
    blockRef: string,
    currentBlockId: string
  ): { isValid: boolean; resolvedBlockId?: string; errorMessage?: string } {
    // Special case: 'start' alias for the starter block is always allowed.
    if (blockRef.toLowerCase() === 'start') {
      const starterBlock = this.workflow.blocks.find((block) => block.metadata?.id === 'starter')
      return starterBlock
        ? { isValid: true, resolvedBlockId: starterBlock.id }
        : { isValid: false, errorMessage: 'Starter block not found in workflow' }
    }

    // Try to find the source block by its exact ID.
    let sourceBlock = this.blockById.get(blockRef)
    if (!sourceBlock) {
      // If not found by ID, try by normalized name.
      const normalizedRef = this.normalizeBlockName(blockRef)
      sourceBlock = this.blockByNormalizedName.get(normalizedRef)

      // Fallback: if still not found, iterate all blocks and check their normalized names.
      if (!sourceBlock) {
        for (const candidate of this.workflow.blocks) {
          const candidateName = candidate.metadata?.name
          if (!candidateName) continue
          const normalizedName = this.normalizeBlockName(candidateName)
          if (normalizedName === normalizedRef) {
            sourceBlock = candidate
            break
          }
        }
      }
    }

    // If the block is not found by ID or name.
    if (!sourceBlock) {
      const accessibleNames = this.getAccessibleBlockNamesForError(currentBlockId)
      return {
        isValid: false,
        errorMessage: `Block "${blockRef}" was not found. Available connected blocks: ${accessibleNames.join(', ')}`,
      }
    }

    // Check if the found block is accessible from the current block according to rules.
    const accessibleBlocks = this.getAccessibleBlocks(currentBlockId)
    if (!accessibleBlocks.has(sourceBlock.id)) {
      const accessibleNames = this.getAccessibleBlockNamesForError(currentBlockId)
      return {
        isValid: false,
        errorMessage: `Block "${blockRef}" is not connected to this block. Available connected blocks: ${accessibleNames.join(', ')}`,
      }
    }

    // If all checks pass, the reference is valid.
    return { isValid: true, resolvedBlockId: sourceBlock.id }
  }

  /**
   * Retrieves the items that a for-each loop is iterating over.
   * It prioritizes items directly defined in `loop.forEachItems`, potentially
   * evaluating them as JSON or JavaScript expressions. As a fallback, it
   * searches for recent arrays or objects in previous block outputs.
   *
   * @param loop - The loop configuration object.
   * @param context - The current execution context.
   * @returns An array or object of items, or `null` if no valid items are found.
   */
  private getLoopItems(loop: any, context: ExecutionContext): any[] | Record<string, any> | null {
    if (!loop) return null

    // If `forEachItems` is already an array or object, return it directly.
    if (loop.forEachItems) {
      if (
        Array.isArray(loop.forEachItems) ||
        (typeof loop.forEachItems === 'object' && loop.forEachItems !== null)
      ) {
        return loop.forEachItems
      }

      // If `forEachItems` is a string, try to evaluate it.
      if (typeof loop.forEachItems === 'string') {
        const trimmedExpression = loop.forEachItems.trim()
        try {
          // 1. Try to parse as JSON (handles both standard JSON and JS-like object literals).
          if (trimmedExpression.startsWith('[') || trimmedExpression.startsWith('{')) {
            try {
              // Normalize single quotes and unquoted keys to double quotes for JSON.parse.
              const normalizedExpression = trimmedExpression
                .replace(/'/g, '"')
                .replace(/(\w+):/g, '"$1":')
                .replace(/,\s*]/g, ']')
                .replace(/,\s*}/g, '}')

              return JSON.parse(normalizedExpression)
            } catch (jsonError) {
              logger.error('Error parsing JSON for loop:', jsonError)
              // If JSON parsing fails, continue to expression evaluation.
            }
          }

          // 2. If not JSON or parsing failed, try to evaluate as a JavaScript expression.
          if (trimmedExpression && !trimmedExpression.startsWith('//')) {
            // `new Function` creates a function from a string, allowing dynamic evaluation.
            const result = new Function('context', `return ${loop.forEachItems}`)(context)
            if (Array.isArray(result) || (typeof result === 'object' && result !== null)) {
              return result
            }
          }
        } catch (e) {
          logger.error('Error evaluating forEach items:', e)
        }
      }
    }

    // Fallback: search for the most recent array or object in any block's output.
    // This is a less reliable heuristic but can catch some cases.
    for (const [_blockId, blockState] of context.blockStates.entries()) {
      const output = blockState.output
      if (output) {
        for (const [_key, value] of Object.entries(output)) {
          if (Array.isArray(value) && value.length > 0) {
            return value
          }
          if (typeof value === 'object' && value !== null && Object.keys(value).length > 0) {
            return value
          }
        }
      }
    }

    // Default to an empty array if no valid items are found.
    return []
  }

  /**
   * Formats a given value for safe insertion into a code context, particularly for 'function' blocks.
   * It ensures that strings are properly quoted (or left unquoted if inside a template literal),
   * objects/arrays are stringified, and other primitives are converted to their string representation.
   *
   * @param value - The value to format.
   * @param block - The block where this value will be used (context for formatting).
   * @param isInTemplateLiteral - True if the value is being inserted into a JavaScript template literal.
   * @returns The properly formatted string for code.
   */
  private formatValueForCodeContext(
    value: any,
    block: SerializedBlock,
    isInTemplateLiteral = false
  ): string {
    // This formatting is primarily for 'function' blocks.
    if (block.metadata?.id === 'function') {
      if (isInTemplateLiteral) {
        // Inside template literals (`...${value}...`), strings are *not* re-quoted.
        if (typeof value === 'string') {
          return value
        }
        // Objects/arrays still need to be stringified to avoid `[object Object]`.
        if (typeof value === 'object' && value !== null) {
          return JSON.stringify(value)
        }
        return String(value) // Numbers, booleans, etc., converted to string.
      }

      // Regular (non-template literal) code contexts:
      // ALL strings must be explicitly quoted for valid JavaScript syntax.
      if (typeof value === 'string') {
        return JSON.stringify(value) // `JSON.stringify` handles quoting and escaping.
      }
      if (typeof value === 'object' && value !== null) {
        return JSON.stringify(value) // Objects/arrays are stringified.
      }
      if (value === undefined) {
        return 'undefined' // Explicit JavaScript 'undefined'.
      }
      if (value === null) {
        return 'null' // Explicit JavaScript 'null'.
      }
      return String(value) // Numbers, booleans, etc., inserted as is.
    }

    // For non-code blocks, fallback to general string conversion.
    return typeof value === 'object' && value !== null ? JSON.stringify(value) : String(value)
  }

  /**
   * Determines if a value needs to be formatted as a code-compatible string literal.
   * This is crucial for blocks that execute code (e.g., 'function', 'condition')
   * to ensure syntax correctness when dynamic values are injected.
   *
   * @param block - The block where the value is being used.
   * @param expression - The full expression string (optional, used for contextual clues).
   * @returns True if the value should be treated as a code string literal, false otherwise.
   */
  private needsCodeStringLiteral(block?: SerializedBlock, expression?: string): boolean {
    if (!block) return false

    // Blocks that directly execute code.
    const codeExecutionBlocks = ['function', 'condition']

    if (block.metadata?.id && codeExecutionBlocks.includes(block.metadata.id)) {
      // 'function' blocks always require code-compatible string literal formatting.
      if (block.metadata.id === 'function') {
        return true
      }

      // For 'condition' blocks, `stringifyForCondition` handles the specific quoting,
      // so this method doesn't force general code literal formatting unless within a complex expression.
      if (block.metadata.id === 'condition' && !expression) {
        return false
      }
      return true
    }

    // Even in non-code blocks, if the expression itself looks like code (e.g., contains operators),
    // it might be a dynamic code snippet that requires string literal formatting.
    if (expression) {
      const codeIndicators = [
        /\(\s*$/, // Function call `myFunc(`
        /\.\w+\s*\(/, // Method call `obj.method(`
        /[=<>!+\-*/%](?:==?)?/, // Common operators (e.g., `=`, `==`, `>=`, `+`)
        /\+=|-=|\*=|\/=|%=|\*\*=?/, // Assignment operators
        /\b(if|else|for|while|return|var|let|const|function)\b/, // JavaScript keywords
        /\b(if|else|elif|for|while|def|return|import|from|as|class|with|try|except)\b/, // Python keywords
        /^['"]use strict['"]?$/, // JS strict mode
        /\$\{.+?\}/, // JS template literals
        /f['"].*?['"]/, // Python f-strings
        /\bprint\s*\(/, // Python print function
        /\bconsole\.\w+\(/, // JS console methods
      ]

      return codeIndicators.some((pattern) => pattern.test(expression))
    }

    return false
  }

  /**
   * Resolves a loop-specific reference (e.g., `<loop.currentItem>`, `<loop.index>`, `<loop.items>`).
   * It retrieves loop-related data from the execution context and formats it based on the
   * consuming block's requirements.
   *
   * @param loopId - The ID of the loop being referenced.
   * @param pathParts - The parts of the path after 'loop' (e.g., ['currentItem', 'key']).
   * @param context - The current execution context.
   * @param currentBlock - The block making the reference.
   * @param isInTemplateLiteral - Whether the reference is inside a template literal.
   * @returns The formatted value of the loop reference, or `null` if invalid.
   * @throws Error if the path is invalid or a nested property is not found.
   */
  private resolveLoopReference(
    loopId: string,
    pathParts: string[],
    context: ExecutionContext,
    currentBlock: SerializedBlock,
    isInTemplateLiteral: boolean
  ): string | null {
    const loop = context.workflow?.loops[loopId]
    if (!loop) return null

    const property = pathParts[0]

    switch (property) {
      case 'currentItem': {
        // Get the current item for this loop iteration from the context.
        const currentItem = context.loopItems.get(loopId)
        if (currentItem === undefined) {
          // If no item, the loop might not be active yet or already finished.
          return ''
        }

        // Handle nested path access (e.g., `<loop.currentItem.key>`).
        if (pathParts.length > 1) {
          // Special handling for `[key, value]` pairs from `Object.entries()` iteration.
          if (
            Array.isArray(currentItem) &&
            currentItem.length === 2 &&
            typeof currentItem[0] === 'string'
          ) {
            const subProperty = pathParts[1]
            if (subProperty === 'key') {
              return this.formatValueForCodeContext(
                currentItem[0],
                currentBlock,
                isInTemplateLiteral
              )
            }
            if (subProperty === 'value') {
              return this.formatValueForCodeContext(
                currentItem[1],
                currentBlock,
                isInTemplateLiteral
              )
            }
          }

          // Navigate nested object paths.
          let value = currentItem
          for (let i = 1; i < pathParts.length; i++) {
            if (!value || typeof value !== 'object') {
              throw new Error(`Invalid path "${pathParts[i]}" in loop item reference`)
            }
            value = (value as any)[pathParts[i] as any]
            if (value === undefined) {
              throw new Error(`No value found at path "loop.${pathParts.join('.')}" in loop item`)
            }
          }
          return this.formatValueForCodeContext(value, currentBlock, isInTemplateLiteral)
        }

        // If no nested path, return the whole current item.
        return this.formatValueForCodeContext(currentItem, currentBlock, isInTemplateLiteral)
      }

      case 'items': {
        // Valid only for 'forEach' loop types.
        if (loop.loopType !== 'forEach') {
          return null
        }

        // Get all items for the loop. Prioritize cached items in context.
        const items = context.loopItems.get(`${loopId}_items`) || this.getLoopItems(loop, context)
        if (!items) {
          return '[]'
        }

        return this.formatValueForCodeContext(items, currentBlock, isInTemplateLiteral)
      }

      case 'index': {
        // Get the current iteration index.
        const index = context.loopIterations.get(loopId) || 0
        // Adjust index (it's often incremented *after* the iteration starts).
        const adjustedIndex = Math.max(0, index - 1)
        return this.formatValueForCodeContext(adjustedIndex, currentBlock, isInTemplateLiteral)
      }

      default:
        return null // Unrecognized loop property.
    }
  }

  /**
   * Resolves a parallel-specific reference (e.g., `<parallel.currentItem>`, `<parallel.index>`, `<parallel.items>`).
   * This is similar to loop references but considers the concurrent nature of parallel execution
   * and potentially virtual block IDs for accurate item retrieval.
   *
   * @param parallelId - The ID of the parallel block being referenced.
   * @param pathParts - The parts of the path after 'parallel'.
   * @param context - The current execution context.
   * @param currentBlock - The block making the reference.
   * @param isInTemplateLiteral - Whether the reference is inside a template literal.
   * @returns The formatted value of the parallel reference, or `null` if invalid.
   * @throws Error if the path is invalid or a nested property is not found.
   */
  private resolveParallelReference(
    parallelId: string,
    pathParts: string[],
    context: ExecutionContext,
    currentBlock: SerializedBlock,
    isInTemplateLiteral: boolean
  ): string | null {
    const parallel = context.workflow?.parallels?.[parallelId]
    if (!parallel) return null

    const property = pathParts[0]

    switch (property) {
      case 'currentItem': {
        let currentItem = context.loopItems.get(parallelId) // Initial attempt to get current item.

        // If a virtual block ID is present (meaning we're in a specific parallel iteration),
        // use it to get the item for that exact iteration.
        if (context.currentVirtualBlockId && context.parallelBlockMapping) {
          const mapping = context.parallelBlockMapping.get(context.currentVirtualBlockId)
          if (mapping && mapping.parallelId === parallelId) {
            const iterationKey = `${parallelId}_iteration_${mapping.iterationIndex}`
            const iterationItem = context.loopItems.get(iterationKey)
            if (iterationItem !== undefined) {
              currentItem = iterationItem
            }
          }
        } else if (parallel.nodes.includes(currentBlock.id)) {
          // Fallback for older contexts or if `currentVirtualBlockId` isn't set,
          // try to find the item based on the current block's involvement in the parallel.
          for (const [_, mapping] of context.parallelBlockMapping || new Map()) {
            if (mapping.originalBlockId === currentBlock.id && mapping.parallelId === parallelId) {
              const iterationKey = `${parallelId}_iteration_${mapping.iterationIndex}`
              const iterationItem = context.loopItems.get(iterationKey)
              if (iterationItem !== undefined) {
                currentItem = iterationItem
                break
              }
            }
          }
        }

        // Additional fallback: check for parallel-specific keys like `parallelId_parallel_0`.
        if (currentItem === undefined) {
          for (let i = 0; i < 100; i++) {
            const parallelKey = `${parallelId}_parallel_${i}`
            if (context.loopItems.has(parallelKey)) {
              currentItem = context.loopItems.get(parallelKey)
              break
            }
          }
        }

        if (currentItem === undefined) {
          return '' // If no item found, return empty string.
        }

        // Handle nested path access (e.g., `<parallel.currentItem.key>`).
        if (pathParts.length > 1) {
          // Special handling for `[key, value]` pairs from `Object.entries()`.
          if (
            Array.isArray(currentItem) &&
            currentItem.length === 2 &&
            typeof currentItem[0] === 'string'
          ) {
            const subProperty = pathParts[1]
            if (subProperty === 'key') {
              return this.formatValueForCodeContext(
                currentItem[0],
                currentBlock,
                isInTemplateLiteral
              )
            }
            if (subProperty === 'value') {
              return this.formatValueForCodeContext(
                currentItem[1],
                currentBlock,
                isInTemplateLiteral
              )
            }
          }

          // Navigate nested object paths.
          let value = currentItem
          for (let i = 1; i < pathParts.length; i++) {
            if (!value || typeof value !== 'object') {
              throw new Error(`Invalid path "${pathParts[i]}" in parallel item reference`)
            }
            value = (value as any)[pathParts[i] as any]
            if (value === undefined) {
              throw new Error(
                `No value found at path "parallel.${pathParts.join('.')}" in parallel item`
              )
            }
          }
          return this.formatValueForCodeContext(value, currentBlock, isInTemplateLiteral)
        }

        // Return the whole current item.
        return this.formatValueForCodeContext(currentItem, currentBlock, isInTemplateLiteral)
      }

      case 'items': {
        // Get all items for the parallel distribution. Prioritize cached items.
        const items =
          context.loopItems.get(`${parallelId}_items`) ||
          (parallel.distribution && this.getParallelItems(parallel, context))
        if (!items) {
          return '[]'
        }

        return this.formatValueForCodeContext(items, currentBlock, isInTemplateLiteral)
      }

      case 'index': {
        let index = context.loopIterations.get(parallelId) // Initial attempt to get index.

        // If virtual block ID is present, get the index for that specific iteration.
        if (context.currentVirtualBlockId && context.parallelBlockMapping) {
          const mapping = context.parallelBlockMapping.get(context.currentVirtualBlockId)
          if (mapping && mapping.parallelId === parallelId) {
            index = mapping.iterationIndex
          }
        } else {
          // Fallback: check for parallel-specific keys like `parallelId_parallel_0`.
          if (index === undefined) {
            for (let i = 0; i < 100; i++) {
              const parallelKey = `${parallelId}_parallel_${i}`
              if (context.loopIterations.has(parallelKey)) {
                index = context.loopIterations.get(parallelKey)
                break
              }
            }
          }
        }

        const adjustedIndex = index !== undefined ? index : 0
        return this.formatValueForCodeContext(adjustedIndex, currentBlock, isInTemplateLiteral)
      }

      default:
        return null // Unrecognized parallel property.
    }
  }

  /**
   * Retrieves the items for a parallel distribution. Similar logic to `getLoopItems`,
   * but specifically for parallel blocks' `distribution` property.
   *
   * @param parallel - The parallel configuration object.
   * @param context - The current execution context.
   * @returns An array or object of items to be distributed, or `null`.
   */
  private getParallelItems(
    parallel: any,
    context: ExecutionContext
  ): any[] | Record<string, any> | null {
    if (!parallel || !parallel.distribution) return null

    // If `distribution` is already an array or object, return it.
    if (
      Array.isArray(parallel.distribution) ||
      (typeof parallel.distribution === 'object' && parallel.distribution !== null)
    ) {
      return parallel.distribution
    }

    // If `distribution` is a string, try to evaluate it as JSON or a JS expression.
    if (typeof parallel.distribution === 'string') {
      const trimmedExpression = parallel.distribution.trim()
      try {
        if (trimmedExpression.startsWith('[') || trimmedExpression.startsWith('{')) {
          try {
            return JSON.parse(trimmedExpression)
          } catch {
            // Continue with expression evaluation if JSON parsing fails.
          }
        }

        if (trimmedExpression && !trimmedExpression.startsWith('//')) {
          const result = new Function('context', `return ${parallel.distribution}`)(context)
          if (Array.isArray(result) || (typeof result === 'object' && result !== null)) {
            return result
          }
        }
      } catch (e) {
        logger.error('Error evaluating parallel distribution items:', e)
      }
    }

    return [] // Default to empty array.
  }

  /**
   * Processes a string value through the full resolution pipeline:
   * 1. Resolve workflow variable references (`<variable.name>`).
   * 2. Resolve block references (`<block.output.field>`).
   * 3. Resolve environment variable references (`{{ENV_VAR}}`).
   * Finally, attempts to parse the resolved string as JSON if it looks like JSON.
   *
   * @param value - The string value to process.
   * @param key - The input parameter key (for special handling like 'code' in function blocks).
   * @param context - The current execution context.
   * @param block - The block containing the value.
   * @returns The fully processed value, potentially parsed as JSON or kept as a string.
   */
  private processStringValue(
    value: string,
    key: string,
    context: ExecutionContext,
    block: SerializedBlock
  ): any {
    // 1. Resolve variable references.
    const resolvedVars = this.resolveVariableReferences(value, block)

    // 2. Resolve block references.
    const resolvedReferences = this.resolveBlockReferences(resolvedVars, context, block)

    // 3. Resolve environment variables.
    const resolvedEnv = this.resolveEnvVariables(resolvedReferences)

    const blockType = block.metadata?.id

    // Special handling: 'function' block's 'code' input should remain a raw string.
    if (blockType === 'function' && key === 'code') {
      return resolvedEnv
    }

    // Special handling: 'api' block's 'body' input might contain JSON, but also raw strings.
    if (blockType === 'api' && key === 'body') {
      return this.tryParseJSON(resolvedEnv)
    }

    // For all other inputs, attempt to parse the string as JSON.
    return this.tryParseJSON(resolvedEnv)
  }

  /**
   * Recursively processes object and array values. It includes special handling
   * for "table-like" arrays (common in API parameters/headers) where each item
   * has a 'cells' property.
   *
   * @param value - The object or array to process.
   * @param context - The current execution context.
   * @param block - The block containing the value.
   * @returns The processed object or array.
   */
  private processObjectValue(value: any, context: ExecutionContext, block: SerializedBlock): any {
    // Special handling for arrays that represent tables (e.g., API headers/params UI).
    // These arrays contain objects, each with a 'cells' property.
    if (
      Array.isArray(value) &&
      value.every((item) => typeof item === 'object' && item !== null && 'cells' in item)
    ) {
      return value.map((row) => ({
        ...row, // Keep other properties of the row.
        cells: Object.entries(row.cells).reduce(
          (acc, [cellKey, cellValue]) => {
            if (typeof cellValue === 'string') {
              const trimmedValue = cellValue.trim()
              // Check for direct variable reference in cells.
              const directVariableMatch = trimmedValue.match(/^<variable\.([^>]+)>$/)

              if (directVariableMatch) {
                const variableName = directVariableMatch[1]
                const variable = this.findVariableByName(variableName)

                if (variable) {
                  acc[cellKey] = this.getTypedVariableValue(variable)
                } else {
                  logger.warn(
                    `Variable reference <variable.${variableName}> not found in table cell`
                  )
                  acc[cellKey] = cellValue
                }
              } else {
                // If not a direct variable, process through the full nested resolution pipeline.
                acc[cellKey] = this.resolveNestedStructure(cellValue, context, block)
              }
            } else {
              // For non-string cell values, also resolve recursively.
              acc[cellKey] = this.resolveNestedStructure(cellValue, context, block)
            }
            return acc
          },
          {} as Record<string, any>
        ),
      }))
    }

    // For all other objects and arrays, use the general recursive resolution.
    return this.resolveNestedStructure(value, context, block)
  }

  /**
   * Attempts to parse a string as JSON if it adheres to common JSON structure.
   * This is a "safe" parsing function that won't throw an error if parsing fails;
   * instead, it returns the original string.
   *
   * @param value - The value to potentially parse.
   * @returns The parsed JSON object/array, or the original value if it's not a parsable JSON string.
   */
  private tryParseJSON(value: any): any {
    if (typeof value !== 'string') {
      return value
    }

    const trimmed = value.trim()
    // Check if the string starts with '{' (object) or '[' (array) to hint it might be JSON.
    if (trimmed.length > 0 && (trimmed.startsWith('{') || trimmed.startsWith('['))) {
      try {
        return JSON.parse(trimmed)
      } catch {
        // If JSON.parse throws an error, it's not valid JSON; fall through to return original value.
      }
    }

    return value // Return original value if not JSON-like or parsing failed.
  }

  /**
   * Public helper to get the ID of the loop that a specific block belongs to.
   *
   * @param blockId - The ID of the block.
   * @returns The containing loop ID or `undefined` if the block is not in a loop.
   */
  getContainingLoopId(blockId: string): string | undefined {
    return this.loopsByBlockId.get(blockId)
  }

  /**
   * Public helper to get the ID of the parallel block that a specific block belongs to.
   *
   * @param blockId - The ID of the block.
   * @returns The containing parallel ID or `undefined` if the block is not in a parallel.
   */
  getContainingParallelId(blockId: string): string | undefined {
    return this.parallelsByBlockId.get(blockId)
  }

  /**
   * Gathers all prefixes (block IDs and normalized names) that are accessible from the
   * current block, along with system-wide prefixes like 'loop' and 'parallel'.
   * This set is used in `resolveBlockReferences` for initial validation.
   *
   * @param block - The current block.
   * @returns A Set of accessible reference prefixes.
   */
  private getAccessiblePrefixes(block: SerializedBlock): Set<string> {
    const prefixes = new Set<string>()

    const accessibleBlocks = this.getAccessibleBlocks(block.id)
    accessibleBlocks.forEach((blockId) => {
      prefixes.add(normalizeBlockName(blockId)) // Add normalized ID.
      const sourceBlock = this.blockById.get(blockId)
      if (sourceBlock?.metadata?.name) {
        prefixes.add(normalizeBlockName(sourceBlock.metadata.name)) // Add normalized name.
      }
    })

    // Include system-defined prefixes (e.g., 'loop', 'parallel', 'start').
    SYSTEM_REFERENCE_PREFIXES.forEach((prefix) => prefixes.add(prefix))

    return prefixes
  }
}
```