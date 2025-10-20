This TypeScript file is a utility module designed to simplify the interaction with the Gmail API for reading and processing email messages. It provides functions to parse raw Gmail API responses into more structured and usable formats, extract message content, identify and download attachments, and generate human-readable summaries.

The core idea is to abstract away the complexity of Gmail's multi-part MIME message structure and base64 encoding, allowing other parts of an application to easily access email data.

---

## Detailed Explanation

### Purpose of this file

At a high level, this file acts as a **"Gmail Message Processor"**. It takes the raw, often complex, data structures returned by the Google Gmail API (specifically when fetching messages) and transforms them into more developer-friendly formats. Its main responsibilities include:

1.  **Standardizing Email Data:** Extracting key information like subject, sender, recipient, and date into easily accessible properties.
2.  **Content Extraction:** Recursively finding and decoding the actual text or HTML body of an email, regardless of its MIME structure.
3.  **Attachment Handling:** Identifying attachments within an email and providing a mechanism to download their binary data.
4.  **Summarization:** Offering ways to create concise summaries of single or multiple emails.

This module is crucial for any application that needs to display email content, process it for AI models, or manage attachments, without having to deal with the low-level intricacies of the Gmail API response format in every component.

---

### Code Explanation

```typescript
import type {
  GmailAttachment,
  GmailMessage,
  GmailReadParams,
  GmailToolResponse,
} from '@/tools/gmail/types'
```

*   **`import type { ... } from '@/tools/gmail/types'`**: This line imports several TypeScript types from a local file (`@/tools/gmail/types`).
    *   The `type` keyword ensures that these imports are only used for type checking during development and are completely removed during compilation to JavaScript, preventing unnecessary code bundling.
    *   **`GmailAttachment`**: Represents a processed attachment, likely including its name, MIME type, size, and actual binary data (e.g., as a `Buffer`).
    *   **`GmailMessage`**: Represents the raw or partially processed structure of a Gmail message as returned by the Gmail API. This includes `id`, `threadId`, `labelIds`, `payload`, etc.
    *   **`GmailReadParams`**: Defines parameters that can be passed when reading a Gmail message, such as whether to include attachments and the necessary access token.
    *   **`GmailToolResponse`**: The standardized output format this module produces after processing a Gmail message, indicating success/failure and containing the extracted email content and metadata.

```typescript
export const GMAIL_API_BASE = 'https://gmail.googleapis.com/gmail/v1/users/me'
```

*   **`export const GMAIL_API_BASE = 'https://gmail.googleapis.com/gmail/v1/users/me'`**: This declares and exports a constant string `GMAIL_API_BASE`.
    *   This string forms the base URL for making requests to the Gmail API for the authenticated user (`/users/me`). It's used to construct full API endpoints, ensuring consistency and making it easier to manage the API version and target user.

---

### `processMessage` Function

This is the primary function for taking a raw Gmail message object and turning it into a rich, structured `GmailToolResponse`, including its content and potentially downloaded attachments.

**Simplified Logic:**
The function first validates the input, then extracts common email headers (subject, sender, recipient, date). It delegates the complex tasks of finding the email body and identifying attachments to helper functions. If attachments are requested and available, it downloads them. Finally, it assembles all this information into a standardized output object.

```typescript
export async function processMessage(
  message: GmailMessage,
  params?: GmailReadParams
): Promise<GmailToolResponse> {
```

*   **`export async function processMessage(...)`**: Defines an asynchronous function named `processMessage`.
    *   `async`: Indicates that this function will perform asynchronous operations (like waiting for `downloadAttachments` to complete).
    *   **`message: GmailMessage`**: The first parameter is the raw or partially processed `GmailMessage` object obtained from the Gmail API.
    *   **`params?: GmailReadParams`**: The second parameter is an optional (`?`) object containing additional parameters for processing, such as whether to include attachments and the `accessToken` for downloading them.
    *   **`: Promise<GmailToolResponse>`**: Specifies that the function returns a Promise that resolves to a `GmailToolResponse` object.

```typescript
  // Check if message and payload exist
  if (!message || !message.payload) {
    return {
      success: true,
      output: {
        content: 'Unable to process email: Invalid message format',
        metadata: {
          id: message?.id || '',
          threadId: message?.threadId || '',
          labelIds: message?.labelIds || [],
        },
      },
    }
  }
```

