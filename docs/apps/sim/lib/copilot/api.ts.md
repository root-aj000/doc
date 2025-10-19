```typescript
import { createLogger } from '@/lib/logs/console/logger'

const logger = createLogger('CopilotAPI')

/**
 * Citation interface for documentation references
 */
export interface Citation {
  id: number
  title: string
  url: string
  similarity?: number
}

/**
 * Message interface for copilot conversations
 */
export interface CopilotMessage {
  id: string
  role: 'user' | 'assistant' | 'system'
  content: string
  timestamp: string
  citations?: Citation[]
}

/**
 * Chat interface for copilot conversations
 */
export interface CopilotChat {
  id: string
  title: string | null
  model: string
  messages: CopilotMessage[]
  messageCount: number
  previewYaml: string | null
  createdAt: Date
  updatedAt: Date
}

/**
 * File attachment interface for message requests
 */
export interface MessageFileAttachment {
  id: string
  key: string
  filename: string
  media_type: string
  size: number
}

/**
 * Request interface for sending messages
 */
export interface SendMessageRequest {
  message: string
  userMessageId?: string // ID from frontend for the user message
  chatId?: string
  workflowId?: string
  mode?: 'ask' | 'agent'
  model?:
    | 'gpt-5-fast'
    | 'gpt-5'
    | 'gpt-5-medium'
    | 'gpt-5-high'
    | 'gpt-4o'
    | 'gpt-4.1'
    | 'o3'
    | 'claude-4-sonnet'
    | 'claude-4.5-sonnet'
    | 'claude-4.1-opus'
  prefetch?: boolean
  createNewChat?: boolean
  stream?: boolean
  implicitFeedback?: string
  fileAttachments?: MessageFileAttachment[]
  abortSignal?: AbortSignal
  contexts?: Array<{
    kind: string
    label?: string
    chatId?: string
    workflowId?: string
    executionId?: string
  }>
}

/**
 * Base API response interface
 */
export interface ApiResponse {
  success: boolean
  error?: string
  status?: number
}

/**
 * Streaming response interface
 */
export interface StreamingResponse extends ApiResponse {
  stream?: ReadableStream
}

/**
 * Handle API errors and return user-friendly error messages
 */
async function handleApiError(response: Response, defaultMessage: string): Promise<string> {
  try {
    const data = await response.json()
    return (data && (data.error || data.message)) || defaultMessage
  } catch {
    return `${defaultMessage} (${response.status})`
  }
}

/**
 * Send a streaming message to the copilot chat API
 * This is the main API endpoint that handles all chat operations
 */
export async function sendStreamingMessage(
  request: SendMessageRequest
): Promise<StreamingResponse> {
  try {
    // Destructure the request object to separate the AbortSignal from the rest of the request body.
    // This allows us to pass the request body without the AbortSignal to the API.
    const { abortSignal, ...requestBody } = request
    try {
      const preview = Array.isArray((requestBody as any).contexts)
        ? (requestBody as any).contexts.map((c: any) => ({
            kind: c?.kind,
            chatId: c?.chatId,
            workflowId: c?.workflowId,
            label: c?.label,
          }))
        : undefined
      logger.info('Preparing to send streaming message', {
        hasContexts: Array.isArray((requestBody as any).contexts),
        contextsCount: Array.isArray((requestBody as any).contexts)
          ? (requestBody as any).contexts.length
          : 0,
        contextsPreview: preview,
      })
    } catch {}

    // Make an API request to the copilot chat endpoint.
    const response = await fetch('/api/copilot/chat', {
      method: 'POST', // Use the POST method for sending data.
      headers: { 'Content-Type': 'application/json' }, // Set the content type to JSON.
      body: JSON.stringify({ ...requestBody, stream: true }), // Serialize the request body to JSON, ensuring stream: true is sent.
      signal: abortSignal, // Pass the AbortSignal to the fetch request to allow cancellation.
      credentials: 'include', // Include cookies for session authentication
    })

    // Check if the response was successful (status code 200-299).
    if (!response.ok) {
      // If the response was not successful, handle the error and return an error response.
      const errorMessage = await handleApiError(response, 'Failed to send streaming message')
      return {
        success: false,
        error: errorMessage,
        status: response.status,
      }
    }

    // Check if the response has a body.
    if (!response.body) {
      // If the response has no body, return an error response.
      return {
        success: false,
        error: 'No response body received',
        status: 500,
      }
    }

    // If the response was successful and has a body, return a success response with the stream.
    return {
      success: true,
      stream: response.body, // Return the response body as a stream.
    }
  } catch (error) {
    // Handle any errors that occur during the API request.
    // Specifically handle AbortErrors, which occur when the user cancels the request.
    if (error instanceof Error && error.name === 'AbortError') {
      logger.info('Streaming message was aborted by user')
      return {
        success: false,
        error: 'Request was aborted',
      }
    }

    logger.error('Failed to send streaming message:', error)
    return {
      success: false,
      error: error instanceof Error ? error.message : 'Unknown error',
    }
  }
}
```

