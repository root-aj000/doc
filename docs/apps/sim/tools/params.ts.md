This TypeScript file is a utility module designed to manage and interpret configurations for various "tools" within an application. Its primary purpose is to provide a standardized way to define, retrieve, categorize, and validate parameters for these tools, especially considering both user interface (UI) presentation and interaction with Large Language Models (LLMs).

Essentially, it acts as a bridge between a tool's raw definition (what parameters it needs) and how it's presented to a user or described to an AI.

### Core Functionalities:

1.  **Parameter Definition:** Defines various data structures (interfaces) to represent tool parameters, their UI components, and how they are grouped.
2.  **Tool Parameter Retrieval:** Fetches a tool's parameters and augments them with UI configuration details, categorizing them by how they should be handled (user input, LLM input, required, optional).
3.  **Schema Generation:** Creates specialized schemas for tools, optimized for either an LLM's understanding (excluding user-provided values) or for direct tool execution.
4.  **Parameter Management:** Provides utilities for merging parameters, filtering schemas, and validating required inputs.
5.  **UI Helpers:** Includes functions to format parameter labels and identify potential password fields for better UI presentation.

---

### Code Explanation

Let's break down the code section by section.

#### Imports

```typescript
import { createLogger } from '@/lib/logs/console/logger'
import type { ParameterVisibility, ToolConfig } from '@/tools/types'
import { getTool } from '@/tools/utils'
```

*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports the `createLogger` function from a local logging utility. This will be used to create a logger instance for this module, enabling structured logging of warnings, errors, and informational messages.
*   `import type { ParameterVisibility, ToolConfig } from '@/tools/types'`: Imports TypeScript `type` definitions for `ParameterVisibility` and `ToolConfig`.
    *   `ToolConfig`: Likely defines the basic structure of a tool, including its ID, description, and raw parameters.
    *   `ParameterVisibility`: An enum or union type that specifies who (user, LLM, or both) should see or provide a specific parameter.
*   `import { getTool } from '@/tools/utils'`: Imports the `getTool` utility function. This function is presumably responsible for fetching a `ToolConfig` object given a tool ID.

#### Logger Initialization

```typescript
const logger = createLogger('ToolsParams')
```

*   `const logger = createLogger('ToolsParams')`: Initializes a logger instance specifically for this file, named 'ToolsParams'. This helps in tracing logs back to their source within the application.

#### Interfaces (Data Structures)

These interfaces define the shapes of various configuration objects and data structures used throughout the file. They are crucial for type-checking and understanding the data flow.

```typescript
export interface Option {
  label: string
  value: string
}
```

*   `export interface Option`: Defines a simple key-value pair, typically used for dropdowns, radio buttons, or select lists.
    *   `label`: The human-readable text displayed to the user.
    *   `value`: The actual value associated with the option.

```typescript
export interface ComponentCondition {
  field: string
  value: string
}
```

*   `export interface ComponentCondition`: Represents a condition that might control the visibility or behavior of a UI component.
    *   `field`: The name of the field (e.g., 'operation', 'provider') to check.
    *   `value`: The specific value the `field` must match for the condition to be true.

```typescript
export interface UIComponentConfig {
  type: string
  options?: Option[]
  placeholder?: string
  password?: boolean
  condition?: ComponentCondition
  title?: string
  layout?: string
  value?: unknown
  provider?: string
  serviceId?: string
  requiredScopes?: string[]
  mimeType?: string
  columns?: string[]
  min?: number
  max?: number
  step?: number
  integer?: boolean
  language?: string
  generationType?: string
  acceptedTypes?: string[]
  multiple?: boolean
  maxSize?: number
}
```

*   `export interface UIComponentConfig`: Defines the detailed configuration for a *single* user interface component (e.g., a text input, a dropdown, a file upload field). It includes properties that control its appearance and behavior. Many properties are optional (`?`) because not every component type will use all of them.
    *   `type`: The type of UI component (e.g., 'text-input', 'select', 'file-upload', 'checkbox').
    *   `options`: An array of `Option`s, used for select boxes or radio buttons.
    *   `placeholder`: Text displayed in an input field when it's empty.
    *   `password`: A boolean indicating if the input should be treated as a password field (obscuring input).
    *   `condition`: A `ComponentCondition` that dictates when this component should be shown or active.
    *   `title`: A display title for the component.
    *   `layout`: Specifies layout hints (e.g., 'full-width', 'inline').
    *   `value`: A default or initial value for the component.
    *   `provider`, `serviceId`, `requiredScopes`, `mimeType`, `columns`, `min`, `max`, `step`, `integer`, `language`, `generationType`, `acceptedTypes`, `multiple`, `maxSize`: Various specialized properties for different component types (e.g., file upload, number input, text area with specific language support, etc.).