*   **`if (!message || !message.payload)`**: This is an input validation check. A valid Gmail message from the API typically contains a `payload` object, which holds the headers and body parts. If `message` or its `payload` is missing, it means the input is malformed or incomplete.
*   **`return { ... }`**: If validation fails, the function immediately returns a `GmailToolResponse`.
    *   **`success: true`**: This might seem counterintuitive for an "error," but in this context, `success: true` often means the *tool itself* ran without crashing, even if it couldn't fully process the input. The `content` then explains the issue.
    *   **`output: { content: '...', metadata: { ... } }`**: Provides a descriptive error message in the `content` field.
    *   **`id: message?.id || ''`**: Uses optional chaining (`?.`) and nullish coalescing (`|| ''`) to safely get the message ID if it exists, otherwise defaults to an empty string. This pattern is repeated for `threadId` and `labelIds`.

```typescript
  const headers = message.payload.headers || []
  const subject = headers.find((h) => h.name.toLowerCase() === 'subject')?.value || ''
  const from = headers.find((h) => h.name.toLowerCase() === 'from')?.value || ''
  const to = headers.find((h) => h.name.toLowerCase() === 'to')?.value || ''
  const date = headers.find((h) => h.name.toLowerCase() === 'date')?.value || ''
```

*   **`const headers = message.payload.headers || []`**: Retrieves the `headers` array from the `message.payload`. If `headers` is `null` or `undefined`, it defaults to an empty array to prevent errors when trying to call `.find()`.
*   **`const subject = headers.find((h) => h.name.toLowerCase() === 'subject')?.value || ''`**:
    *   Uses `Array.prototype.find()` to search through the `headers` array.
    *   The callback function `(h) => h.name.toLowerCase() === 'subject'` checks if a header's `name` (converted to lowercase for case-insensitivity) is 'subject'.
    *   `?.value`: If a header with the name 'subject' is found, it attempts to access its `value` property using optional chaining (to prevent errors if `find` returns `undefined`).
    *   `|| ''`: If no such header is found, or its value is `null`/`undefined`, it defaults to an empty string.
    *   This pattern is repeated for `from`, `to`, and `date` headers, extracting common email metadata.

```typescript
  // Extract the message body
  const body = extractMessageBody(message.payload)

  // Check for attachments
  const attachmentInfo = extractAttachmentInfo(message.payload)
  const hasAttachments = attachmentInfo.length > 0
```

*   **`const body = extractMessageBody(message.payload)`**: Calls the `extractMessageBody` helper function (explained later) to get the main textual content of the email, passing the message's `payload`.
*   **`const attachmentInfo = extractAttachmentInfo(message.payload)`**: Calls the `extractAttachmentInfo` helper function (explained later) to get a list of metadata for any attachments present in the email.
*   **`const hasAttachments = attachmentInfo.length > 0`**: A simple boolean flag indicating whether any attachments were found.

```typescript
  // Download attachments if requested
  let attachments: GmailAttachment[] | undefined
  if (params?.includeAttachments && hasAttachments && params.accessToken) {
    try {
      attachments = await downloadAttachments(message.id, attachmentInfo, params.accessToken)
    } catch (error) {
      // Continue without attachments rather than failing the entire request
    }
  }
```

*   **`let attachments: GmailAttachment[] | undefined`**: Declares a variable `attachments` that will store an array of `GmailAttachment` objects, or be `undefined` if no attachments are downloaded.
*   **`if (params?.includeAttachments && hasAttachments && params.accessToken)`**: This condition checks three things before attempting to download attachments:
    *   **`params?.includeAttachments`**: If the `includeAttachments` flag is set to `true` in the optional `params` object.
    *   **`hasAttachments`**: If the email actually has any attachments (as determined by `attachmentInfo.length > 0`).
    *   **`params.accessToken`**: If an `accessToken` is provided in the `params`, which is necessary for authentication to download attachments.
*   **`try { ... } catch (error) { ... }`**: A `try...catch` block wraps the attachment download process.
    *   **`attachments = await downloadAttachments(message.id, attachmentInfo, params.accessToken)`**: If all conditions are met, it asynchronously calls the `downloadAttachments` helper function (explained later) to fetch the actual attachment data.
    *   **`catch (error) { ... }`**: If an error occurs during attachment download (e.g., network issue, API error), the error is caught. The comment `// Continue without attachments rather than failing the entire request` explains the strategy: individual attachment failures won't prevent the rest of the email processing from completing.

