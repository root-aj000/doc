This file defines a "tool" designed to send emails using the Google Gmail API. It acts as a configuration blueprint for a system (likely an AI agent framework or a general-purpose tool orchestrator) that needs to understand how to interact with the Gmail service to send messages.

Think of it as a detailed instruction manual for a robot:
*   "To send an email, here's what you need to know."
*   "These are the pieces of information I need from you (recipient, subject, body)."
*   "This is how you should authenticate."
*   "This is how you should talk to the Gmail API (which URL, what data to send, how to format it)."
*   "This is what kind of response you'll get back, and how to interpret it."

The core purpose is to encapsulate all the necessary information for sending an email via Gmail into a single, standardized, and reusable `ToolConfig` object.

---

### Simplified Complex Logic: Constructing the Raw Email Body

The most intricate part of this code is how it constructs the email body for the Gmail API. The Gmail API doesn't just take `to`, `subject`, `body` as separate fields for sending raw emails; it expects a single, specially formatted string that represents the entire email, including its headers, in a format compliant with email standards (like RFC 2822).

Here's the simplification:

1.  **Email as a Text String:** Imagine an email is just a long text file. At the beginning of this text file, you have "headers" (like `To: recipient@example.com`, `Subject: My Email`, `Content-Type: text/plain`). After all the headers, there's an empty line, and then the actual message content (the "body").

    This code manually builds that text string:
    ```
    Content-Type: text/plain; charset="UTF-8"
    MIME-Version: 1.0
    To: [recipient email]
    Cc: [CC emails, if any]
    Bcc: [BCC emails, if any]
    Subject: [email subject]

    [email body content]
    ```

2.  **Encoding for API Transmission (Base64url):** Once this complete email string is put together, it's treated as raw data. Because email content can contain various characters (including special ones) that aren't always safe or easy to transmit directly within a web request body, the Gmail API requires this raw email string to be *encoded*.

    The encoding used here is `base64url`. This is a way to convert any sequence of bytes (our raw email string) into an ASCII string that is safe to use in URLs and JSON, and is specifically required by the Gmail API for sending raw messages. It essentially makes sure all characters are "web-friendly" before sending them over the internet.

So, in short: The code first builds the complete email content as a standard text string (including headers), then converts this string into a special "web-safe" format (`base64url`) that the Gmail API understands for sending raw emails.

---

### Line-by-Line Explanation

```typescript
import type { GmailSendParams, GmailToolResponse } from '@/tools/gmail/types'
```
*   **`import type { ... } from '...'`**: This line imports TypeScript *type definitions*. These are not actual code that runs, but rather provide information to TypeScript to help ensure your code is correctly structured and used.
    *   **`GmailSendParams`**: This is a type that defines the expected input parameters when someone wants to use this "Gmail Send" tool (e.g., `to`, `subject`, `body`, `accessToken`).
    *   **`GmailToolResponse`**: This is a type that defines the expected structure of the output returned by this "Gmail Send" tool after it successfully sends an email.
*   **`from '@/tools/gmail/types'`**: These types are imported from a specific file path within the project, indicating where these definitions are located.

```typescript
import { GMAIL_API_BASE } from '@/tools/gmail/utils'
```
*   **`import { GMAIL_API_BASE } from '...'`**: This line imports a constant value.
    *   **`GMAIL_API_BASE`**: This is likely a string constant that holds the base URL for the Gmail API (e.g., `https://www.googleapis.com/gmail/v1/users/me`).
*   **`from '@/tools/gmail/utils'`**: This constant is imported from a utility file related to Gmail tools.