### Purpose of this file

This TypeScript file defines the data structures (interfaces) and the function `sendStreamingMessage` necessary for interacting with a Copilot chat API. It handles sending user messages, managing chat context, and receiving responses from the API in a streaming fashion. The file also includes error handling to provide user-friendly error messages.

### Explanation of each line of code

1.  **`import { createLogger } from '@/lib/logs/console/logger'`**

    *   Imports the `createLogger` function from a module responsible for creating console loggers. This allows the code to log events and errors for debugging and monitoring purposes.  The `@` alias likely refers to the project's `src` directory, indicating a project-relative import.

2.  **`const logger = createLogger('CopilotAPI')`**

    *   Creates a logger instance using the `createLogger` function, named 'CopilotAPI'.  This logger will be used throughout the file to record information related to API calls and processing.

3.  **`export interface Citation { ... }`**

    *   Defines an interface named `Citation`. This interface represents a documentation reference that supports the AI's response. It contains:
        *   `id`: A unique identifier for the citation (number).
        *   `title`: The title of the cited document (string).
        *   `url`: The URL of the cited document (string).
        *   `similarity`: An optional number that indicates how closely the citation matches the user's query.

4.  **`export interface CopilotMessage { ... }`**

    *   Defines an interface named `CopilotMessage`. This interface represents a single message within a Copilot conversation. It contains:
        *   `id`: A unique identifier for the message (string).
        *   `role`:  Indicates who sent the message, which can be 'user', 'assistant', or 'system' (string).
        *   `content`: The actual text content of the message (string).
        *   `timestamp`:  A string representing the time when the message was created.
        *   `citations`: An optional array of `Citation` objects, indicating the sources used to generate the message.

5.  **`export interface CopilotChat { ... }`**

    *   Defines an interface named `CopilotChat`. This interface represents an entire Copilot chat conversation. It contains:
        *   `id`: A unique identifier for the chat (string).
        *   `title`: The title of the chat, which can be `null` if not set (string | null).
        *   `model`: The name or identifier of the language model used in the chat (string).
        *   `messages`: An array of `CopilotMessage` objects, representing the messages in the chat.
        *   `messageCount`: The number of messages in the chat (number).
        *   `previewYaml`: A YAML representation of a preview, which can be `null` if not available (string | null).
        *   `createdAt`: A `Date` object representing when the chat was created.
        *   `updatedAt`: A `Date` object representing when the chat was last updated.

6.  **`export interface MessageFileAttachment { ... }`**

    *   Defines an interface named `MessageFileAttachment`. This interface represents a file attached to a user's message. It contains:
        *   `id`: A unique identifier for the file attachment (string).
        *   `key`:  The storage key for the file (string).
        *   `filename`: The original name of the file (string).
        *   `media_type`: The MIME type of the file (string).
        *   `size`: The size of the file in bytes (number).