```typescript
export interface SubBlockConfig {
  id: string
  type: string
  title?: string
  options?: Option[]
  placeholder?: string
  password?: boolean
  condition?: ComponentCondition
  layout?: string
  value?: unknown
  provider?: string
  serviceId?: string
  requiredScopes?: string[]
  mimeType?: string
  columns?: string[]
  min?: number
  max?: number
  step?: number
  integer?: boolean
  language?: string
  generationType?: string
  acceptedTypes?: string[]
  multiple?: boolean
  maxSize?: number
}
```

*   `export interface SubBlockConfig`: Represents a single configurable parameter or field *within* a larger UI block. It's essentially very similar to `UIComponentConfig` but includes an `id` to link it to a specific tool parameter. This is used when UI components are structured into logical groups.

```typescript
export interface BlockConfig {
  type: string
  subBlocks?: SubBlockConfig[]
}
```

*   `export interface BlockConfig`: Represents a larger UI block, which can contain multiple `SubBlockConfig`s. These blocks are often used to group related parameters or define complex UI layouts for a tool.
    *   `type`: The type of the block (e.g., 'basic-tool-form', 'advanced-settings').
    *   `subBlocks`: An array of `SubBlockConfig`s, defining the individual parameters/components within this block.

```typescript
export interface SchemaProperty {
  type: string
  description: string
}
```

*   `export interface SchemaProperty`: Describes a single property (parameter) within a tool's JSON schema, primarily for LLM interaction or validation.
    *   `type`: The data type of the property (e.g., 'string', 'number', 'boolean', 'object', 'array').
    *   `description`: A textual description of the property, helpful for LLMs or documentation.

```typescript
export interface ToolSchema {
  type: 'object'
  properties: Record<string, SchemaProperty>
  required: string[]
}
```

*   `export interface ToolSchema`: Defines a standard JSON schema structure for a tool's parameters. This is often used to communicate the tool's expected inputs to an LLM or for validation.
    *   `type: 'object'`: Indicates that the schema describes an object (a collection of properties).
    *   `properties`: A `Record` (an object where keys are strings) mapping parameter IDs to their `SchemaProperty` definitions.
    *   `required`: An array of strings, listing the IDs of parameters that *must* be provided.

```typescript
export interface ValidationResult {
  valid: boolean
  missingParams: string[]
}
```

*   `export interface ValidationResult`: The structure returned after validating tool parameters.
    *   `valid`: A boolean indicating whether all required parameters were provided.
    *   `missingParams`: An array of strings, listing the IDs of any required parameters that were not provided.

```typescript
export interface ToolParameterConfig {
  id: string
  type: string
  required?: boolean // Required for tool execution
  visibility?: ParameterVisibility // Controls who can/must provide this parameter
  userProvided?: boolean // User filled this parameter
  description?: string
  default?: unknown
  // UI component information from block config
  uiComponent?: UIComponentConfig
}
```

*   `export interface ToolParameterConfig`: This is a crucial interface, representing a single tool parameter in a standardized format, combining information from the `ToolConfig` with potential UI configuration from a `BlockConfig`.
    *   `id`: The unique identifier for the parameter.
    *   `type`: The data type of the parameter (e.g., 'string', 'number').
    *   `required`: A boolean indicating if this parameter is mandatory for the tool to function correctly.
    *   `visibility`: Determines whether the parameter is shown to the user, an LLM, both, or hidden.
    *   `userProvided`: A boolean flag (optional, can be added later) indicating if the user has already supplied a value for this parameter.
    *   `description`: A textual explanation of the parameter's purpose.
    *   `default`: A default value if none is explicitly provided.
    *   `uiComponent`: An optional `UIComponentConfig` object, containing details on how this parameter should be rendered in the UI.

```typescript
export interface ToolWithParameters {
  toolConfig: ToolConfig
  allParameters: ToolParameterConfig[]
  userInputParameters: ToolParameterConfig[] // Parameters shown to user
  requiredParameters: ToolParameterConfig[] // Must be filled by user or LLM
  optionalParameters: ToolParameterConfig[] // Nice to have, shown to user
}
```

