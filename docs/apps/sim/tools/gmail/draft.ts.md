This TypeScript file defines a "tool" configuration, specifically for drafting emails using the Gmail API. In a larger system (like an AI agent framework, an automation platform, or a microservice orchestration layer), this configuration acts as a blueprint, telling the system:

1.  **What the tool is:** Its name, description, and unique identifier.
2.  **What inputs it needs:** The parameters required to perform its action (e.g., recipient, subject, body).
3.  **How to authenticate:** Which authentication method it requires (Google OAuth in this case).
4.  **How to make the API call:** The exact HTTP request (URL, method, headers, and body) to send to the Gmail API.
5.  **How to interpret the response:** How to take the raw API response and transform it into a useful, standardized output for the system.
6.  **What output it provides:** The structured schema of the data it returns upon success.

Essentially, this file sets up a declarative interface for a "Gmail Draft" capability, allowing other parts of the application to interact with Gmail without needing to know the low-level API details.

---

### Detailed Explanation

Let's break down the code section by section:

#### Imports

```typescript
import type { GmailSendParams, GmailToolResponse } from '@/tools/gmail/types'
import type { ToolConfig } from '@/tools/types'
```

*   **`import type { GmailSendParams, GmailToolResponse } } from '@/tools/gmail/types'`**: This line imports two TypeScript types:
    *   `GmailSendParams`: Defines the structure of the input parameters (like `to`, `subject`, `body`) that this tool expects. Using `type` ensures these are only for type checking and don't get compiled into JavaScript.
    *   `GmailToolResponse`: Defines the structure of the output this tool will produce after successfully drafting an email.
    *   The path (`@/tools/gmail/types`) suggests these types are defined in a shared types file within a larger `gmail` module.
*   **`import type { ToolConfig } from '@/tools/types'`**: This line imports the `ToolConfig` type. This is a generic type that likely defines the overall structure for *any* tool in the system, ensuring all tools conform to a consistent interface. It takes two type arguments: `GmailSendParams` (for the input parameters) and `GmailToolResponse` (for the output).
    *   The path (`@/tools/types`) suggests this is a common type definition for all tools in the system.

#### Constant: Gmail API Base URL

```typescript
const GMAIL_API_BASE = 'https://gmail.googleapis.com/gmail/v1/users/me'
```

*   **`const GMAIL_API_BASE = 'https://gmail.googleapis.com/gmail/v1/users/me'`**: This declares a constant string `GMAIL_API_BASE`. It holds the base URL for making API requests to the Gmail API for the authenticated user (`/users/me`). Using a constant makes the code cleaner and easier to update if the API endpoint changes.

#### The `gmailDraftTool` Configuration Object

```typescript
export const gmailDraftTool: ToolConfig<GmailSendParams, GmailToolResponse> = {
  // ... configuration details ...
}
```

*   **`export const gmailDraftTool: ToolConfig<GmailSendParams, GmailToolResponse> = { ... }`**: This is the main definition of our Gmail drafting tool.
    *   `export`: Makes this `gmailDraftTool` object available for other files to import and use.
    *   `const gmailDraftTool`: Declares a constant variable named `gmailDraftTool`.
    *   `: ToolConfig<GmailSendParams, GmailToolResponse>`: This is a TypeScript type annotation. It states that `gmailDraftTool` *must* conform to the `ToolConfig` interface, using `GmailSendParams` for its input parameters and `GmailToolResponse` for its output. This provides strong type checking and ensures the tool's definition is correct.

Now, let's look at the properties *inside* this `gmailDraftTool` object:

---

##### Basic Tool Metadata

```typescript
  id: 'gmail_draft',
  name: 'Gmail Draft',
  description: 'Draft emails using Gmail',
  version: '1.0.0',
```

*   **`id: 'gmail_draft'`**: A unique identifier for this specific tool. This is often used internally by the system to reference the tool.
*   **`name: 'Gmail Draft'`**: A human-readable name for the tool, which might be displayed in a UI or logs.
*   **`description: 'Draft emails using Gmail'`**: A brief explanation of what the tool does. Useful for documentation or when presented to a user/LLM.
*   **`version: '1.0.0'`**: A version number for the tool, useful for managing updates and compatibility.

---

##### OAuth (Authentication) Configuration

```typescript
  oauth: {
    required: true,
    provider: 'google-email',
    additionalScopes: [],
  },
```

*   This `oauth` object defines the authentication requirements for this tool.
    *   **`required: true`**: Specifies that this tool absolutely requires authentication to function.
    *   **`provider: 'google-email'`**: Indicates that the authentication should be handled by a service known as `'google-email'`. This likely refers to a pre-configured OAuth provider within the larger system that manages Google account access.
    *   **`additionalScopes: []`**: An array for specifying any *extra* OAuth scopes (permissions) needed beyond what the default `'google-email'` provider grants. In this case, it's an empty array, meaning the default permissions for sending emails are sufficient.