```typescript
import type { ToolConfig } from '@/tools/types'
```
*   **`import type { ToolConfig } from '...'`**: Imports another TypeScript type definition.
    *   **`ToolConfig`**: This is a generic type that defines the overall structure for *any* tool configuration within this system. It takes two generic arguments: `TParams` (the type of the input parameters) and `TResponse` (the type of the tool's output).

```typescript
export const gmailSendTool: ToolConfig<GmailSendParams, GmailToolResponse> = {
```
*   **`export const gmailSendTool`**: This declares a constant variable named `gmailSendTool` and makes it available for other files to import and use (`export`).
*   **`: ToolConfig<GmailSendParams, GmailToolResponse>`**: This is a TypeScript type annotation. It specifies that `gmailSendTool` must conform to the `ToolConfig` interface, using `GmailSendParams` as its input parameter type and `GmailToolResponse` as its output response type. This ensures that the object we're defining has all the required properties and methods as specified by `ToolConfig`.
*   **`=`**: Assigns the object literal that follows to the `gmailSendTool` constant.
*   **`{ ... }`**: This is the start of the object definition that holds all the configuration for our Gmail Send tool.

---

### `gmailSendTool` Object Properties

#### Basic Tool Identification

```typescript
  id: 'gmail_send',
```
*   **`id`**: A unique string identifier for this tool within the system.

```typescript
  name: 'Gmail Send',
```
*   **`name`**: A human-readable name for the tool, used in user interfaces or logs.

```typescript
  description: 'Send emails using Gmail',
```
*   **`description`**: A brief explanation of what the tool does.

```typescript
  version: '1.0.0',
```
*   **`version`**: The version number of this tool configuration.

#### OAuth (Authentication) Configuration

```typescript
  oauth: {
    required: true,
    provider: 'google-email',
    additionalScopes: ['https://www.googleapis.com/auth/gmail.send'],
  },
```
*   **`oauth`**: This object defines the authentication requirements for using this tool.
    *   **`required: true`**: Indicates that this tool *must* be authenticated to be used.
    *   **`provider: 'google-email'`**: Specifies which OAuth provider to use for authentication (in this case, Google's email services).
    *   **`additionalScopes: ['https://www.googleapis.com/auth/gmail.send']`**: This is crucial. It lists the specific permissions (OAuth scopes) that the user must grant to this application for it to send emails on their behalf. `https://www.googleapis.com/auth/gmail.send` is the specific scope needed to send emails via the Gmail API.

#### `params` (Input Parameters)

This section defines the arguments or inputs that the tool expects when it's invoked. Each parameter has a type, whether it's required, its visibility, and a description.

```typescript
  params: {
    accessToken: {
      type: 'string',
      required: true,
      visibility: 'hidden',
      description: 'Access token for Gmail API',
    },
```
*   **`accessToken`**:
    *   **`type: 'string'`**: It expects a string value.
    *   **`required: true`**: This parameter is mandatory for the tool to function.
    *   **`visibility: 'hidden'`**: This token should not be exposed directly to end-users or even to an LLM (Large Language Model) if this is part of an AI agent system. It's a sensitive credential provided by the underlying system.
    *   **`description: 'Access token for Gmail API'`**: Explains its purpose.

```typescript
    to: {
      type: 'string',
      required: true,
      visibility: 'user-or-llm',
      description: 'Recipient email address',
    },
```
*   **`to`**:
    *   **`type: 'string'`**: Expects a string (the recipient's email).
    *   **`required: true`**: Mandatory.
    *   **`visibility: 'user-or-llm'`**: This parameter can be provided either by a direct user input or generated by an LLM that is using this tool.
    *   **`description: 'Recipient email address'`**: Explains its purpose.

```typescript
    subject: {
      type: 'string',
      required: true,
      visibility: 'user-or-llm',
      description: 'Email subject',
    },
```
*   **`subject`**: Similar to `to`, but for the email's subject line. Mandatory and visible.

```typescript
    body: {
      type: 'string',
      required: true,
      visibility: 'user-or-llm',
      description: 'Email body content',
    },
```
*   **`body`**: Similar to `to`, but for the main content of the email. Mandatory and visible.

```typescript
    cc: {
      type: 'string',
      required: false,
      visibility: 'user-or-llm',
      description: 'CC recipients (comma-separated)',
    },
```
*   **`cc`**:
    *   **`type: 'string'`**: Expects a string (comma-separated CC email addresses).
    *   **`required: false`**: This parameter is optional.
    *   **`visibility: 'user-or-llm'`**: Can be provided by user or LLM.
    *   **`description: 'CC recipients (comma-separated)'`**: Explains its purpose.

```typescript
    bcc: {
      type: 'string',
      required: false,
      visibility: 'user-or-llm',
      description: 'BCC recipients (comma-separated)',
    },
  },
```
*   **`bcc`**: Similar to `cc`, but for BCC recipients. Optional and visible.

#### `request` (HTTP Request Configuration)

This section defines how to construct the actual HTTP request to the Gmail API.

```typescript
  request: {
    url: () => `${GMAIL_API_BASE}/messages/send`,
```
*   **`url`**: A function that returns the URL for the API endpoint.
    *   **`() => ...`**: This is an arrow function that takes no arguments.
    *   **`` `${GMAIL_API_BASE}/messages/send` ``**: It constructs the full URL by combining the base Gmail API URL (imported earlier) with the specific endpoint for sending messages (`/messages/send`).

```typescript
    method: 'POST',
```
*   **`method`**: The HTTP method to use for the request, which is `POST` for sending data.

```typescript
    headers: (params: GmailSendParams) => ({
      Authorization: `Bearer ${params.accessToken}`,
      'Content-Type': 'application/json',
    }),
```
*   **`headers`**: A function that returns an object containing HTTP headers for the request.
    *   **`(params: GmailSendParams) => ({ ... })`**: It takes the tool's input `params` (which includes `accessToken`) as an argument.
    *   **`Authorization: `Bearer ${params.accessToken}``**: This is the standard way to send an OAuth 2.0 access token. The `Bearer` scheme indicates that the token is a bearer token, meaning anyone who possesses it can use it.
    *   **`'Content-Type': 'application/json'`**: Specifies that the body of the request will be in JSON format.

```typescript
    body: (params: GmailSendParams): Record<string, any> => {
      const emailHeaders = [
        'Content-Type: text/plain; charset="UTF-8"',
        'MIME-Version: 1.0',
        `To: ${params.to}`,
      ]
```
*   **`body`**: This is a function that constructs the HTTP request body. It takes the tool's input `params` and returns an object.
    *   **`const emailHeaders = [...]`**: Initializes an array to hold individual email header lines.
        *   **`'Content-Type: text/plain; charset="UTF-8"'`**: Specifies that the email content is plain text and uses UTF-8 character encoding.
        *   **`'MIME-Version: 1.0'`**: Standard MIME version header.
        *   **`` `To: ${params.to}` ``**: Adds the recipient's email address using a template literal to insert the `to` parameter.

```typescript
      if (params.cc) {
        emailHeaders.push(`Cc: ${params.cc}`)
      }
      if (params.bcc) {
        emailHeaders.push(`Bcc: ${params.bcc}`)
      }
```
*   **`if (params.cc) { ... }`**: If a CC recipient is provided in the `params`, add a `Cc:` header line to the `emailHeaders` array.
*   **`if (params.bcc) { ... }`**: If a BCC recipient is provided, add a `Bcc:` header line.

```typescript
      emailHeaders.push(`Subject: ${params.subject}`, '', params.body)
      const email = emailHeaders.join('\n')
```
*   **`emailHeaders.push(...)`**:
    *   **`` `Subject: ${params.subject}` ``**: Adds the email subject.
    *   **`''`**: Pushes an *empty string*. This is crucial! In email formatting, headers are separated from the message body by a single blank line.
    *   **`params.body`**: Pushes the actual email body content.
*   **`const email = emailHeaders.join('\n')`**: Joins all the collected header and body parts into a single string, using a newline character (`\n`) to separate each line. This `email` string now represents the complete raw email in its standard text format.

```typescript
      return {
        raw: Buffer.from(email).toString('base64url'),
      }
    },
  },
```
*   **`return { ... }`**: This object is what will be sent as the HTTP request body to the Gmail API.
    *   **`raw:`**: The Gmail API expects the raw email content under a key named `raw`.
    *   **`Buffer.from(email)`**: Converts the raw email string into a Node.js `Buffer` (a way to handle binary data).
    *   **`.toString('base64url')`**: Encodes this buffer into a `base64url` string. As explained above, this converts the raw email into a web-safe format required by the Gmail API for sending messages.

#### `transformResponse` (Response Processing)

This section defines how to process the response received from the Gmail API after the request is made.

```typescript
  transformResponse: async (response) => {
    const data = await response.json()
```
*   **`transformResponse: async (response) => { ... }`**: This is an asynchronous function that takes the raw HTTP `response` object from the API call.
*   **`const data = await response.json()`**: It waits for the response body to be parsed as JSON and assigns the resulting JavaScript object to the `data` variable.

```typescript
    return {
      success: true,
      output: {
        content: 'Email sent successfully',
        metadata: {
          id: data.id,
          threadId: data.threadId,
          labelIds: data.labelIds,
        },
      },
    }
  },
```
*   **`return { ... }`**: Returns a structured object conforming to the `GmailToolResponse` type.
    *   **`success: true`**: Indicates that the operation was successful.
    *   **`output: { ... }`**: Contains the actual output of the tool.
        *   **`content: 'Email sent successfully'`**: A simple, human-readable success message.
        *   **`metadata: { ... }`**: An object containing additional details about the sent email, extracted directly from the Gmail API's response (`data`).
            *   **`id: data.id`**: The unique ID of the sent message.
            *   **`threadId: data.threadId`**: The ID of the conversation thread this email belongs to.
            *   **`labelIds: data.labelIds`**: An array of labels applied to the sent email (e.g., 'SENT').

#### `outputs` (Output Description)

This section formally describes the structure of the output that this tool will produce, similar to how `params` describes inputs.

```typescript
  outputs: {
    content: { type: 'string', description: 'Success message' },
```
*   **`content`**:
    *   **`type: 'string'`**: The `content` field of the output will be a string.
    *   **`description: 'Success message'`**: Describes what this string represents.

```typescript
    metadata: {
      type: 'object',
      description: 'Email metadata',
      properties: {
        id: { type: 'string', description: 'Gmail message ID' },
        threadId: { type: 'string', description: 'Gmail thread ID' },
        labelIds: { type: 'array', items: { type: 'string' }, description: 'Email labels' },
      },
    },
  },
}
```
*   **`metadata`**:
    *   **`type: 'object'`**: The `metadata` field of the output will be an object.
    *   **`description: 'Email metadata'`**: Describes what this object contains.
    *   **`properties: { ... }`**: Defines the expected properties within the `metadata` object:
        *   **`id`**: A string representing the Gmail message ID.
        *   **`threadId`**: A string representing the Gmail thread ID.
        *   **`labelIds`**: An array where each item is a string, representing the email's labels.

---

In summary, this `gmailSendTool` object provides a complete, self-contained definition for a "send email" capability, ready to be integrated into a larger system that can interpret this configuration to execute the actual email sending process.