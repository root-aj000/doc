```typescript
/**
 * Copilot Types - Consolidated from various locations
 * This file contains all copilot-related type definitions
 */

// Tool call state types (from apps/sim/types/tool-call.ts)
// ======================================================
// These interfaces define the structure for tracking the state of individual tool calls.

/**
 * `ToolCallState`: Represents the state of a single tool call within the copilot system.
 *   It encapsulates information about the tool's execution, status, and results.
 */
export interface ToolCallState {
  /** A unique identifier for this specific tool call. */
  id: string;
  /** The programmatic name of the tool being called (e.g., "search_api"). */
  name: string;
  /** A user-friendly name for the tool to be displayed in the UI (e.g., "Search"). */
  displayName?: string;
  /**  Key-value pairs representing the input parameters for the tool.  The `any` type is used for flexibility but could be made more specific with a type or interface.*/
  parameters?: Record<string, any>;
  /**
   * The current state of the tool call.  Represents the lifecycle of a tool's execution.
   *   - `detecting`: The tool call is being identified.
   *   - `pending`: The tool call is waiting to be executed.
   *   - `executing`: The tool is currently running.
   *   - `completed`: The tool has finished successfully.
   *   - `error`: The tool execution resulted in an error.
   *   - `rejected`: The tool call was rejected by the user or system.
   *   - `applied`: The result of the tool call has been applied.
   *   - `ready_for_review`:  The tool call is completed and waiting for review (e.g., user confirmation).
   *   - `aborted`: The tool call was stopped prematurely.
   *   - `skipped`: The tool call was intentionally skipped.
   *   - `background`: The tool is running in the background.
   */
  state:
    | 'detecting'
    | 'pending'
    | 'executing'
    | 'completed'
    | 'error'
    | 'rejected'
    | 'applied'
    | 'ready_for_review'
    | 'aborted'
    | 'skipped'
    | 'background';
  /** The timestamp (in milliseconds) when the tool call started. */
  startTime?: number;
  /** The timestamp (in milliseconds) when the tool call finished. */
  endTime?: number;
  /** The duration of the tool call in milliseconds. */
  duration?: number;
  /** The result data returned by the tool.  The `any` type is used, but a more specific type would be beneficial. */
  result?: any;
  /** An error message, if the tool call failed. */
  error?: string;
  /** A string representing the progress of the tool. */
  progress?: string;
}

/**
 * `ToolCallGroup`: Represents a group of related `ToolCallState` objects.  This is useful for managing tool calls that are executed sequentially or in parallel as part of a larger task.
 */
export interface ToolCallGroup {
  /** A unique identifier for the group of tool calls. */
  id: string;
  /** An array of `ToolCallState` objects that belong to this group. */
  toolCalls: ToolCallState[];
  /**
   * The overall status of the tool call group.
   *   - `pending`:  At least one tool call is pending.
   *   - `in_progress`: At least one tool call is executing.
   *   - `completed`: All tool calls in the group have completed successfully.
   *   - `error`: At least one tool call in the group has resulted in an error.
   */
  status: 'pending' | 'in_progress' | 'completed' | 'error';
  /** The timestamp when the tool call group started. */
  startTime?: number;
  /** The timestamp when the tool call group finished. */
  endTime?: number;
  /** A summary of the tool call group's execution. */
  summary?: string;
}

/**
 * `InlineContent`: Represents a piece of content that can be either text or a tool call.
 *   This is used for structuring messages that contain both text and references to tool calls.  This supports interleaving normal content with tool calls.
 */
export interface InlineContent {
  /** The type of content: either 'text' or 'tool_call'. */
  type: 'text' | 'tool_call';
  /** The actual content of the inline element (e.g., the text or a serialized representation of the tool call). */
  content: string;
  /** The `ToolCallState` associated with this inline content, if it's a tool call. */
  toolCall?: ToolCallState;
}

/**
 * `ParsedMessageContent`:  Represents the structure of a message that has been parsed to extract text content, tool calls, and tool groups. This is likely used after processing a response from an LLM that might include tool call requests embedded within the text.
 */
export interface ParsedMessageContent {
  /** The plain text content of the message. */
  textContent: string;
  /** An array of `ToolCallState` objects extracted from the message. */
  toolCalls: ToolCallState[];
  /** An array of `ToolCallGroup` objects extracted from the message. */
  toolGroups: ToolCallGroup[];
  /** An array of `InlineContent` objects, representing the interleaved text and tool calls in the message. */
  inlineContent?: InlineContent[];
}

import type { ProviderId } from '@/providers/types';

// Copilot Tools Type Definitions (from workspace copilot lib)
// ==========================================================
// These types define the structure for copilot tools, their metadata, and execution.
import type { CopilotToolCall, ToolState } from '@/stores/copilot/types';

/**
 * `NotificationStatus`: Defines the possible statuses for notifications related to tool calls.
 *   These statuses are used to provide feedback to the user about the progress and outcome of tool executions.
 */
export type NotificationStatus =
  | 'pending'
  | 'success'
  | 'error'
  | 'accepted'
  | 'rejected'
  | 'background';

// Export the consolidated types
export type { CopilotToolCall, ToolState };

/**
 * `StateDisplayConfig`: Defines how a specific `ToolState` should be displayed in the UI.
 */
export interface StateDisplayConfig {
  /** The user-friendly name for the state. */
  displayName: string;
  /** An optional icon to display for the state. */
  icon?: string;
  /** An optional CSS class to apply to the state's display. */
  className?: string;
}

/**
 * `ToolDisplayConfig`: Defines the overall display configuration for a tool, including the display settings for each possible state.
 */
export interface ToolDisplayConfig {
  /** A mapping of `ToolState` to `StateDisplayConfig`, defining how each state should be displayed. */
  states: {
    [K in ToolState]?: StateDisplayConfig;
  };
  /** An optional function that can dynamically generate the display name for a tool based on its current state and parameters.  This allows for more context-aware display names. */
  getDynamicDisplayName?: (
    state: ToolState,
    params: Record<string, any>
  ) => string | null;
}

/**
 * `ToolSchema`: Defines the schema for a tool, including its name, description, and parameters (using an OpenAI function calling format). This schema is crucial for LLMs to understand how to call the tool correctly.
 */
export interface ToolSchema {
  /** The programmatic name of the tool. */
  name: string;
  /** A description of what the tool does. */
  description: string;
  /**
   * The parameters that the tool accepts, in a format similar to OpenAI function calling.
   *   - `type`: Always 'object' for function calling.
   *   - `properties`:  A record of property names to their schemas
   *   - `required`: An array of property names that are required.
   */
  parameters?: {
    type: 'object';
    properties: Record<string, any>;
    required?: string[];
  };
}

/**
 * `ToolMetadata`: Contains static configuration information about a tool, such as its display settings, schema, and interrupt behavior.
 */
export interface ToolMetadata {
  /** A unique identifier for the tool. */
  id: string;
  /** The display configuration for the tool. */
  displayConfig: ToolDisplayConfig;
  /** The schema for the tool's parameters. */
  schema: ToolSchema;
  /** Indicates whether the tool requires an interrupt (e.g., user confirmation) before execution. */
  requiresInterrupt: boolean;
  /** Indicates whether the tool can be executed in the background. */
  allowBackgroundExecution?: boolean;
  /** A mapping of `NotificationStatus` to messages, allowing for customized messages based on the status of the tool execution. */
  stateMessages?: Partial<Record<NotificationStatus, string>>;
}

/**
 * `ToolExecuteResult`: Defines the result of executing a tool.
 */
export interface ToolExecuteResult {
  /** Indicates whether the tool execution was successful. */
  success: boolean;
  /** The data returned by the tool, if successful. The `any` type is used, but a more specific type would be beneficial. */
  data?: any;
  /** An error message, if the tool execution failed. */
  error?: string;
}

/**
 * `ToolConfirmResponse`: Defines the response from a confirmation API, used when a tool requires user confirmation before execution.
 */
export interface ToolConfirmResponse {
  /** Indicates whether the confirmation was successful. */
  success: boolean;
  /** An optional message to display to the user. */
  message?: string;
}

/**
 * `ToolExecutionOptions`: Defines the options that can be passed to a tool's `execute` method.
 */
export interface ToolExecutionOptions {
  /** A callback function that is called when the tool's state changes. */
  onStateChange?: (state: ToolState) => void;
  /** A callback function that is called before the tool is executed.  It should return a promise that resolves to a boolean indicating whether execution should proceed. */
  beforeExecute?: () => Promise<boolean>;
  /** A callback function that is called after the tool has been executed. */
  afterExecute?: (result: ToolExecuteResult) => Promise<void>;
  /** A context object that can be passed to the tool.  This allows the tool to access external data or services. */
  context?: Record<string, any>;
}

/**
 * `Tool`:  The main interface that all copilot tools must implement. It defines the methods that are required for a tool to be integrated into the copilot system.
 */
export interface Tool {
  /** The metadata for the tool. */
  metadata: ToolMetadata;
  /** Executes the tool with the given parameters and options. */
  execute(
    toolCall: CopilotToolCall,
    options?: ToolExecutionOptions
  ): Promise<ToolExecuteResult>;
  /** Returns the display name for the tool, based on the given parameters. */
  getDisplayName(toolCall: CopilotToolCall): string;
  /** Returns the icon for the tool, based on the given parameters. */
  getIcon(toolCall: CopilotToolCall): string;
  /** Handles a user action (e.g., run, skip, background) for the tool. */
  handleUserAction(
    toolCall: CopilotToolCall,
    action: 'run' | 'skip' | 'background',
    options?: ToolExecutionOptions
  ): Promise<void>;
  /** Indicates whether the tool requires confirmation from the user before execution. */
  requiresConfirmation(toolCall: CopilotToolCall): boolean;
}

// Provider configuration for Sim Agent requests
// ===============================================
// These types define the configuration options for different LLM providers.
// This type is only for the `provider` field in requests sent to the Sim Agent
/**
 * `CopilotProviderConfig`: Defines the configuration for a specific LLM provider used by the copilot.
 *   This is used when making requests to the Sim Agent.
 */
export type CopilotProviderConfig =
  | {
      /** Specifies that the Azure OpenAI provider is being used. */
      provider: 'azure-openai';
      /** The name of the model to use (e.g., "gpt-4"). */
      model: string;
      /** The API key for accessing the Azure OpenAI service. */
      apiKey?: string;
      /** The API version to use (e.g., "2023-05-15"). */
      apiVersion?: string;
      /** The endpoint URL for the Azure OpenAI service. */
      endpoint?: string;
    }
  | {
      /** Specifies that a provider other than Azure OpenAI is being used. The `Exclude` type ensures that 'azure-openai' is not a possible value here. */
      provider: Exclude<ProviderId, 'azure-openai'>;
      /** The name of the model to use (e.g., "gpt-3.5-turbo"). */
      model?: string;
      /** The API key for accessing the LLM provider. */
      apiKey?: string;
    };

```