7.  **`export interface SendMessageRequest { ... }`**

    *   Defines an interface named `SendMessageRequest`. This interface represents the data structure for sending a message to the Copilot API. It contains:
        *   `message`: The text content of the user's message (string).
        *   `userMessageId`: An optional ID from the frontend for the user message (string).
        *   `chatId`: An optional ID of the existing chat to which the message belongs (string).
        *   `workflowId`: An optional ID for an associated workflow (string).
        *   `mode`: An optional string indicating the mode such as "ask" or "agent"
        *   `model`: An optional string specifying which language model to use (string), with a union of possible model identifiers.
        *   `prefetch`: An optional boolean indicating whether to prefetch data (boolean).
        *   `createNewChat`: An optional boolean indicating whether to create a new chat (boolean).
        *   `stream`: An optional boolean, defaults to false, but should be set to true to enable stream.
        *   `implicitFeedback`: An optional string that enables the passing of feedback to the backend
        *   `fileAttachments`: An optional array of `MessageFileAttachment` objects (MessageFileAttachment[]).
        *   `abortSignal`: An optional `AbortSignal` object used to cancel the request (AbortSignal).
        *  `contexts`: An optional array of objects of kind, label, chatId, workflowId, and executionId (Array<...>)

8.  **`export interface ApiResponse { ... }`**

    *   Defines an interface named `ApiResponse`. This interface represents the base structure for API responses. It contains:
        *   `success`: A boolean indicating whether the API call was successful (boolean).
        *   `error`: An optional error message (string).
        *   `status`: An optional HTTP status code (number).

9.  **`export interface StreamingResponse { ... }`**

    *   Defines an interface named `StreamingResponse`. This interface represents the structure for API responses that return a stream of data. It extends the `ApiResponse` interface and adds:
        *   `stream`: An optional `ReadableStream` object, which allows reading data in chunks (ReadableStream).

10. **`async function handleApiError(response: Response, defaultMessage: string): Promise<string> { ... }`**

    *   Defines an asynchronous function named `handleApiError`. This function takes a `Response` object (from a `fetch` call) and a default error message as input. It attempts to parse the response body as JSON and extract an error message from it. If parsing fails or no error message is found in the JSON, it returns the `defaultMessage` along with the HTTP status code.  This centralizes error handling and provides more informative error messages.
    *   **`try { ... } catch { ... }`**:  A `try...catch` block is used for error handling.  It attempts to parse the JSON response. If an error occurs during parsing (e.g., the response is not valid JSON), the `catch` block is executed.
    *   **`const data = await response.json()`**:  Attempts to parse the response body as JSON.  The `await` keyword means the function will pause execution until the promise returned by `response.json()` is resolved.
    *   **`return (data && (data.error || data.message)) || defaultMessage`**:  This line attempts to extract an error message from the parsed JSON data.
        *   `data && ...`:  First, it checks if `data` is truthy (i.e., not `null` or `undefined`).  This prevents errors if the JSON parsing failed.
        *   `(data.error || data.message)`:  If `data` exists, it checks if `data.error` exists.  If it does, that value is returned. Otherwise, it checks if `data.message` exists and returns that value if it does. This handles different formats for error messages in the API response.
        *   `... || defaultMessage`:  If neither `data.error` nor `data.message` exists, it returns the `defaultMessage` that was passed as an argument to the function.
    *   **`return `${defaultMessage} (${response.status})``**:  If an error occurs during JSON parsing, the `catch` block is executed.  This line returns the `defaultMessage` along with the HTTP status code of the response, providing some context about the error.

