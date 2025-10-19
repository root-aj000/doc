This TypeScript file acts as a **central definition hub** for "Tools" within an application, likely an AI agent system, an automation platform, or a microservice orchestration layer. Think of these "Tools" as external functions or APIs that your application (or an AI) can call to perform specific actions, like fetching data, creating records, or executing a business process.

The primary goal of this file is to define the **schema and configuration contracts** for these tools, ensuring they are well-defined, discoverable, and can be integrated consistently.

---

### Simplified Explanation of Core Concepts

At its heart, this file defines:

1.  **`ToolConfig`**: The blueprint for *any* tool. It describes:
    *   **What it is:** ID, name, description.
    *   **What it needs:** `params` (input arguments) with their types, descriptions, and who provides them (user, LLM).
    *   **What it gives back:** `outputs` (the structure of its results).
    *   **How it works:** `request` (how to make the API call), `oauth` (if it needs authentication).
    *   **Advanced logic:** `postProcess` (for chaining actions) and `transformResponse` (for formatting results).
2.  **`ToolResponse`**: A standardized format for the *outcome* of running any tool, including success status, output data, and potential errors.
3.  **Helper Types**: Definitions for common concepts like HTTP methods, authentication settings, and specific data structures like files or table rows.

---

### Detailed Code Explanation

Let's break down each part of the file:

#### 1. Imports

```typescript
import type { OAuthService } from '@/lib/oauth/oauth'
```

*   **`import type { OAuthService } from '@/lib/oauth/oauth'`**: This line imports a type definition named `OAuthService` from another file in your project (`@/lib/oauth/oauth`). The `import type` syntax ensures that `OAuthService` is only used for type checking and won't generate any JavaScript code, keeping the bundle smaller. It's expected to be an enumeration or a union type representing different OAuth providers (e.g., 'Google', 'Slack', 'GitHub').

#### 2. HTTP Method Definition

```typescript
export type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH' | 'HEAD'
```

*   **`export type HttpMethod = ...`**: This defines a custom type called `HttpMethod`. It's a **union type**, meaning a variable of type `HttpMethod` can only hold one of the specified string literal values: `'GET'`, `'POST'`, `'PUT'`, `'DELETE'`, `'PATCH'`, or `'HEAD'`. These are the standard HTTP request methods.
    *   **Purpose:** Provides type safety and clarity when specifying the HTTP method for a tool's API request.

#### 3. Output Property Schema

```typescript
export interface OutputProperty {
  type: string
  description?: string
  optional?: boolean
  properties?: Record<string, OutputProperty>
  items?: {
    type: string
    description?: string
    properties?: Record<string, OutputProperty>
  }
}
```

*   **`export interface OutputProperty { ... }`**: This interface describes the schema for a **single property** that a tool might output. It's designed to be recursive, meaning it can describe complex, nested data structures.
    *   **`type: string`**: The data type of the property (e.g., `'string'`, `'number'`, `'boolean'`, `'object'`, `'array'`).
    *   **`description?: string`**: An optional human-readable description of the property. The `?` makes it optional.
    *   **`optional?: boolean`**: An optional flag indicating whether this output property might be absent in the tool's response.
    *   **`properties?: Record<string, OutputProperty>`**: If the `type` of this `OutputProperty` is `'object'`, this property describes its nested fields. `Record<string, OutputProperty>` means it's an object where keys are property names (strings) and values are themselves `OutputProperty` definitions. This is how nesting is achieved.
    *   **`items?: { ... }`**: If the `type` is `'array'`, this property describes the elements contained within the array.
        *   **`type: string`**: The type of each item in the array.
        *   **`description?: string`**: Description for array items.
        *   **`properties?: Record<string, OutputProperty>`**: If the array items are objects, this describes their structure, similar to the main `properties` field.
    *   **Purpose:** To define a standardized way to describe the expected structure of a tool's output, allowing for validation, documentation, and dynamic UI generation.

#### 4. Parameter Visibility Control

```typescript
export type ParameterVisibility =
  | 'user-or-llm' // User can provide OR LLM must generate
  | 'user-only' // Only user can provide (required/optional determined by required field)
  | 'llm-only' // Only LLM provides (computed values)
  | 'hidden' // Not shown to user or LLM
```