```typescript
  const result: GmailToolResponse = {
    success: true,
    output: {
      content: body || 'No content found in email',
      metadata: {
        id: message.id || '',
        threadId: message.threadId || '',
        labelIds: message.labelIds || [],
        from,
        to,
        subject,
        date,
        hasAttachments,
        attachmentCount: attachmentInfo.length,
      },
      // Always include attachments array (empty if none downloaded)
      attachments: attachments || [],
    },
  }

  return result
}
```

*   **`const result: GmailToolResponse = { ... }`**: Constructs the final `GmailToolResponse` object.
    *   **`success: true`**: Indicates the processing was successful.
    *   **`output: { ... }`**: Contains the primary data extracted from the email.
        *   **`content: body || 'No content found in email'`**: The extracted email body, defaulting to a message if `body` is empty.
        *   **`metadata: { ... }`**: An object containing various pieces of metadata.
            *   `id`, `threadId`, `labelIds`: Basic message identifiers.
            *   `from`, `to`, `subject`, `date`: Headers extracted earlier.
            *   `hasAttachments`: Boolean flag.
            *   `attachmentCount: attachmentInfo.length`: Total number of attachments identified.
        *   **`attachments: attachments || []`**: The array of downloaded `GmailAttachment` objects. If `attachments` is `undefined` (because they weren't requested or failed to download), it defaults to an empty array `[]`, ensuring this property is always an array.
*   **`return result`**: Returns the fully constructed `GmailToolResponse`.

---

### `processMessageForSummary` Function

This function provides a lightweight way to get basic information about a message without performing the more intensive body extraction or attachment checks. It's useful for displaying lists of emails.

**Simplified Logic:**
Similar to `processMessage` but only extracts essential headers and the message snippet, then returns a simpler object.

```typescript
export function processMessageForSummary(message: GmailMessage): any {
```

*   **`export function processMessageForSummary(message: GmailMessage): any`**: Defines a function to process a message for a summary.
    *   **`message: GmailMessage`**: Takes a raw `GmailMessage` as input.
    *   **`: any`**: The return type is `any` because it returns a simple JavaScript object with specific fields, which might not have a dedicated type alias for this specific "summary" format.

```typescript
  if (!message || !message.payload) {
    return {
      id: message?.id || '',
      threadId: message?.threadId || '',
      subject: 'Unknown Subject',
      from: 'Unknown Sender',
      date: '',
      snippet: message?.snippet || '',
    }
  }
```

*   **`if (!message || !message.payload)`**: Similar validation as `processMessage`. If the message is malformed, it returns a default summary object.
    *   **`subject: 'Unknown Subject'`**, **`from: 'Unknown Sender'`**: Provides default strings for key fields if the message is invalid.
    *   **`snippet: message?.snippet || ''`**: The Gmail API often provides a `snippet` (a short preview of the message content) directly on the `message` object, which is included here.

```typescript
  const headers = message.payload.headers || []
  const subject = headers.find((h) => h.name.toLowerCase() === 'subject')?.value || 'No Subject'
  const from = headers.find((h) => h.name.toLowerCase() === 'from')?.value || 'Unknown Sender'
  const date = headers.find((h) => h.name.toLowerCase() === 'date')?.value || ''
```

*   These lines are identical in logic to the header extraction in `processMessage`, but they use default values like `'No Subject'` and `'Unknown Sender'` for better human readability in a summary.

```typescript
  return {
    id: message.id,
    threadId: message.threadId,
    subject,
    from,
    date,
    snippet: message.snippet || '',
  }
}
```

*   Returns a plain JavaScript object containing the extracted `id`, `threadId`, `subject`, `from`, `date`, and `snippet` for the message.

---

### `extractMessageBody` Function

This is a crucial helper for decoding the actual email content. Gmail messages (and emails in general) can have complex structures with multiple parts (e.g., text, HTML, attachments, nested parts) organized in a MIME tree. This function attempts to find the most relevant text content.

**Simplified Logic:**
It first checks for a direct body. If not found, it recursively searches through parts, prioritizing plain text, then HTML, and finally diving into nested parts until a body is found. All content found is base64 decoded.

```typescript
export function extractMessageBody(payload: any): string {
```

*   **`export function extractMessageBody(payload: any): string`**: Defines a function to extract the message body.
    *   **`payload: any`**: Takes the `payload` object of a `GmailMessage` (which can contain headers, body, and parts) as input. Using `any` here is practical because the `payload` structure can be highly dynamic and deeply nested, making a precise type definition overly complex for a helper.
    *   **`: string`**: Returns the decoded message body as a string.

```typescript
  // If the payload has a body with data, decode it
  if (payload.body?.data) {
    return Buffer.from(payload.body.data, 'base64').toString()
  }
```

*   **`if (payload.body?.data)`**: Checks if the current `payload` (or `part` in recursive calls) directly has a `body` object with a `data` property. This often happens for simple, single-part emails.
*   **`return Buffer.from(payload.body.data, 'base64').toString()`**: If data is found, it's typically base64 encoded.
    *   `Buffer.from(..., 'base64')`: Creates a Node.js `Buffer` object from the base64-encoded string.
    *   `.toString()`: Converts the `Buffer` back into a human-readable string.

```typescript
  // If there are no parts, return empty string
  if (!payload.parts || !Array.isArray(payload.parts) || payload.parts.length === 0) {
    return ''
  }
```

*   **`if (!payload.parts || !Array.isArray(payload.parts) || payload.parts.length === 0)`**: If the current `payload` doesn't have a direct `body.data` and also has no `parts` (or `parts` is not an array, or it's an empty array), it means there's no content to extract at this level, so it returns an empty string.