*   `export interface ToolWithParameters`: This interface defines the comprehensive output of the `getToolParametersConfig` function, grouping a tool's parameters into useful categories.
    *   `toolConfig`: The original `ToolConfig` object.
    *   `allParameters`: A complete list of all parameters for the tool, each enhanced with `ToolParameterConfig` details.
    *   `userInputParameters`: A subset of `allParameters` that are intended to be shown to the user for input.
    *   `requiredParameters`: A subset of `allParameters` that are strictly necessary for the tool to execute.
    *   `optionalParameters`: A subset of `allParameters` that are not required but can be optionally provided by the user.

#### Global Cache for Block Configurations

```typescript
let blockConfigCache: Record<string, BlockConfig> | null = null
```

*   `let blockConfigCache: Record<string, BlockConfig> | null = null`: Declares a module-level variable to cache `BlockConfig` objects. It's initialized to `null`.
    *   `Record<string, BlockConfig>`: A TypeScript utility type indicating an object where keys are strings (typically block types) and values are `BlockConfig` objects.
    *   **Purpose**: Caching prevents repeated loading and parsing of block configurations, improving performance if `getBlockConfigurations` is called multiple times.

#### `getBlockConfigurations()` Function

```typescript
function getBlockConfigurations(): Record<string, BlockConfig> {
  if (!blockConfigCache) {
    try {
      const { getAllBlocks } = require('@/blocks')
      const allBlocks = getAllBlocks()
      blockConfigCache = {}
      allBlocks.forEach((block: BlockConfig) => {
        blockConfigCache![block.type] = block
      })
    } catch (error) {
      console.warn('Could not load block configuration:', error)
      blockConfigCache = {}
    }
  }
  return blockConfigCache
}
```

*   `function getBlockConfigurations(): Record<string, BlockConfig>`: This function is responsible for loading all UI block configurations.
*   `if (!blockConfigCache)`: Checks if the cache is empty. If it is, the configurations need to be loaded.
*   `try { ... } catch (error) { ... }`: A `try-catch` block handles potential errors during the loading process.
*   `const { getAllBlocks } = require('@/blocks')`: This line uses a dynamic `require` statement.
    *   **Explanation**: In a typical TypeScript/ESM setup, `import` is used. `require` is a CommonJS module loading function. This suggests that the `@/blocks` module might be a CommonJS module or that this project uses a build setup (like Webpack or Babel) that transpiles `require` or allows mixing module types. It dynamically loads a module that exports a `getAllBlocks` function.
    *   `getAllBlocks()`: This function, once imported, is called to retrieve all available `BlockConfig` objects.
*   `blockConfigCache = {}`: Initializes the cache as an empty object.
*   `allBlocks.forEach((block: BlockConfig) => { blockConfigCache![block.type] = block })`: Iterates through the loaded blocks and populates the `blockConfigCache`. Each block is stored using its `type` property as the key, allowing for quick lookup.
*   `catch (error)`: If `require` or `getAllBlocks()` fails (e.g., module not found, file read error), a warning is logged, and `blockConfigCache` is still initialized to an empty object to prevent further loading attempts.
*   `return blockConfigCache`: Returns the populated (or empty) `blockConfigCache`.

#### `getToolParametersConfig()` Function (Detailed Explanation)

This is the most central and complex function in the file. It aggregates tool definition parameters with UI component configurations to provide a comprehensive view of a tool's parameters.