*   **`export type ParameterVisibility = ...`**: This union type defines who or what is responsible for providing a parameter when a tool is invoked.
    *   **`'user-or-llm'`**: The parameter can either be explicitly provided by a human user or automatically generated by a Large Language Model (LLM). This offers flexibility.
    *   **`'user-only'`**: This parameter *must* come directly from the human user. The LLM should not attempt to generate it. Its requirement (optional/required) is determined by other fields.
    *   **`'llm-only'`**: This parameter is expected to be *only* generated by the LLM. It's often for computed or internal values that the user shouldn't see or interact with.
    *   **`'hidden'`**: The parameter is not exposed to either the user or the LLM. It might be an internal parameter set by the system itself.
    *   **Purpose:** To control the user experience and the LLM's interaction with tool parameters, ensuring the right values are provided by the right source.

#### 5. Standard Tool Response Structure

```typescript
export interface ToolResponse {
  success: boolean // Whether the tool execution was successful
  output: Record<string, any> // The structured output from the tool
  error?: string // Error message if success is false
  timing?: {
    startTime: string // ISO timestamp when the tool execution started
    endTime: string // ISO timestamp when the tool execution ended
    duration: number // Duration in milliseconds
  }
}
```

*   **`export interface ToolResponse { ... }`**: This interface defines the standard format for the result of *any* tool execution.
    *   **`success: boolean`**: A boolean flag indicating whether the tool's operation completed successfully.
    *   **`output: Record<string, any>`**: The main data returned by the tool. It's an object (dictionary) where keys are output property names and values are their corresponding data. `any` is used here for maximum flexibility, as the specific structure is defined in `ToolConfig['outputs']`.
    *   **`error?: string`**: An optional string that provides an error message if `success` is `false`.
    *   **`timing?: { ... }`**: An optional object for performance metrics.
        *   **`startTime: string`**: An ISO 8601 formatted timestamp indicating when the tool execution began.
        *   **`endTime: string`**: An ISO 8601 formatted timestamp indicating when the tool execution ended.
        *   **`duration: number`**: The total duration of the execution in milliseconds.
    *   **Purpose:** Provides a consistent way for all tools to report their outcomes, making it easier to process results, handle errors, and monitor performance.

#### 6. OAuth Configuration

```typescript
export interface OAuthConfig {
  required: boolean // Whether this tool requires OAuth authentication
  provider: OAuthService // The service that needs to be authorized
  additionalScopes?: string[] // Additional scopes required for the tool
}
```

*   **`export interface OAuthConfig { ... }`**: This interface defines the OAuth (Open Authorization) authentication requirements for a tool.
    *   **`required: boolean`**: If `true`, this tool *must* be authenticated via OAuth to function.
    *   **`provider: OAuthService`**: Specifies which OAuth service (e.g., Google, Slack, etc., as defined by the imported `OAuthService` type) is needed for authorization.
    *   **`additionalScopes?: string[]`**: An optional array of strings, specifying any extra permissions (scopes) beyond the basic ones that the tool requires from the OAuth provider.
    *   **Purpose:** To clearly indicate if a tool needs OAuth, which provider it uses, and what permissions it requests, enabling the system to manage user authorizations correctly.

#### 7. The Core: Tool Configuration

```typescript
export interface ToolConfig<P = any, R = any> {
  // Basic tool identification
  id: string
  name: string
  description: string
  version: string

  // Parameter schema - what this tool accepts
  params: Record<
    string,
    {
      type: string
      required?: boolean
      visibility?: ParameterVisibility
      default?: any
      description?: string
    }
  >

  // Output schema - what this tool produces
  outputs?: Record<
    string,
    {
      type: 'string' | 'number' | 'boolean' | 'json' | 'file' | 'file[]' | 'array' | 'object'
      description?: string
      optional?: boolean
      fileConfig?: {
        mimeType?: string // Expected MIME type for file outputs
        extension?: string // Expected file extension
      }
      items?: {
        type: string
        properties?: Record<string, OutputProperty>
      }
      properties?: Record<string, OutputProperty>
    }
  >

  // OAuth configuration for this tool (if it requires authentication)
  oauth?: OAuthConfig

  // Request configuration
  request: {
    url: string | ((params: P) => string)
    method: HttpMethod | ((params: P) => HttpMethod)
    headers: (params: P) => Record<string, string>
    body?: (params: P) => Record<string, any>
  }

  // Post-processing (optional) - allows additional processing after the initial request
  postProcess?: (
    result: R extends ToolResponse ? R : ToolResponse,
    params: P,
    executeTool: (toolId: string, params: Record<string, any>) => Promise<ToolResponse>
  ) => Promise<R extends ToolResponse ? R : ToolResponse>

  // Response handling
  transformResponse?: (response: Response, params?: P) => Promise<R>
}
```