```typescript
  // First try to find a text/plain part
  const textPart = payload.parts.find((part: any) => part.mimeType === 'text/plain')
  if (textPart?.body?.data) {
    return Buffer.from(textPart.body.data, 'base64').toString()
  }
```

*   **`const textPart = payload.parts.find((part: any) => part.mimeType === 'text/plain')`**: If there are `parts`, it first tries to find a part with `mimeType` set to `'text/plain'`. This is usually the preferred content format as it's cleaner and easier to read than HTML.
*   **`if (textPart?.body?.data)`**: If a `text/plain` part is found and it contains `body.data`, it decodes and returns it.

```typescript
  // If no text/plain, try to find text/html
  const htmlPart = payload.parts.find((part: any) => part.mimeType === 'text/html')
  if (htmlPart?.body?.data) {
    return Buffer.from(htmlPart.body.data, 'base64').toString()
  }
```

*   **`const htmlPart = payload.parts.find((part: any) => part.mimeType === 'text/html')`**: If no `text/plain` part was found, it then looks for a `text/html` part.
*   **`if (htmlPart?.body?.data)`**: If an `text/html` part is found with `body.data`, it decodes and returns it.

```typescript
  // If we have multipart/alternative or other complex types, recursively check parts
  for (const part of payload.parts) {
    if (part.parts) {
      const nestedBody = extractMessageBody(part)
      if (nestedBody) {
        return nestedBody
      }
    }
  }
```

*   **`for (const part of payload.parts)`**: This loop iterates through all the `parts` of the current `payload`. This handles complex MIME structures, like `multipart/alternative` or `multipart/mixed`, where parts themselves can contain nested parts.
*   **`if (part.parts)`**: Checks if the current `part` itself has nested `parts`.
*   **`const nestedBody = extractMessageBody(part)`**: This is the **recursive step**. It calls `extractMessageBody` again, but this time passing the `part` as the new `payload`. This allows the function to traverse the entire MIME tree.
*   **`if (nestedBody)`**: If the recursive call successfully finds and returns a body string, that string is immediately returned, stopping further searching.

```typescript
  // If we couldn't find any text content, return empty string
  return ''
}
```

*   **`return ''`**: If none of the above conditions resulted in finding and returning a message body (e.g., the email was purely attachments, or the structure was unrecognized), an empty string is returned as a fallback.

---

### `extractAttachmentInfo` Function

This function traverses the email payload to identify any attachments and gather their metadata (ID, filename, MIME type, size). It doesn't download the actual file data, just finds references to them.

**Simplified Logic:**
It uses an inner recursive function to walk through all parts of the email. If a part has an `attachmentId` and a `filename`, it's considered an attachment, and its metadata is collected.

```typescript
export function extractAttachmentInfo(
  payload: any
): Array<{ attachmentId: string; filename: string; mimeType: string; size: number }> {
```

*   **`export function extractAttachmentInfo(...)`**: Defines a function to extract attachment information.
    *   **`payload: any`**: Takes the message `payload` object.
    *   **`: Array<{ ... }>`**: Returns an array of objects. Each object describes an attachment with its `attachmentId`, `filename`, `mimeType`, and `size`.

```typescript
  const attachments: Array<{
    attachmentId: string
    filename: string
    mimeType: string
    size: number
  }> = []
```

*   **`const attachments: Array<{ ... }> = []`**: Initializes an empty array `attachments` that will store the metadata of all found attachments.