```typescript
export function getToolParametersConfig(
  toolId: string,
  blockType?: string
): ToolWithParameters | null {
  try {
    const toolConfig = getTool(toolId)
    if (!toolConfig) {
      logger.warn(`Tool not found: ${toolId}`)
      return null
    }

    // Validate that toolConfig has required properties
    if (!toolConfig.params || typeof toolConfig.params !== 'object') {
      logger.warn(`Tool ${toolId} has invalid params configuration`)
      return null
    }

    // Get block configuration for UI component information
    let blockConfig: BlockConfig | null = null
    if (blockType) {
      const blockConfigs = getBlockConfigurations()
      blockConfig = blockConfigs[blockType] || null
    }

    // Convert tool params to our standard format with UI component info
    const allParameters: ToolParameterConfig[] = Object.entries(toolConfig.params).map(
      ([paramId, param]) => {
        const toolParam: ToolParameterConfig = {
          id: paramId,
          type: param.type,
          required: param.required ?? false,
          visibility: param.visibility ?? (param.required ? 'user-or-llm' : 'user-only'),
          description: param.description,
          default: param.default,
        }

        // Add UI component information from block config if available
        if (blockConfig) {
          // For multi-operation tools, find the subblock that matches both the parameter ID
          // and the current tool operation
          let subBlock = blockConfig.subBlocks?.find((sb: SubBlockConfig) => {
            if (sb.id !== paramId) return false

            // If there's a condition, check if it matches the current tool
            if (sb.condition && sb.condition.field === 'operation') {
              // First try exact match with full tool ID
              if (sb.condition.value === toolId) return true

              // Then try extracting operation from tool ID
              // For tools like 'google_calendar_quick_add', extract 'quick_add'
              const parts = toolId.split('_')
              if (parts.length >= 3) {
                // Join everything after the provider prefix (e.g., 'google_calendar_')
                const operation = parts.slice(2).join('_')
                if (sb.condition.value === operation) return true
              }

              // Fallback to last part only
              const operation = parts[parts.length - 1]
              return sb.condition.value === operation
            }

            // If no condition, it's a global parameter (like apiKey)
            return !sb.condition
          })

          // Fallback: if no operation-specific match, find any matching parameter
          if (!subBlock) {
            subBlock = blockConfig.subBlocks?.find((sb: SubBlockConfig) => sb.id === paramId)
          }

          // Special case: Check if this boolean parameter is part of a checkbox-list
          if (!subBlock && param.type === 'boolean' && blockConfig) {
            // Look for a checkbox-list that includes this parameter as an option
            const checkboxListBlock = blockConfig.subBlocks?.find(
              (sb: SubBlockConfig) =>
                sb.type === 'checkbox-list' &&
                Array.isArray(sb.options) &&
                sb.options.some((opt: any) => opt.id === paramId)
            )

            if (checkboxListBlock) {
              subBlock = checkboxListBlock
            }
          }

          if (subBlock) {
            toolParam.uiComponent = {
              type: subBlock.type,
              options: subBlock.options,
              placeholder: subBlock.placeholder,
              password: subBlock.password,
              condition: subBlock.condition,
              title: subBlock.title,
              layout: subBlock.layout,
              value: subBlock.value,
              provider: subBlock.provider,
              serviceId: subBlock.serviceId,
              requiredScopes: subBlock.requiredScopes,
              mimeType: subBlock.mimeType,
              columns: subBlock.columns,
              min: subBlock.min,
              max: subBlock.max,
              step: subBlock.step,
              integer: subBlock.integer,
              language: subBlock.language,
              generationType: subBlock.generationType,
              acceptedTypes: subBlock.acceptedTypes,
              multiple: subBlock.multiple,
              maxSize: subBlock.maxSize,
            }
          }
        }

        return toolParam
      }
    )

    // Parameters that should be shown to the user for input
    const userInputParameters = allParameters.filter(
      (param) => param.visibility === 'user-or-llm' || param.visibility === 'user-only'
    )

    // Parameters that are required (must be filled by user or LLM)
    const requiredParameters = allParameters.filter((param) => param.required)

    // Parameters that are optional but can be provided by user
    const optionalParameters = allParameters.filter(
      (param) => param.visibility === 'user-only' && !param.required
    )

    return {
      toolConfig,
      allParameters,
      userInputParameters,
      requiredParameters,
      optionalParameters,
    }
  } catch (error) {
    logger.error('Error getting tool parameters config:', error)
    return null
  }
}
```

*   `export function getToolParametersConfig(toolId: string, blockType?: string): ToolWithParameters | null`:
    *   Takes `toolId` (string) to identify the tool and an optional `blockType` (string) to specify a UI block configuration to use.
    *   Returns a `ToolWithParameters` object (containing categorized parameters) or `null` if an error occurs or the tool/config is not found.
*   `try { ... } catch (error) { ... }`: Wraps the entire function logic to catch and log any unexpected errors, returning `null` in such cases.
*   `const toolConfig = getTool(toolId)`: Fetches the base configuration for the tool using `getTool` from ` '@/tools/utils'`.
*   `if (!toolConfig)`: Checks if the tool was found. If not, logs a warning and returns `null`.
*   `if (!toolConfig.params || typeof toolConfig.params !== 'object')`: Validates that `toolConfig` has a valid `params` property, which should be an object containing parameter definitions. Logs a warning and returns `null` if invalid.
*   **Loading `blockConfig`**:
    *   `let blockConfig: BlockConfig | null = null`: Initializes a variable to hold the UI block configuration.
    *   `if (blockType)`: If a `blockType` is provided, it means there's a specific UI configuration linked to this tool or context.
    *   `const blockConfigs = getBlockConfigurations()`: Retrieves all cached block configurations.
    *   `blockConfig = blockConfigs[blockType] || null`: Attempts to find the specific `BlockConfig` using `blockType` as the key. If not found, `blockConfig` remains `null`.