*   **`export interface ToolConfig<P = any, R = any> { ... }`**: This is the most comprehensive interface, defining the complete configuration for a single tool.
    *   **`<P = any, R = any>`**: These are **generics**.
        *   `P` (for Parameters): Represents the specific type of the input parameters object for *this particular tool*. By default, it's `any`. Using a specific type here (e.g., `ToolConfig<{ query: string }, MyCustomResponse>`) allows for strong type checking of the `params` object in functions and callback signatures.
        *   `R` (for Response): Represents the specific type of the output object for *this particular tool*. By default, it's `any`, but it's often a type derived from `ToolResponse` or `ToolResponse` itself.

    *   **Basic tool identification:**
        *   **`id: string`**: A unique identifier for the tool (e.g., `'google-search'`, `'salesforce-create-lead'`).
        *   **`name: string`**: A human-readable name for the tool (e.g., `'Google Search'`, `'Create Salesforce Lead'`).
        *   **`description: string`**: A detailed explanation of what the tool does, useful for documentation or for an LLM to understand its purpose.
        *   **`version: string`**: The version of this tool definition (e.g., `'1.0.0'`).

    *   **`params: Record<string, { ... }>`**: Defines the input parameters the tool accepts. It's an object where keys are parameter names (strings).
        *   **`type: string`**: The data type of the parameter (e.g., `'string'`, `'number'`, `'boolean'`).
        *   **`required?: boolean`**: If `true`, this parameter *must* be provided.
        *   **`visibility?: ParameterVisibility`**: Uses the `ParameterVisibility` type to control who provides this parameter.
        *   **`default?: any`**: An optional default value if the parameter is not explicitly provided.
        *   **`description?: string`**: A description of the parameter.

    *   **`outputs?: Record<string, { ... }> `**: Defines the structured output that the tool produces. It's an optional object where keys are output property names. This is more specific than the generic `OutputProperty` as it has a defined set of basic types and specific configurations for files.
        *   **`type: 'string' | 'number' | 'boolean' | 'json' | 'file' | 'file[]' | 'array' | 'object'`**: A specific union of common types, including `'file'` and `'file[]'` for file-based outputs.
        *   **`description?: string`**: Description of the output property.
        *   **`optional?: boolean`**: If `true`, this output property might be missing.
        *   **`fileConfig?: { ... }`**: An optional configuration specific to file outputs (`'file'` or `'file[]'`).
            *   **`mimeType?: string`**: Expected MIME type of the file (e.g., `'application/pdf'`, `'image/jpeg'`).
            *   **`extension?: string`**: Expected file extension (e.g., `'pdf'`, `'jpg'`).
        *   **`items?: { ... }`**: If the `type` is `'array'` or `'file[]'`, this describes the elements within the array. It uses a simplified `OutputProperty`-like structure.
        *   **`properties?: Record<string, OutputProperty>`**: If the `type` is `'object'`, this describes its nested fields using the `OutputProperty` interface.

    *   **`oauth?: OAuthConfig`**: An optional property that links to the `OAuthConfig` interface, specifying if this tool requires OAuth authentication.

    *   **`request: { ... }`**: Defines how the tool makes its underlying HTTP request to an external API.
        *   **`url: string | ((params: P) => string)`**: The URL to call. It can be a static string or a function that dynamically generates the URL based on the tool's input `params`.
        *   **`method: HttpMethod | ((params: P) => HttpMethod)`**: The HTTP method to use. Can be a static `HttpMethod` or a function that dynamically determines the method.
        *   **`headers: (params: P) => Record<string, string>`**: A function that returns an object of HTTP headers to send with the request. It *must* be a function to allow dynamic headers (e.g., injecting authentication tokens).
        *   **`body?: (params: P) => Record<string, any>`**: An optional function that returns the request body for methods like `POST`, `PUT`, or `PATCH`.

    *   **`postProcess?: (result: R extends ToolResponse ? R : ToolResponse, params: P, executeTool: (toolId: string, params: Record<string, any>) => Promise<ToolResponse>) => Promise<R extends ToolResponse ? R : ToolResponse>`**: An optional function for executing additional logic *after* the initial HTTP request is made and `transformResponse` (if present) has run.
        *   **`result`**: The structured response from the initial tool execution (or the `transformResponse` output). The `R extends ToolResponse ? R : ToolResponse` syntax means if a specific `R` type was provided for the tool, use that, otherwise default to `ToolResponse`.
        *   **`params`**: The original input parameters given to the tool.
        *   **`executeTool: (toolId: string, params: Record<string, any>) => Promise<ToolResponse>`**: This is a **powerful feature for tool chaining**. It's a function that allows this tool to *call another tool* (identified by `toolId`) with new `params` as part of its post-processing logic. This enables complex workflows where one tool's output feeds into another.
        *   **Returns `Promise<R extends ToolResponse ? R : ToolResponse>`**: It must return a Promise that resolves to a `ToolResponse` (or the specific `R` type), potentially modified or a new response from chained tools.
        *   **Purpose:** Enables complex workflows, conditional logic, and tool chaining, where a single "tool call" can trigger multiple actions or subsequent tool invocations.

    *   **`transformResponse?: (response: Response, params?: P) => Promise<R>`**: An optional function responsible for taking the *raw HTTP `Response` object* received from the `request` and converting it into the structured `ToolResponse` (`R`).
        *   **`response: Response`**: The standard Web Fetch API `Response` object.
        *   **`params?: P`**: The original input parameters, which might be useful for transforming the response.
        *   **Returns `Promise<R>`**: It must return a Promise that resolves to the final, structured `R` (the tool's specific response type).
        *   **Purpose:** To abstract away the details of parsing raw HTTP responses (e.g., JSON parsing, error handling) into a standardized `ToolResponse` format.

#### 8. Helper Interfaces for Data Structures

```typescript
export interface TableRow {
  id: string
  cells: {
    Key: string
    Value: any
  }
}
```

*   **`export interface TableRow { ... }`**: A generic interface defining the structure for a single row in a table-like data set.
    *   **`id: string`**: A unique identifier for the row.
    *   **`cells: { Key: string; Value: any }`**: An object containing key-value pairs representing the cells in that row. `Key` would be the column name, and `Value` its content.
    *   **Purpose:** Provides a consistent schema for tools that might output or consume tabular data.

```typescript
export interface OAuthTokenPayload {
  credentialId: string
  workflowId?: string
}
```

*   **`export interface OAuthTokenPayload { ... }`**: Defines the necessary information to retrieve or manage an OAuth token.
    *   **`credentialId: string`**: A unique identifier for the specific OAuth credential (e.g., which user's Google account).
    *   **`workflowId?: string`**: An optional identifier for the workflow context, which might be relevant for some OAuth implementations.
    *   **Purpose:** To standardize the data passed when requesting or refreshing OAuth tokens.

```typescript
/**
 * File data that tools can return for file-typed outputs
 */
export interface ToolFileData {
  name: string
  mimeType: string
  data?: Buffer | string // Buffer or base64 string
  url?: string // URL to download file from
  size?: number
}
```

*   **`export interface ToolFileData { ... }`**: This interface describes a standardized format for file-based outputs from a tool.
    *   **`name: string`**: The name of the file (e.g., `'report.pdf'`).
    *   **`mimeType: string`**: The MIME type of the file (e.g., `'application/pdf'`, `'image/png'`).
    *   **`data?: Buffer | string`**: Optional. The actual content of the file. It can be a Node.js `Buffer` (for binary data) or a Base64 encoded string.
    *   **`url?: string`**: Optional. If the file is stored externally, this provides a URL where it can be downloaded. This is an alternative to providing the `data` directly.
    *   **`size?: number`**: Optional. The size of the file in bytes.
    *   **Purpose:** To provide a consistent way for tools to return files, whether embedded directly or as a downloadable link, allowing consuming applications to handle them appropriately.

---

In summary, this `index.ts` file is critical for defining the robust, flexible, and type-safe architecture necessary to integrate and manage various external functionalities as "tools" within a larger system, particularly beneficial for AI agents that need to interact with the real world through APIs.