```typescript
  function processPayloadPart(part: any) {
    // Check if this part has an attachment
    if (part.body?.attachmentId && part.filename) {
      attachments.push({
        attachmentId: part.body.attachmentId,
        filename: part.filename,
        mimeType: part.mimeType || 'application/octet-stream',
        size: part.body.size || 0,
      })
    }

    // Recursively process nested parts
    if (part.parts && Array.isArray(part.parts)) {
      part.parts.forEach(processPayloadPart)
    }
  }
```

*   **`function processPayloadPart(part: any)`**: This is an inner helper function designed to be called recursively. It processes a single part of the email's MIME structure.
    *   **`if (part.body?.attachmentId && part.filename)`**: This is the core condition to identify an attachment. An attachment part in Gmail API responses typically has a `body` object containing an `attachmentId` (a unique ID for fetching the attachment) and a `filename`.
    *   **`attachments.push({ ... })`**: If an attachment is identified, its relevant metadata (`attachmentId`, `filename`, `mimeType`, `size`) is extracted and pushed into the `attachments` array.
        *   `mimeType`: Defaults to `'application/octet-stream'` if not specified.
        *   `size`: Defaults to `0` if not specified.
    *   **`if (part.parts && Array.isArray(part.parts))`**: Checks if the current `part` itself contains nested `parts`.
    *   **`part.parts.forEach(processPayloadPart)`**: If there are nested parts, it recursively calls `processPayloadPart` for each of them. This ensures the entire MIME tree is traversed to find all attachments.

```typescript
  // Process the main payload
  processPayloadPart(payload)

  return attachments
}
```

*   **`processPayloadPart(payload)`**: Starts the recursive process by calling `processPayloadPart` with the initial `payload` object.
*   **`return attachments`**: Returns the array containing all the identified attachment metadata.

---

### `downloadAttachments` Function

This function takes the attachment metadata and an access token, then makes API calls to actually download the binary data for each attachment.

**Simplified Logic:**
It loops through each attachment's metadata. For each, it makes an authenticated `fetch` request to the Gmail API using the attachment's ID. The received base64url data is converted to base64, then decoded into a Node.js Buffer.

```typescript
export async function downloadAttachments(
  messageId: string,
  attachmentInfo: Array<{ attachmentId: string; filename: string; mimeType: string; size: number }>,
  accessToken: string
): Promise<GmailAttachment[]> {
```

*   **`export async function downloadAttachments(...)`**: Defines an asynchronous function to download attachments.
    *   **`messageId: string`**: The ID of the parent message to which the attachments belong.
    *   **`attachmentInfo: Array<{ ... }>`**: An array of attachment metadata (as returned by `extractAttachmentInfo`).
    *   **`accessToken: string`**: The OAuth 2.0 access token required to authenticate with the Gmail API.
    *   **`: Promise<GmailAttachment[]>`**: Returns a Promise that resolves to an array of `GmailAttachment` objects, which include the actual binary data.

```typescript
  const downloadedAttachments: GmailAttachment[] = []

  for (const attachment of attachmentInfo) {
    try {
      // Download attachment from Gmail API
      const attachmentResponse = await fetch(
        `${GMAIL_API_BASE}/messages/${messageId}/attachments/${attachment.attachmentId}`,
        {
          headers: {
            Authorization: `Bearer ${accessToken}`,
            'Content-Type': 'application/json',
          },
        }
      )

      if (!attachmentResponse.ok) {
        // If an attachment fails, log and continue with others
        console.warn(`Failed to download attachment ${attachment.filename}: ${attachmentResponse.statusText}`)
        continue
      }

      const attachmentData = (await attachmentResponse.json()) as { data: string; size: number }

      // Decode base64url data to buffer
      // Gmail API returns data in base64url format (URL-safe base64)
      const base64Data = attachmentData.data.replace(/-/g, '+').replace(/_/g, '/')
      const buffer = Buffer.from(base64Data, 'base64')

      downloadedAttachments.push({
        name: attachment.filename,
        data: buffer,
        mimeType: attachment.mimeType,
        size: attachment.size,
      })
    } catch (error) {
      // Continue with other attachments
      console.error(`Error downloading attachment ${attachment.filename}:`, error)
    }
  }

  return downloadedAttachments
}
```