*   **Mapping Tool Parameters (`allParameters` generation)**:
    *   `const allParameters: ToolParameterConfig[] = Object.entries(toolConfig.params).map(...)`: This is the core mapping logic. It iterates over each parameter defined in `toolConfig.params` and transforms it into the more comprehensive `ToolParameterConfig` format.
    *   `([paramId, param]) => { ... }`: For each parameter, it destructures the `paramId` (key) and `param` (value from `toolConfig.params`).
    *   `const toolParam: ToolParameterConfig = { ... }`: Creates the base `ToolParameterConfig` object for the current parameter, populating it with values from `param`.
        *   `required: param.required ?? false`: Sets `required` to `true` if `param.required` is defined and truthy, otherwise `false`. The `??` (nullish coalescing operator) handles `null` or `undefined`.
        *   `visibility: param.visibility ?? (param.required ? 'user-or-llm' : 'user-only')`: Sets `visibility`. If `param.visibility` is not explicitly set, it defaults to `'user-or-llm'` if the parameter is required, or `'user-only'` if optional. This ensures a default behavior for how parameters are exposed.
*   **Adding UI Component Information (Complex Logic Simplified)**:
    *   `if (blockConfig)`: This block executes only if a `blockConfig` was successfully loaded. It attempts to find a matching `SubBlockConfig` within the `blockConfig` to get UI details for the current `toolParam`.
    *   **Finding the `subBlock` (Operation-Specific Matching First)**:
        *   `let subBlock = blockConfig.subBlocks?.find((sb: SubBlockConfig) => { ... })`: It searches through the `subBlocks` array of the `blockConfig`.
        *   `if (sb.id !== paramId) return false`: A `subBlock` must at least match the `paramId`.
        *   `if (sb.condition && sb.condition.field === 'operation')`: This is crucial for multi-operation tools (e.g., a Google Calendar tool might have `add_event`, `list_events` operations). If a `subBlock` has a condition based on `operation`, it tries to match the `toolId` to that operation.
            *   `if (sb.condition.value === toolId) return true`: Checks for an exact match with the full `toolId`.
            *   **Operation Extraction Logic**: If no exact match, it tries to extract the "operation" part from the `toolId`. For example, for `google_calendar_quick_add`, it tries to match `quick_add` or just `add`.
                *   `const parts = toolId.split('_')`: Splits the `toolId` by underscores.
                *   `if (parts.length >= 3) { ... }`: For longer IDs, it assumes a `provider_service_operation` pattern and tries to match the `operation` by joining parts after the second underscore.
                *   `const operation = parts[parts.length - 1]`: As a fallback, it takes only the last part of the `toolId` as the operation.
            *   `return sb.condition.value === operation`: Checks if the extracted `operation` matches the `subBlock`'s condition value.
        *   `return !sb.condition`: If a `subBlock` has no condition, it's considered a "global" parameter (like an `apiKey`) that applies regardless of the specific operation, so it's a match.
    *   **Fallback Matching (Any `subBlock` for the parameter)**:
        *   `if (!subBlock) { subBlock = blockConfig.subBlocks?.find((sb: SubBlockConfig) => sb.id === paramId) }`: If no operation-specific `subBlock` was found, it tries to find *any* `subBlock` that simply matches the `paramId`, ignoring conditions.
    *   **Special Case: Boolean Parameters in Checkbox Lists**:
        *   `if (!subBlock && param.type === 'boolean' && blockConfig)`: If still no `subBlock` is found and the parameter is a boolean type, it specifically looks for `checkbox-list` blocks.
        *   `const checkboxListBlock = blockConfig.subBlocks?.find(...)`: It tries to find a `subBlock` of `type === 'checkbox-list'` which has an `options` array where one of the options has an `id` matching the current `paramId`. This handles cases where a boolean parameter isn't a standalone checkbox but rather an option within a grouped checkbox list.
        *   `if (checkboxListBlock) { subBlock = checkboxListBlock }`: If such a `checkboxListBlock` is found, it uses that as the `subBlock` for the parameter.
    *   **Assigning `uiComponent`**:
        *   `if (subBlock) { toolParam.uiComponent = { ... } }`: If a `subBlock` was found (through any of the matching strategies), its properties are copied to the `uiComponent` property of the `toolParam`. This transfers all the UI rendering hints.
*   **Categorizing Parameters**: After `allParameters` are constructed, they are filtered into more specific lists:
    *   `const userInputParameters = allParameters.filter(...)`: Filters for parameters where `visibility` is `user-or-llm` or `user-only`. These are the ones the user might interact with directly.
    *   `const requiredParameters = allParameters.filter(...)`: Filters for parameters where `required` is `true`. These are essential for tool execution.
    *   `const optionalParameters = allParameters.filter(...)`: Filters for parameters that are `user-only` (i.e., not for LLM) and *not* `required`. These are optional inputs for the user.
