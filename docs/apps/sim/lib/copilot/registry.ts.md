```typescript
import { z } from 'zod'

// Tool IDs supported by the new Copilot runtime
// This defines the possible values for `ToolId`, which represents the ID of a tool that the Copilot runtime can use.
// It uses `z.enum` from the Zod library to create a schema that only allows the specified string values.
export const ToolIds = z.enum([
  'get_user_workflow',
  'edit_workflow',
  'run_workflow',
  'get_workflow_console',
  'get_blocks_and_tools',
  'get_blocks_metadata',
  'get_trigger_examples',
  'get_examples_rag',
  'get_operations_examples',
  'search_documentation',
  'search_online',
  'make_api_request',
  'get_environment_variables',
  'set_environment_variables',
  'get_oauth_credentials',
  'gdrive_request_access',
  'list_gdrive_files',
  'read_gdrive_file',
  'reason',
  // New tools
  'list_user_workflows',
  'get_workflow_from_name',
  // New variable tools
  'get_global_workflow_variables',
  'set_global_workflow_variables',
  // New
  'oauth_request_access',
  'get_trigger_blocks',
])

// This line defines a TypeScript type `ToolId` based on the `ToolIds` Zod schema.
// `z.infer` extracts the TypeScript type that corresponds to the Zod schema.  In this case, it will be a union of string literal types, each representing a valid tool ID.
export type ToolId = z.infer<typeof ToolIds>

// Base SSE wrapper for tool_call events emitted by the LLM
// This defines a base schema for Server-Sent Events (SSE) that represent a tool call.
// SSE is a server push technology enabling real-time data streaming from a server to a client
// It uses `z.object` to define the structure of the SSE message, including the `type` (which is always 'tool_call'), and the `data` which includes `id`, `name`, `arguments`, and `partial`.
const ToolCallSSEBase = z.object({
  type: z.literal('tool_call'),
  data: z.object({
    id: z.string(),
    name: ToolIds,
    arguments: z.record(z.any()),
    partial: z.boolean().default(false),
  }),
})
export type ToolCallSSE = z.infer<typeof ToolCallSSEBase>

// Reusable small schemas
// Defines reusable schemas for common data types. This promotes consistency and reduces code duplication.
const StringArray = z.array(z.string())
const BooleanOptional = z.boolean().optional()
const NumberOptional = z.number().optional()

// Tool argument schemas (per SSE examples provided)
// This is the core of the file. It defines the expected structure of the `arguments` field for each tool, when emitted in a `tool_call` SSE.
// It's a `const` object, so it can't be modified after it's initialized.  This helps ensure type safety and prevent unexpected behavior.
export const ToolArgSchemas = {
  get_user_workflow: z.object({}),
  // New tools
  list_user_workflows: z.object({}),
  get_workflow_from_name: z.object({ workflow_name: z.string() }),
  // New variable tools
  get_global_workflow_variables: z.object({}),
  set_global_workflow_variables: z.object({
    operations: z.array(
      z.object({
        operation: z.enum(['add', 'delete', 'edit']),
        name: z.string(),
        type: z.enum(['plain', 'number', 'boolean', 'array', 'object']).optional(),
        value: z.string().optional(),
      })
    ),
  }),
  // New
  oauth_request_access: z.object({}),

  edit_workflow: z.object({
    operations: z
      .array(
        z.object({
          operation_type: z.enum(['add', 'edit', 'delete']),
          block_id: z.string(),
          params: z.record(z.any()).optional(),
        })
      )
      .min(1),
  }),

  run_workflow: z.object({
    workflow_input: z.string(),
  }),

  get_workflow_console: z.object({
    limit: NumberOptional,
    includeDetails: BooleanOptional,
  }),

  get_blocks_and_tools: z.object({}),

  get_blocks_metadata: z.object({
    blockIds: StringArray.min(1),
  }),

  get_trigger_blocks: z.object({}),

  get_block_best_practices: z.object({
    blockIds: StringArray.min(1),
  }),

  get_edit_workflow_examples: z.object({
    exampleIds: StringArray.min(1),
  }),

  get_trigger_examples: z.object({}),

  get_examples_rag: z.object({
    query: z.string(),
  }),

  get_operations_examples: z.object({
    query: z.string(),
  }),

  search_documentation: z.object({
    query: z.string(),
    topK: NumberOptional,
  }),

  search_online: z.object({
    query: z.string(),
    num: z.number().optional().default(10),
    type: z.enum(['search', 'news', 'places', 'images']).optional().default('search'),
    gl: z.string().optional(),
    hl: z.string().optional(),
  }),

  make_api_request: z.object({
    url: z.string(),
    method: z.enum(['GET', 'POST', 'PUT']),
    queryParams: z.record(z.union([z.string(), z.number(), z.boolean()])).optional(),
    headers: z.record(z.string()).optional(),
    body: z.union([z.record(z.any()), z.string()]).optional(),
  }),

  get_environment_variables: z.object({}),

  set_environment_variables: z.object({
    variables: z.record(z.string()),
  }),

  get_oauth_credentials: z.object({}),

  gdrive_request_access: z.object({}),

  list_gdrive_files: z.object({
    search_query: z.string().optional(),
    num_results: z.number().optional().default(50),
  }),

  read_gdrive_file: z.object({
    fileId: z.string(),
    type: z.enum(['doc', 'sheet']),
    range: z.string().optional(),
  }),

  reason: z.object({
    reasoning: z.string(),
  }),
} as const
export type ToolArgSchemaMap = typeof ToolArgSchemas

// Tool-specific SSE schemas (tool_call with typed arguments)
// This function dynamically creates a Zod schema for a specific tool's SSE message.
// It takes the tool's name (ToolId) and the schema for its arguments, and returns a new Zod schema that extends the `ToolCallSSEBase` to include the specific tool name and argument schema.
function toolCallSSEFor<TName extends ToolId, TArgs extends z.ZodTypeAny>(
  name: TName,
  argsSchema: TArgs
) {
  return ToolCallSSEBase.extend({
    data: ToolCallSSEBase.shape.data.extend({
      name: z.literal(name),
      arguments: argsSchema,
    }),
  })
}

// This object defines the specific SSE schemas for each tool.
// It uses the `toolCallSSEFor` function to create a schema for each tool, using the appropriate `ToolId` and `ToolArgSchemas`.
export const ToolSSESchemas = {
  get_user_workflow: toolCallSSEFor('get_user_workflow', ToolArgSchemas.get_user_workflow),
  // New tools
  list_user_workflows: toolCallSSEFor('list_user_workflows', ToolArgSchemas.list_user_workflows),
  get_workflow_from_name: toolCallSSEFor(
    'get_workflow_from_name',
    ToolArgSchemas.get_workflow_from_name
  ),
  // New variable tools
  get_global_workflow_variables: toolCallSSEFor(
    'get_global_workflow_variables',
    ToolArgSchemas.get_global_workflow_variables
  ),
  set_global_workflow_variables: toolCallSSEFor(
    'set_global_workflow_variables',
    ToolArgSchemas.set_global_workflow_variables
  ),
  edit_workflow: toolCallSSEFor('edit_workflow', ToolArgSchemas.edit_workflow),
  run_workflow: toolCallSSEFor('run_workflow', ToolArgSchemas.run_workflow),
  get_workflow_console: toolCallSSEFor('get_workflow_console', ToolArgSchemas.get_workflow_console),
  get_blocks_and_tools: toolCallSSEFor('get_blocks_and_tools', ToolArgSchemas.get_blocks_and_tools),
  get_blocks_metadata: toolCallSSEFor('get_blocks_metadata', ToolArgSchemas.get_blocks_metadata),
  get_trigger_blocks: toolCallSSEFor('get_trigger_blocks', ToolArgSchemas.get_trigger_blocks),

  get_trigger_examples: toolCallSSEFor('get_trigger_examples', ToolArgSchemas.get_trigger_examples),
  get_examples_rag: toolCallSSEFor('get_examples_rag', ToolArgSchemas.get_examples_rag),
  get_operations_examples: toolCallSSEFor(
    'get_operations_examples',
    ToolArgSchemas.get_operations_examples
  ),
  search_documentation: toolCallSSEFor('search_documentation', ToolArgSchemas.search_documentation),
  search_online: toolCallSSEFor('search_online', ToolArgSchemas.search_online),
  make_api_request: toolCallSSEFor('make_api_request', ToolArgSchemas.make_api_request),
  get_environment_variables: toolCallSSEFor(
    'get_environment_variables',
    ToolArgSchemas.get_environment_variables
  ),
  set_environment_variables: toolCallSSEFor(
    'set_environment_variables',
    ToolArgSchemas.set_environment_variables
  ),
  get_oauth_credentials: toolCallSSEFor(
    'get_oauth_credentials',
    ToolArgSchemas.get_oauth_credentials
  ),
  gdrive_request_access: toolCallSSEFor(
    'gdrive_request_access',
    ToolArgSchemas.gdrive_request_access as any
  ),
  list_gdrive_files: toolCallSSEFor('list_gdrive_files', ToolArgSchemas.list_gdrive_files),
  read_gdrive_file: toolCallSSEFor('read_gdrive_file', ToolArgSchemas.read_gdrive_file),
  reason: toolCallSSEFor('reason', ToolArgSchemas.reason),
  // New
  oauth_request_access: toolCallSSEFor('oauth_request_access', ToolArgSchemas.oauth_request_access),
} as const
export type ToolSSESchemaMap = typeof ToolSSESchemas

// Known result schemas per tool (what tool_result.result should conform to)
// Note: Where legacy variability exists, schema captures the common/expected shape for new runtime.
// This section defines the expected structure of the `result` field for each tool, when the tool returns a result.
// Similar to `ToolArgSchemas`, it's a `const` object, ensuring type safety.
const BuildOrEditWorkflowResult = z.object({
  yamlContent: z.string(),
  description: z.string().optional(),
  workflowState: z.unknown().optional(),
  data: z
    .object({
      blocksCount: z.number(),
      edgesCount: z.number(),
    })
    .optional(),
})

const ExecutionEntry = z.object({
  id: z.string(),
  executionId: z.string(),
  level: z.string(),
  trigger: z.string(),
  startedAt: z.string(),
  endedAt: z.string().nullable(),
  durationMs: z.number().nullable(),
  totalCost: z.number().nullable(),
  totalTokens: z.number().nullable(),
  blockExecutions: z.array(z.any()), // can be detailed per need
  output: z.any().optional(),
  errorMessage: z.string().optional(),
  errorBlock: z
    .object({
      blockId: z.string().optional(),
      blockName: z.string().optional(),
      blockType: z.string().optional(),
    })
    .optional(),
})

export const ToolResultSchemas = {
  get_user_workflow: z.object({ yamlContent: z.string() }).or(z.string()),
  // New tools
  list_user_workflows: z.object({ workflow_names: z.array(z.string()) }),
  get_workflow_from_name: z
    .object({ yamlContent: z.string() })
    .or(z.object({ userWorkflow: z.string() }))
    .or(z.string()),
  // New variable tools
  get_global_workflow_variables: z
    .object({ variables: z.record(z.any()) })
    .or(z.array(z.object({ name: z.string(), value: z.any() }))),
  set_global_workflow_variables: z
    .object({ variables: z.record(z.any()) })
    .or(z.object({ message: z.any().optional(), data: z.any().optional() })),
  // New
  oauth_request_access: z.object({
    granted: z.boolean().optional(),
    message: z.string().optional(),
  }),

  edit_workflow: BuildOrEditWorkflowResult,
  run_workflow: z.object({
    executionId: z.string().optional(),
    message: z.any().optional(),
    data: z.any().optional(),
  }),
  get_workflow_console: z.object({ entries: z.array(ExecutionEntry) }),
  get_blocks_and_tools: z.object({ blocks: z.array(z.any()), tools: z.array(z.any()) }),
  get_blocks_metadata: z.object({ metadata: z.record(z.any()) }),
  get_trigger_blocks: z.object({ triggerBlockIds: z.array(z.string()) }),
  get_block_best_practices: z.object({ bestPractices: z.array(z.any()) }),
  get_edit_workflow_examples: z.object({
    examples: z.array(
      z.object({
        id: z.string(),
        title: z.string().optional(),
        operations: z.array(z.any()).optional(),
      })
    ),
  }),
  get_trigger_examples: z.object({
    examples: z.array(
      z.object({
        id: z.string(),
        title: z.string().optional(),
        operations: z.array(z.any()).optional(),
      })
    ),
  }),
  get_examples_rag: z.object({
    examples: z.array(
      z.object({
        id: z.string(),
        title: z.string().optional(),
        operations: z.array(z.any()).optional(),
      })
    ),
  }),
  get_operations_examples: z.object({
    examples: z.array(
      z.object({
        id: z.string(),
        title: z.string().optional(),
        operations: z.array(z.any()).optional(),
      })
    ),
  }),
  search_documentation: z.object({ results: z.array(z.any()) }),
  search_online: z.object({ results: z.array(z.any()) }),
  make_api_request: z.object({
    status: z.number(),
    statusText: z.string().optional(),
    headers: z.record(z.string()).optional(),
    body: z.any().optional(),
  }),
  get_environment_variables: z.object({ variables: z.record(z.string()) }),
  set_environment_variables: z
    .object({ variables: z.record(z.string()) })
    .or(z.object({ message: z.any().optional(), data: z.any().optional() })),
  get_oauth_credentials: z.object({
    credentials: z.array(
      z.object({ id: z.string(), provider: z.string(), isDefault: z.boolean().optional() })
    ),
  }),
  gdrive_request_access: z.object({
    granted: z.boolean().optional(),
    message: z.string().optional(),
  }),
  list_gdrive_files: z.object({
    files: z.array(
      z.object({
        id: z.string(),
        name: z.string().optional(),
        mimeType: z.string().optional(),
        size: z.number().optional(),
      })
    ),
  }),
  read_gdrive_file: z.object({ content: z.string().optional(), data: z.any().optional() }),
  reason: z.object({ reasoning: z.string() }),
} as const
export type ToolResultSchemaMap = typeof ToolResultSchemas

// Consolidated registry entry per tool
// This creates a single, consolidated registry of all tools, including their argument schemas, SSE schemas, and result schemas.
// This makes it easier to access all the information about a tool in one place.
export const ToolRegistry = Object.freeze(
  (Object.keys(ToolArgSchemas) as ToolId[]).reduce(
    (acc, toolId) => {
      const args = (ToolArgSchemas as any)[toolId] as z.ZodTypeAny
      const sse = (ToolSSESchemas as any)[toolId] as z.ZodTypeAny
      const result = (ToolResultSchemas as any)[toolId] as z.ZodTypeAny
      acc[toolId] = { id: toolId, args, sse, result }
      return acc
    },
    {} as Record<
      ToolId,
      { id: ToolId; args: z.ZodTypeAny; sse: z.ZodTypeAny; result: z.ZodTypeAny }
    >
  )
)
export type ToolRegistryMap = typeof ToolRegistry

// Convenience helper types inferred from schemas
// These types provide a convenient way to infer the TypeScript types for the arguments, results, and SSE messages of a specific tool, based on the Zod schemas.
export type InferArgs<T extends ToolId> = z.infer<(typeof ToolArgSchemas)[T]>
export type InferResult<T extends ToolId> = z.infer<(typeof ToolResultSchemas)[T]>
export type InferToolCallSSE<T extends ToolId> = z.infer<(typeof ToolSSESchemas)[T]>
```