11. **`export async function sendStreamingMessage(request: SendMessageRequest): Promise<StreamingResponse> { ... }`**

    *   Defines an asynchronous function named `sendStreamingMessage`. This is the main function in the file. It takes a `SendMessageRequest` object as input and returns a `Promise` that resolves to a `StreamingResponse`. This function is responsible for sending a message to the Copilot API and handling the streaming response.
    *   **`const { abortSignal, ...requestBody } = request`**: This line destructures the `request` object.
        *   `abortSignal`: Extracts the `abortSignal` property from the `request` object and assigns it to a variable named `abortSignal`. This signal is used to allow cancelling the request.
        *   `...requestBody`: Uses the rest operator (`...`) to create a new object named `requestBody` that contains all the remaining properties from the `request` object (excluding `abortSignal`). This allows you to pass the rest of the request data without the abortSignal.
    *  **Contexts Preview**: This `try...catch` block attempts to create a preview of the `contexts` array for logging purposes. It maps the contexts array to extract relevant information (kind, chatId, workflowId, label) for each context. If any error occurs during this process, the catch block will prevent the app from breaking.
    *   **`const response = await fetch('/api/copilot/chat', { ... })`**:  This line uses the `fetch` API to make a POST request to the `/api/copilot/chat` endpoint. The `await` keyword means the function will pause execution until the promise returned by `fetch` is resolved.
        *   **`method: 'POST'`**:  Specifies that the request method is POST, which is typically used for sending data to the server.
        *   **`headers: { 'Content-Type': 'application/json' }`**:  Sets the `Content-Type` header to `application/json`, indicating that the request body is in JSON format.
        *   **`body: JSON.stringify({ ...requestBody, stream: true })`**: Serializes the request body object as JSON and sends it as the body of the POST request. It merges the `requestBody` with `stream:true` so the backend can recognize that it needs to use the streaming controller.
        *   **`signal: abortSignal`**:  Passes the `abortSignal` to the `fetch` request. This allows the request to be cancelled using an `AbortController`.
        *   **`credentials: 'include'`**:  This is very important, because the cookie that contains the session is not sent by default.
    *   **`if (!response.ok) { ... }`**:  Checks if the response status code indicates success (200-299). If the response is not ok (e.g., a 400 or 500 error), the code inside the `if` block is executed.
        *   **`const errorMessage = await handleApiError(response, 'Failed to send streaming message')`**:  Calls the `handleApiError` function to get a user-friendly error message from the response.
        *   **`return { success: false, error: errorMessage, status: response.status }`**:  Returns an error `StreamingResponse` object, indicating that the API call failed.
    *   **`if (!response.body) { ... }`**:  Checks if the response body is null or undefined. This can happen if the server returns an empty response.
        *   **`return { success: false, error: 'No response body received', status: 500 }`**:  Returns an error `StreamingResponse` object, indicating that the API call failed because the response body was empty.
    *   **`return { success: true, stream: response.body }`**:  If the response was successful and the response body is not empty, this line returns a success `StreamingResponse` object, including the `ReadableStream` from the response body. This allows the client to consume the response data in chunks as it becomes available.
    *   **`catch (error) { ... }`**:  A `try...catch` block is used to handle errors that may occur during the API call.
        *   **`if (error instanceof Error && error.name === 'AbortError') { ... }`**:  Checks if the error is an `AbortError`. This error is thrown when the `AbortSignal` is used to cancel the `fetch` request.
            *   **`logger.info('Streaming message was aborted by user')`**: Logs an info message indicating the user aborted the request
            *   **`return { success: false, error: 'Request was aborted' }`**:  Returns an error `StreamingResponse` object, indicating that the request was aborted.
        *   **`logger.error('Failed to send streaming message:', error)`**:  Logs an error message indicating that the API call failed.
        *   **`return { success: false, error: error instanceof Error ? error.message : 'Unknown error' }`**:  Returns an error `StreamingResponse` object, including the error message.

### Summary

This file is a crucial component for integrating a Copilot chat feature with a backend API. It provides type safety through interfaces, handles API communication using `fetch`, gracefully manages streaming responses, and includes robust error handling. The code is well-structured and uses best practices for asynchronous JavaScript development, making it easy to understand and maintain.