---

##### Parameters (Inputs) Definition

```typescript
  params: {
    accessToken: {
      type: 'string',
      required: true,
      visibility: 'hidden',
      description: 'Access token for Gmail API',
    },
    to: { /* ... */ },
    subject: { /* ... */ },
    body: { /* ... */ },
    cc: { /* ... */ },
    bcc: { /* ... */ },
  },
```

This `params` object defines all the inputs the tool expects. Each property within `params` is a specific input parameter:

*   **`accessToken`**:
    *   **`type: 'string'`**: The access token is expected to be a string.
    *   **`required: true`**: This parameter *must* be provided.
    *   **`visibility: 'hidden'`**: This is a crucial detail. It means this parameter is not exposed directly to a user or a Language Model (LLM). It's typically injected by the underlying system after successful OAuth authentication.
    *   **`description: 'Access token for Gmail API'`**: Explains its purpose.
*   **`to`**:
    *   **`type: 'string'`**: The recipient's email address.
    *   **`required: true`**: A recipient is mandatory.
    *   **`visibility: 'user-or-llm'`**: This parameter can be provided either by a human user or generated by an LLM.
    *   **`description: 'Recipient email address'`**: Self-explanatory.
*   **`subject`**: Similar to `to`, but for the email's subject line. `required: true` and `visibility: 'user-or-llm'`.
*   **`body`**: Similar to `to`, but for the main content of the email. `required: true` and `visibility: 'user-or-llm'`.
*   **`cc`**:
    *   **`type: 'string'`**: CC recipients.
    *   **`required: false`**: This parameter is optional.
    *   **`visibility: 'user-or-llm'`**: Can be provided by user or LLM.
    *   **`description: 'CC recipients (comma-separated)'`**: Specifies the format.
*   **`bcc`**: Similar to `cc`, but for BCC recipients. `required: false` and `visibility: 'user-or-llm'`.

---

##### Request Configuration

```typescript
  request: {
    url: () => `${GMAIL_API_BASE}/drafts`,
    method: 'POST',
    headers: (params: GmailSendParams) => ({ /* ... */ }),
    body: (params: GmailSendParams): Record<string, any> => { /* ... */ },
  },
```

This `request` object defines how to construct the actual HTTP request to the Gmail API.

*   **`url: () => `${GMAIL_API_BASE}/drafts``**:
    *   `url` is a function that returns a string. This allows for dynamic URL construction, though here it's simple.
    *   It constructs the full endpoint for creating a draft: `https://gmail.googleapis.com/gmail/v1/users/me/drafts`.
*   **`method: 'POST'`**: Specifies that the HTTP request should use the `POST` method, which is standard for creating new resources.
*   **`headers: (params: GmailSendParams) => ({ ... })`**:
    *   `headers` is a function that takes the tool's input `params` and returns an object of HTTP headers.
    *   **`Authorization: `Bearer ${params.accessToken}``**: This is crucial for authentication. It adds an `Authorization` header with the `Bearer` token scheme, using the `accessToken` provided in the tool's parameters.
    *   **`'Content-Type': 'application/json'`**: Tells the server that the request body will be in JSON format.
*   **`body: (params: GmailSendParams): Record<string, any> => { ... }`**:
    *   `body` is a function that takes the tool's input `params` and constructs the request body. This is the most complex part of the configuration.
    *   The Gmail API expects the raw email content (headers + body) to be provided as a Base64 URL-safe encoded string within a JSON object.

    Let's break down the body construction:
    ```typescript
          const emailHeaders = [
            'Content-Type: text/plain; charset="UTF-8"',
            'MIME-Version: 1.0',
            `To: ${params.to}`,
          ]

          if (params.cc) {
            emailHeaders.push(`Cc: ${params.cc}`)
          }
          if (params.bcc) {
            emailHeaders.push(`Bcc: ${params.bcc}`)
          }

          emailHeaders.push(`Subject: ${params.subject}`, '', params.body)
          const email = emailHeaders.join('\n')

          return {
            message: {
              raw: Buffer.from(email).toString('base64url'),
            },
          }
    ```
    *   **`const emailHeaders = [...]`**: An array is initialized to build the email's raw header and body content.
        *   **`'Content-Type: text/plain; charset="UTF-8"'`**: Standard MIME header indicating plain text content with UTF-8 encoding.
        *   **`'MIME-Version: 1.0'`**: Standard MIME version header.
        *   **``To: ${params.to}``**: Adds the "To" recipient from the tool's parameters.
    *   **`if (params.cc) { emailHeaders.push(...) }`**: If a CC recipient is provided in the parameters, a "Cc" header is added.
    *   **`if (params.bcc) { emailHeaders.push(...) }`**: If a BCC recipient is provided, a "Bcc" header is added.
    *   **`emailHeaders.push(`Subject: ${params.subject}`, '', params.body)`**:
        *   Adds the "Subject" header.
        *   Adds an **empty string `''`**: This is critical! In raw email format, an empty line separates the email headers from the email body.
        *   Adds the `params.body`: The actual content of the email.
    *   **`const email = emailHeaders.join('\n')`**: All the header and body parts are joined together into a single string, with each element separated by a newline character (`\n`), creating a complete raw email string.
    *   **`return { message: { raw: Buffer.from(email).toString('base64url'), }, }`**: This is the final object returned as the HTTP request body.
        *   **`message: { ... }`**: The Gmail API expects the email content inside a `message` object.
        *   **`raw: ...`**: The actual email content must be provided in a `raw` field.
        *   **`Buffer.from(email)`**: Converts the constructed `email` string into a Node.js `Buffer` (a raw binary data buffer). This is necessary before encoding.
        *   **`.toString('base64url')`**: Encodes the `Buffer` into a Base64 URL-safe string. The Gmail API specifically requires this encoding for the `raw` field to handle characters that might otherwise cause issues in URLs.