**Purpose of this file:**

This TypeScript file defines schemas and types for interacting with a Copilot runtime that utilizes various tools. It uses the Zod library for schema definition and validation. The primary purpose is to establish a contract for data exchange between the Copilot system and its tools, ensuring data integrity and type safety.  It specifies the structure of requests sent to tools (arguments), the structure of responses received from tools (results), and the structure of Server-Sent Events (SSE) used to communicate tool calls.

**Simplifying Complex Logic:**

The file uses the following techniques to simplify complex logic:

*   **Zod Schemas:** Zod is used to define schemas for data structures, which provides a declarative way to specify the expected format and types of data. This makes the code easier to read and understand compared to manual type checking.
*   **Reusability:** Common schemas like `StringArray`, `BooleanOptional`, and `NumberOptional` are defined once and reused throughout the file.
*   **`toolCallSSEFor` Function:** This function encapsulates the logic for creating SSE schemas for tools, reducing code duplication and making the code more maintainable.
*   **`ToolRegistry` Object:** This object centralizes all information about each tool, making it easier to access and manage tool-related data.
*   **Inferred Types:** The `InferArgs`, `InferResult`, and `InferToolCallSSE` types automatically derive TypeScript types from the Zod schemas, simplifying type definitions and reducing the risk of errors.
*   **`Object.freeze`:** The `ToolRegistry` is frozen to prevent accidental modifications, enhancing code reliability.

**Explanation of each line of code:**

The explanation is provided in the comments within the code.

**In summary:**

This file serves as a comprehensive data contract for the Copilot runtime and its tools, using Zod schemas and TypeScript types to ensure data integrity and type safety. It employs various techniques to simplify complex logic and promote code reusability and maintainability. By using these defined schemas, the Copilot runtime can reliably interact with various tools, knowing exactly what data to send and expect in return. This contributes to a robust and predictable system.