*   `return { ... }`: Returns the `ToolWithParameters` object, containing the original `toolConfig` and all the categorized lists of parameters.

#### `createLLMToolSchema()` Function

```typescript
export function createLLMToolSchema(
  toolConfig: ToolConfig,
  userProvidedParams: Record<string, unknown>
): ToolSchema {
  const schema: ToolSchema = {
    type: 'object',
    properties: {},
    required: [],
  }

  // Only include parameters that the LLM should/can provide
  Object.entries(toolConfig.params).forEach(([paramId, param]) => {
    const isUserProvided =
      userProvidedParams[paramId] !== undefined &&
      userProvidedParams[paramId] !== null &&
      userProvidedParams[paramId] !== ''

    // Skip parameters that user has already provided
    if (isUserProvided) {
      return
    }

    // Skip parameters that are user-only (never shown to LLM)
    if (param.visibility === 'user-only') {
      return
    }

    // Skip hidden parameters
    if (param.visibility === 'hidden') {
      return
    }

    // Add parameter to LLM schema
    let schemaType = param.type
    if (param.type === 'json' || param.type === 'any') {
      schemaType = 'object'
    }

    schema.properties[paramId] = {
      type: schemaType,
      description: param.description || '',
    }

    // Add to required if LLM must provide it and it's originally required
    if ((param.visibility === 'user-or-llm' || param.visibility === 'llm-only') && param.required) {
      schema.required.push(paramId)
    }
  })

  return schema
}
```

*   `export function createLLMToolSchema(toolConfig: ToolConfig, userProvidedParams: Record<string, unknown>): ToolSchema`: This function generates a `ToolSchema` specifically tailored for an LLM. It takes the `toolConfig` and an object `userProvidedParams` (containing values already filled by the user).
*   `const schema: ToolSchema = { ... }`: Initializes an empty `ToolSchema` object.
*   `Object.entries(toolConfig.params).forEach(...)`: Iterates over each original tool parameter.
*   `const isUserProvided = ...`: Checks if the current parameter `paramId` has been provided a non-empty, non-null, non-undefined value by the user.
*   **Filtering Logic for LLM Schema**:
    *   `if (isUserProvided) { return }`: If the user has *already* provided this parameter, the LLM doesn't need to generate it, so it's skipped.
    *   `if (param.visibility === 'user-only') { return }`: Parameters marked `user-only` are exclusively for user input and should never be exposed to the LLM.
    *   `if (param.visibility === 'hidden') { return }`: Parameters explicitly marked as `hidden` are also skipped.
*   **Adding to Schema**: For parameters that pass the filters:
    *   `let schemaType = param.type; if (param.type === 'json' || param.type === 'any') { schemaType = 'object' }`: Maps special types like `'json'` or `'any'` to `'object'` as a standard JSON schema type for LLMs.
    *   `schema.properties[paramId] = { ... }`: Adds the parameter to the `properties` of the schema with its type and description.
    *   `if ((param.visibility === 'user-or-llm' || param.visibility === 'llm-only') && param.required) { schema.required.push(paramId) }`: If the parameter is designated for LLM input (or user/LLM) *and* is originally required, it's added to the `required` array of the schema.
*   `return schema`: Returns the constructed LLM-specific tool schema.

#### `createExecutionToolSchema()` Function

```typescript
export function createExecutionToolSchema(toolConfig: ToolConfig): ToolSchema {
  const schema: ToolSchema = {
    type: 'object',
    properties: {},
    required: [],
  }

  Object.entries(toolConfig.params).forEach(([paramId, param]) => {
    schema.properties[paramId] = {
      type: param.type === 'json' ? 'object' : param.type,
      description: param.description || '',
    }

    if (param.required) {
      schema.required.push(paramId)
    }
  })

  return schema
}
```

*   `export function createExecutionToolSchema(toolConfig: ToolConfig): ToolSchema`: This function generates a `ToolSchema` that includes *all* parameters required for executing the tool, without any filtering for user-provided values or LLM visibility.
*   `const schema: ToolSchema = { ... }`: Initializes an empty `ToolSchema` object.
*   `Object.entries(toolConfig.params).forEach(...)`: Iterates through all original tool parameters.
*   `schema.properties[paramId] = { ... }`: Adds each parameter to the `properties` of the schema. `json` type is mapped to `object`.
*   `if (param.required) { schema.required.push(paramId) }`: If a parameter is marked as `required` in the `toolConfig`, its ID is added to the `required` array in the schema.
*   `return schema`: Returns the complete tool execution schema.