*   **`const downloadedAttachments: GmailAttachment[] = []`**: Initializes an empty array to store the successfully downloaded attachments.
*   **`for (const attachment of attachmentInfo)`**: Loops through each piece of attachment metadata in the `attachmentInfo` array.
*   **`try { ... } catch (error) { ... }`**: Each download attempt is wrapped in a `try...catch` block. If one attachment fails, it won't stop the processing of others.
*   **`const attachmentResponse = await fetch(...)`**: Makes an asynchronous HTTP request using the `fetch` API.
    *   **`` `${GMAIL_API_BASE}/messages/${messageId}/attachments/${attachment.attachmentId}` ``**: Constructs the specific URL for downloading an attachment.
    *   **`headers: { Authorization: \`Bearer ${accessToken}\`, 'Content-Type': 'application/json' }`**: Sets the necessary HTTP headers for authentication (using the `accessToken` in a `Bearer` token) and specifies the expected content type.
*   **`if (!attachmentResponse.ok) { ... continue; }`**: Checks if the HTTP response was successful (`.ok` means status code 2xx). If not, it logs a warning and uses `continue` to move to the next attachment without crashing.
*   **`const attachmentData = (await attachmentResponse.json()) as { data: string; size: number }`**: Parses the JSON response from the API. The Gmail API typically returns an object with a `data` field (containing the base64url encoded file content) and a `size` field.
*   **`const base64Data = attachmentData.data.replace(/-/g, '+').replace(/_/g, '/')`**: **This is a critical step for Gmail API.** Gmail uses a "URL-safe" base64 encoding (often called base64url), where `-` replaces `+` and `_` replaces `/`. Node.js's `Buffer.from` expects standard base64, so these replacements convert base64url to standard base64.
*   **`const buffer = Buffer.from(base64Data, 'base64')`**: Decodes the (now standard) base64 string into a Node.js `Buffer`, which represents the raw binary data of the file.
*   **`downloadedAttachments.push({ ... })`**: Creates a `GmailAttachment` object with the filename, the binary `buffer` data, MIME type, and size, then adds it to the `downloadedAttachments` array.
*   **`catch (error) { ... }`**: If any error occurs during the `fetch` or decoding for a specific attachment, it logs the error and continues the loop.
*   **`return downloadedAttachments`**: Returns the array of all attachments that were successfully downloaded.

---

### `createMessagesSummary` Function

This function takes an array of processed messages (typically from `processMessageForSummary`) and formats them into a single, human-readable summary string.

**Simplified Logic:**
It checks if there are messages. If so, it iterates through them, formatting key details (subject, from, date, snippet) into a list, and then adds a helpful instruction on how to read full messages.

```typescript
export function createMessagesSummary(messages: any[]): string {
```

*   **`export function createMessagesSummary(messages: any[]): string`**: Defines a function to create a summary string for multiple messages.
    *   **`messages: any[]`**: Takes an array of message objects. These objects are expected to have `subject`, `from`, `date`, `snippet`, and `id` properties (e.g., as returned by `processMessageForSummary`).
    *   **`: string`**: Returns a single string representing the summary.

```typescript
  if (messages.length === 0) {
    return 'No messages found.'
  }
```

*   **`if (messages.length === 0)`**: Checks if the input array of messages is empty. If so, it returns a simple "No messages found." string.

```typescript
  let summary = `Found ${messages.length} messages:\n\n`

  messages.forEach((msg, index) => {
    summary += `${index + 1}. Subject: ${msg.subject}\n`
    summary += `   From: ${msg.from}\n`
    summary += `   Date: ${msg.date}\n`
    summary += `   Preview: ${msg.snippet}\n\n`
  })
```

*   **`let summary = \`Found ${messages.length} messages:\n\n\` `**: Initializes a `summary` string using a template literal, stating the total number of messages found.
*   **`messages.forEach((msg, index) => { ... })`**: Iterates over each message in the `messages` array.
    *   **`summary += ...`**: For each message, it appends formatted lines to the `summary` string, including a numbered list item, subject, sender, date, and a preview (snippet). Newlines (`\n`) are used for readability.

```typescript
  summary += `To read full content of a specific message, use the gmail_read tool with messageId: ${messages.map((m) => m.id).join(', ')}`

  return summary
}
```

*   **`summary += \`To read full content ...\` `**: Appends a helpful instruction to the summary. It tells the user how they can read the full content of any of these messages, listing their IDs.
    *   **`messages.map((m) => m.id).join(', ')`**: This part extracts all the `id`s from the messages array and joins them into a comma-separated string.
*   **`return summary`**: Returns the complete, formatted summary string.