---

##### Response Transformation

```typescript
  transformResponse: async (response) => {
    const data = await response.json()

    return {
      success: true,
      output: {
        content: 'Email drafted successfully',
        metadata: {
          id: data.id,
          message: {
            id: data.message?.id,
            threadId: data.message?.threadId,
            labelIds: data.message?.labelIds,
          },
        },
      },
    }
  },
```

*   **`transformResponse: async (response) => { ... }`**: This is an asynchronous function that takes the raw HTTP `response` from the Gmail API call and transforms it into a standardized output format defined by `GmailToolResponse`.
    *   **`const data = await response.json()`**: Parses the JSON body of the API response into a JavaScript object. This `data` object will contain the details of the newly created draft.
    *   **`return { ... }`**: The function returns a structured object:
        *   **`success: true`**: A boolean flag indicating that the operation was successful.
        *   **`output: { ... }`**: This object holds the actual results of the tool's execution.
            *   **`content: 'Email drafted successfully'`**: A simple, human-readable confirmation message.
            *   **`metadata: { ... }`**: An object containing more detailed information extracted from the Gmail API response.
                *   **`id: data.id`**: The ID of the created draft itself.
                *   **`message: { ... }`**: An object containing metadata about the actual email message *within* the draft.
                    *   **`id: data.message?.id`**: The ID of the Gmail message. The `?.` (optional chaining) handles cases where `data.message` might be null or undefined, preventing errors.
                    *   **`threadId: data.message?.threadId`**: The ID of the email thread this message belongs to.
                    *   **`labelIds: data.message?.labelIds`**: An array of labels (e.g., 'INBOX', 'DRAFT') associated with the message.

---

##### Outputs (Schema) Definition

```typescript
  outputs: {
    content: { type: 'string', description: 'Success message' },
    metadata: {
      type: 'object',
      description: 'Draft metadata',
      properties: {
        id: { type: 'string', description: 'Draft ID' },
        message: {
          type: 'object',
          description: 'Message metadata',
          properties: {
            id: { type: 'string', description: 'Gmail message ID' },
            threadId: { type: 'string', description: 'Gmail thread ID' },
            labelIds: { type: 'array', items: { type: 'string' }, description: 'Email labels' },
          },
        },
      },
    },
  },
```

This `outputs` object describes the *schema* of the data that the `transformResponse` function will return. It's a declarative way to tell the system (and developers) what kind of data to expect from this tool. It closely mirrors the structure defined in `GmailToolResponse` and produced by `transformResponse`.

*   **`content: { type: 'string', description: 'Success message' }`**: Describes the `content` field, which is a string containing a success message.
*   **`metadata: { ... }`**: Describes the `metadata` object.
    *   **`type: 'object'`**: It's an object.
    *   **`description: 'Draft metadata'`**: Describes its purpose.
    *   **`properties: { ... }`**: Defines the properties within the `metadata` object.
        *   **`id: { type: 'string', description: 'Draft ID' }`**: The draft's ID.
        *   **`message: { ... }`**: Describes the nested `message` object.
            *   **`type: 'object'`**: It's an object.
            *   **`description: 'Message metadata'`**: Describes its purpose.
            *   **`properties: { ... }`**: Defines the properties within the `message` object.
                *   **`id: { type: 'string', description: 'Gmail message ID' }`**: The internal Gmail message ID.
                *   **`threadId: { type: 'string', description: 'Gmail thread ID' }`**: The ID of the conversation thread.
                *   **`labelIds: { type: 'array', items: { type: 'string' }, description: 'Email labels' }`**: An array where each item is a string (representing a label ID like 'INBOX', 'DRAFT', 'SENT').

---

In summary, this file is a comprehensive and self-contained configuration for integrating a "Gmail Draft" feature into a larger system, covering everything from input parameters and authentication to the exact API call logic and output formatting.