#### `mergeToolParameters()` Function

```typescript
export function mergeToolParameters(
  userProvidedParams: Record<string, unknown>,
  llmGeneratedParams: Record<string, unknown>
): Record<string, unknown> {
  // User-provided parameters take precedence
  return {
    ...llmGeneratedParams,
    ...userProvidedParams,
  }
}
```

*   `export function mergeToolParameters(...)`: Merges two sets of parameters: those provided by the user and those generated by an LLM.
*   `return { ...llmGeneratedParams, ...userProvidedParams, }`: Uses the spread syntax to create a new object. Parameters from `llmGeneratedParams` are spread first, then `userProvidedParams`. This ensures that if a parameter exists in both, the value from `userProvidedParams` *overwrites* the LLM-generated value, giving user input precedence.

#### `filterSchemaForLLM()` Function

```typescript
export function filterSchemaForLLM(
  originalSchema: ToolSchema,
  userProvidedParams: Record<string, unknown>
): ToolSchema {
  if (!originalSchema || !originalSchema.properties) {
    return originalSchema
  }

  const filteredProperties = { ...originalSchema.properties }
  const filteredRequired = [...(originalSchema.required || [])]

  // Remove user-provided parameters from the schema
  Object.keys(userProvidedParams).forEach((paramKey) => {
    if (
      userProvidedParams[paramKey] !== undefined &&
      userProvidedParams[paramKey] !== null &&
      userProvidedParams[paramKey] !== ''
    ) {
      delete filteredProperties[paramKey]
      const reqIndex = filteredRequired.indexOf(paramKey)
      if (reqIndex > -1) {
        filteredRequired.splice(reqIndex, 1)
      }
    }
  })

  return {
    ...originalSchema,
    properties: filteredProperties,
    required: filteredRequired,
  }
}
```

*   `export function filterSchemaForLLM(...)`: This function takes an `originalSchema` (likely the execution schema or a previously generated LLM schema) and a set of `userProvidedParams`. It returns a *new* schema where any parameters that the user has already provided are removed. This is useful for dynamically adjusting the schema presented to an LLM, so it doesn't try to generate values for things already covered.
*   `if (!originalSchema || !originalSchema.properties) { return originalSchema }`: Handles edge cases where the schema is invalid or empty.
*   `const filteredProperties = { ...originalSchema.properties }`: Creates a shallow copy of the `properties` object to avoid modifying the original schema.
*   `const filteredRequired = [...(originalSchema.required || [])]`: Creates a shallow copy of the `required` array.
*   `Object.keys(userProvidedParams).forEach((paramKey) => { ... })`: Iterates through the keys of the `userProvidedParams`.
*   `if (userProvidedParams[paramKey] !== undefined && ... )`: Checks if the user-provided value for `paramKey` is meaningful (not undefined, null, or empty string).
*   `delete filteredProperties[paramKey]`: If the parameter was provided by the user, it's removed from the `properties` object.
*   `const reqIndex = filteredRequired.indexOf(paramKey); if (reqIndex > -1) { filteredRequired.splice(reqIndex, 1) }`: If the parameter was also in the `required` list, it's removed from there.
*   `return { ...originalSchema, properties: filteredProperties, required: filteredRequired, }`: Returns a new `ToolSchema` object with the filtered `properties` and `required` lists.

#### `validateToolParameters()` Function

```typescript
export function validateToolParameters(
  toolConfig: ToolConfig,
  finalParams: Record<string, unknown>
): ValidationResult {
  const requiredParams = Object.entries(toolConfig.params)
    .filter(([_, param]) => param.required)
    .map(([paramId]) => paramId)

  const missingParams = requiredParams.filter(
    (paramId) =>
      finalParams[paramId] === undefined ||
      finalParams[paramId] === null ||
      finalParams[paramId] === ''
  )

  return {
    valid: missingParams.length === 0,
    missingParams,
  }
}
```

*   `export function validateToolParameters(...)`: Checks if all required parameters for a tool have been provided in a given set of `finalParams`.
*   `const requiredParams = Object.entries(toolConfig.params) .filter(...) .map(...)`: First, it extracts all `paramId`s from `toolConfig.params` that are marked as `required: true`.
*   `const missingParams = requiredParams.filter(...)`: Then, it filters this `requiredParams` list to find any parameters that are `undefined`, `null`, or an empty string in the `finalParams` object.
*   `return { valid: missingParams.length === 0, missingParams, }`: Returns a `ValidationResult` object, indicating `valid: true` if no required parameters are missing, along with a list of any `missingParams`.