**Purpose of this file:**

This TypeScript file acts as a central repository for all type definitions related to the Copilot system.  It consolidates types from different parts of the codebase (e.g., `apps/sim/types/tool-call.ts`, workspace copilot lib, and `src/providers/types`) to provide a single, consistent source of truth for copilot-related data structures. This promotes code reusability, maintainability, and reduces the risk of type inconsistencies across different modules.  The file covers tool execution, state management, display configurations, and provider configurations.

**Simplifying Complex Logic:**

While the code consists primarily of type definitions, it implicitly simplifies logic by:

*   **Encapsulation:**  Types like `ToolCallState` and `ToolMetadata` encapsulate complex data structures, making it easier to work with them in other parts of the code.
*   **Abstraction:** The `Tool` interface provides an abstraction over the different types of tools that can be used by the copilot.  This allows the core copilot logic to interact with tools in a generic way, without needing to know the specifics of each tool.
*   **Standardization:** By defining a common set of types, the file ensures that all copilot-related components use the same data structures, reducing the potential for errors and inconsistencies.

**Line-by-line Explanation:**

(See the code block above with detailed comments)

**Key Concepts and Relationships:**

*   **Tools and Tool Calls:**  A `Tool` represents an action the Copilot can take (e.g., searching the web, sending an email). A `ToolCall` is a specific *instance* of a tool being invoked with particular parameters.  The `ToolCallState` tracks the progress of this specific invocation.
*   **State Management:**  The `ToolState` and `ToolCallState` enums and interfaces are critical for managing the lifecycle of tool executions, allowing the UI and other components to provide feedback and handle errors appropriately.
*   **Display Configuration:**  The `ToolDisplayConfig` and `StateDisplayConfig` interfaces control how tools and their states are presented to the user, allowing for customization and a consistent user experience.
*   **Provider Abstraction:** `CopilotProviderConfig` allows the Copilot to work with different LLM providers (e.g., Azure OpenAI, OpenAI) by providing a standardized configuration format.
*   **Schema and Metadata:**  The `ToolSchema` and `ToolMetadata` interfaces provide information about a tool's capabilities and requirements, enabling the Copilot to reason about which tools to use and how to call them. The `ToolSchema` aligns with the OpenAI function calling format which is key for LLMs understanding the tool.

In summary, this file provides a well-structured and comprehensive set of type definitions for the Copilot system.  It promotes code clarity, consistency, and maintainability by centralizing and standardizing the data structures used throughout the codebase.