#### `isPasswordParameter()` Function

```typescript
export function isPasswordParameter(paramId: string): boolean {
  const passwordFields = [
    'password',
    'apiKey',
    'token',
    'secret',
    'key',
    'credential',
    'accessToken',
    'refreshToken',
    'botToken',
    'authToken',
  ]

  return passwordFields.some((field) => paramId.toLowerCase().includes(field.toLowerCase()))
}
```

*   `export function isPasswordParameter(paramId: string): boolean`: A utility function that heuristically determines if a parameter ID likely represents a sensitive password or API key field.
*   `const passwordFields = [...]`: Defines a list of common keywords associated with sensitive information.
*   `return passwordFields.some((field) => paramId.toLowerCase().includes(field.toLowerCase()))`: Checks if the `paramId` (converted to lowercase) includes any of the keywords from the `passwordFields` list (also converted to lowercase). Returns `true` if any match is found, `false` otherwise. This helps UIs to obscure such inputs.

#### `formatParameterLabel()` Function

```typescript
export function formatParameterLabel(paramId: string): string {
  // Special cases
  if (paramId === 'apiKey') return 'API Key'
  if (paramId === 'apiVersion') return 'API Version'
  if (paramId === 'accessToken') return 'Access Token'
  if (paramId === 'refreshToken') return 'Refresh Token'
  if (paramId === 'botToken') return 'Bot Token'

  // Handle underscore and hyphen separated words
  if (paramId.includes('_') || paramId.includes('-')) {
    return paramId
      .split(/[-_]/)
      .map((word) => word.charAt(0).toUpperCase() + word.slice(1))
      .join(' ')
  }

  // Handle single character parameters
  if (paramId.length === 1) return paramId.toUpperCase()

  // Handle camelCase
  if (/[A-Z]/.test(paramId)) {
    const result = paramId.replace(/([A-Z])/g, ' $1')
    return (
      result.charAt(0).toUpperCase() +
      result
        .slice(1)
        .replace(/ Api/g, ' API')
        .replace(/ Id/g, ' ID')
        .replace(/ Url/g, ' URL')
        .replace(/ Uri/g, ' URI')
        .replace(/ Ui/g, ' UI')
    )
  }

  // Simple case - just capitalize first letter
  return paramId.charAt(0).toUpperCase() + paramId.slice(1)
}
```

*   `export function formatParameterLabel(paramId: string): string`: This function takes a technical `paramId` (e.g., `userEmail`, `api_key`, `min-price`) and converts it into a more human-readable label for UI display (e.g., `User Email`, `API Key`, `Min Price`).
*   **Special Cases**:
    *   `if (paramId === 'apiKey') return 'API Key'`: Handles specific IDs that have common, preferred capitalizations (e.g., 'API Key' instead of 'Api Key').
*   **Underscore/Hyphen Separated Words**:
    *   `if (paramId.includes('_') || paramId.includes('-')) { ... }`: If the ID contains underscores or hyphens, it splits the string by these delimiters.
    *   `.map((word) => word.charAt(0).toUpperCase() + word.slice(1))`: Capitalizes the first letter of each resulting word.
    *   `.join(' ')`: Joins them back together with spaces (e.g., `min_price` -> `Min Price`).
*   **Single Character Parameters**:
    *   `if (paramId.length === 1) return paramId.toUpperCase()`: For single-letter IDs, simply capitalizes it (e.g., `q` -> `Q`).
*   **CamelCase**:
    *   `if (/[A-Z]/.test(paramId)) { ... }`: If the ID contains uppercase letters (indicating camelCase), it processes it.
    *   `const result = paramId.replace(/([A-Z])/g, ' $1')`: Inserts a space before each uppercase letter (e.g., `userEmail` -> `user Email`).
    *   `result.charAt(0).toUpperCase() + result.slice(1).replace(...)`: Capitalizes the very first letter of the result, then performs additional replacements for common acronyms (e.g., `Api` -> `API`, `Id` -> `ID`, `Url` -> `URL`). This handles `user Email` becoming `User Email` and `userId` becoming `User ID`.
*   **Simple Capitalization**:
    *   `return paramId.charAt(0).toUpperCase() + paramId.slice(1)`: As a fallback for all other cases (e.g., `email`), it simply capitalizes the first letter.

---

This file is a comprehensive utility for managing tool parameters, centralizing the logic for how tools are configured, presented, and interacted with by both humans and AI.