This TypeScript file is a comprehensive set of **type definitions** used for building an application that interacts with Gmail. It defines the expected shapes of data for:

1.  **Input Parameters:** What information your application needs to provide to perform various Gmail operations (like sending, reading, or searching emails).
2.  **Output Responses:** What kind of data your application will receive back from these Gmail operations.
3.  **Internal Structures:** How raw Gmail messages are represented and how processed attachments are structured.

Essentially, this file acts as a **blueprint** or **contract** for interacting with a Gmail service or API within a larger application framework (likely a "tools" system, as indicated by the `ToolResponse` import). It ensures that data passed around related to Gmail operations is consistent and type-safe.

---

### Understanding Core TypeScript Concepts (Quick Refresher)

Before diving into each line, here's a quick explanation of some common TypeScript terms you'll see:

*   **`interface`**: Defines the "shape" that an object must have. It's like a contract specifying what properties an object should contain and what their types should be.
*   **`type`**: Creates a new name for an existing type or a combination of types (like a "union" of types). It's more flexible than `interface`.
*   **`export`**: Makes a type or interface available for use in other files in your project.
*   **`extends`**: Allows an interface to inherit properties from another interface, adding new ones or overriding existing ones. This promotes code reuse and consistency.
*   **`?` (Optional Property)**: A question mark after a property name (e.g., `cc?: string`) means that property is optional and doesn't *have* to be present on the object.
*   **`[]` (Array Type)**: Indicates an array of a specific type (e.g., `string[]` means an array of strings).
*   **`|` (Union Type)**: Combines multiple types, meaning a value can be *any one* of the specified types (e.g., `TypeA | TypeB` means it can be `TypeA` or `TypeB`).
*   **`import type`**: This is a type-only import. It tells TypeScript to import a type definition but ensures that no JavaScript code is generated for this import, keeping the compiled output clean.

---

### Detailed Explanation of Each Line

Let's break down the code section by section.

#### 1. Importing External Types

```typescript
import type { ToolResponse } from '@/tools/types'
```

*   **`import type { ToolResponse } from '@/tools/types'`**: This line imports a type named `ToolResponse` from a file located at `src/tools/types.ts` (the `@/` is typically an alias configured in `tsconfig.json` for the `src` directory). This indicates that the `GmailToolResponse` (defined later) will build upon a more generic `ToolResponse` structure, implying a broader "tooling" framework.

#### 2. Base Parameters for Gmail Operations

```typescript
// Base parameters shared by all operations
interface BaseGmailParams {
  accessToken: string
}
```

*   **`interface BaseGmailParams`**: Defines an interface named `BaseGmailParams`. This will serve as a foundational set of parameters for all specific Gmail operations.
*   **`accessToken: string`**: Every Gmail operation requires authentication. This property ensures that a `string` representing an OAuth 2.0 access token is provided, allowing the application to act on behalf of a user.

#### 3. Parameters for Specific Gmail Operations

These interfaces extend `BaseGmailParams`, adding fields specific to each operation.

```typescript
// Send operation parameters
export interface GmailSendParams extends BaseGmailParams {
  to: string
  cc?: string
  bcc?: string
  subject: string
  body: string
}
```

*   **`export interface GmailSendParams extends BaseGmailParams`**: Defines parameters for sending an email. It inherits `accessToken` from `BaseGmailParams`.
*   **`to: string`**: The primary recipient's email address (required).
*   **`cc?: string`**: An optional carbon copy recipient's email address.
*   **`bcc?: string`**: An optional blind carbon copy recipient's email address.
*   **`subject: string`**: The subject line of the email (required).
*   **`body: string`**: The main content of the email (required).

```typescript
// Read operation parameters
export interface GmailReadParams extends BaseGmailParams {
  messageId: string
  folder: string
  unreadOnly?: boolean
  maxResults?: number
  includeAttachments?: boolean
}
```

*   **`export interface GmailReadParams extends BaseGmailParams`**: Defines parameters for reading a specific email. It also inherits `accessToken`.
*   **`messageId: string`**: The unique identifier of the email message to be read (required).
*   **`folder: string`**: The folder (e.g., "INBOX", "SENT") where the message is located (required).
*   **`unreadOnly?: boolean`**: An optional flag. If `true`, it might filter results to only include unread messages.
*   **`maxResults?: number`**: An optional limit on the number of results to return (though for `read` it's usually one message, this might apply if reading a thread or a folder).
*   **`includeAttachments?: boolean`**: An optional flag to specify whether attachment data should be included in the response when reading a message.

```typescript
// Search operation parameters
export interface GmailSearchParams extends BaseGmailParams {
  query: string
  maxResults?: number
}
```

*   **`export interface GmailSearchParams extends BaseGmailParams`**: Defines parameters for searching emails. It inherits `accessToken`.
*   **`query: string`**: The search string (e.g., "from:alice subject:report"). This is similar to what you'd type into the Gmail search bar (required).
*   **`maxResults?: number`**: An optional limit on the number of search results to return.

#### 4. Union Type for All Gmail Parameters (Simplifying Logic)

```typescript
// Union type for all Gmail tool parameters
export type GmailToolParams = GmailSendParams | GmailReadParams | GmailSearchParams
```

*   **`export type GmailToolParams`**: This is a **union type**. It defines `GmailToolParams` as being *either* a `GmailSendParams` object, *or* a `GmailReadParams` object, *or* a `GmailSearchParams` object.
*   **Simplification**: Instead of creating separate functions or conditional logic for handling each parameter type, a single function can accept `GmailToolParams`. TypeScript will then help ensure that the correct properties are accessed based on the actual type of the object at runtime (e.g., using type guards). This makes the code more flexible and easier to maintain.

#### 5. Response Metadata Structures

These interfaces define the supplementary information (metadata) that comes back with a response, varying based on the operation.

```typescript
// Response metadata
interface BaseGmailMetadata {
  id?: string
  threadId?: string
  labelIds?: string[]
}
```

*   **`interface BaseGmailMetadata`**: A base interface for common metadata fields.
*   **`id?: string`**: The optional unique ID of a message.
*   **`threadId?: string`**: The optional ID of the conversation thread this message belongs to.
*   **`labelIds?: string[]`**: An optional array of label IDs (e.g., "INBOX", "STARRED", custom labels) applied to the message.

```typescript
interface EmailMetadata extends BaseGmailMetadata {
  from?: string
  to?: string
  subject?: string
  date?: string
  hasAttachments?: boolean
  attachmentCount?: number
}
```

*   **`interface EmailMetadata extends BaseGmailMetadata`**: Metadata specific to a single email (e.g., returned after a `read` or `send` operation). It inherits `id`, `threadId`, and `labelIds`.
*   **`from?: string`**: The optional sender's email address.
*   **`to?: string`**: The optional recipient's email address.
*   **`subject?: string`**: The optional subject of the email.
*   **`date?: string`**: The optional date the email was sent or received.
*   **`hasAttachments?: boolean`**: An optional flag indicating if the email contains attachments.
*   **`attachmentCount?: number`**: An optional count of attachments in the email.

```typescript
interface SearchMetadata extends BaseGmailMetadata {
  results: Array<{
    id: string
    threadId: string
  }>
}
```

*   **`interface SearchMetadata extends BaseGmailMetadata`**: Metadata specific to a search operation. It inherits `id`, `threadId`, and `labelIds` (though these would likely be undefined or less relevant for the overall search result, more for individual `results` items).
*   **`results: Array<{ id: string; threadId: string }>`**: This is the core of search metadata. It's an array where each element is an object representing a found message, containing its required `id` and `threadId`.

#### 6. Overall Gmail Tool Response Structure

```typescript
// Response format
export interface GmailToolResponse extends ToolResponse {
  output: {
    content: string
    metadata: EmailMetadata | SearchMetadata
    attachments?: GmailAttachment[]
  }
}
```

*   **`export interface GmailToolResponse extends ToolResponse`**: This defines the complete structure for a response coming back from the Gmail "tool." It extends the generic `ToolResponse` type imported earlier, meaning it will include all properties from `ToolResponse` plus its own specific `output` property.
*   **`output: { ... }`**: This object holds the actual data resulting from the Gmail operation.
    *   **`content: string`**: The primary textual output, which could be the body of a read email, a summary of search results, or a confirmation message for a sent email.
    *   **`metadata: EmailMetadata | SearchMetadata`**: This is another **union type**, similar to `GmailToolParams`. The `metadata` property can either be `EmailMetadata` (for operations like read/send) or `SearchMetadata` (for search operations). This allows the response to flexibly carry relevant metadata based on the initial request.
    *   **`attachments?: GmailAttachment[]`**: An optional array of `GmailAttachment` objects. These would typically be present after a `read` operation if `includeAttachments` was set to `true`.

#### 7. Raw Gmail Message Structure (from API)

```typescript
// Email Message Interface
export interface GmailMessage {
  id: string
  threadId: string
  labelIds: string[]
  snippet: string
  payload: {
    headers: Array<{
      name: string
      value: string
    }>
    body: {
      data?: string
      attachmentId?: string
      size?: number
    }
    parts?: Array<{
      mimeType: string
      filename?: string
      body: {
        data?: string
        attachmentId?: string
        size?: number
      }
      parts?: Array<any>
    }>
  }
}
```

*   **`export interface GmailMessage`**: This interface represents the *raw, detailed structure* of an email message, likely as it's received directly from a low-level Gmail API. It's designed to capture the full complexity of email formats (e.g., multipart messages).
*   **`id: string`**: The unique ID of the message (required).
*   **`threadId: string`**: The ID of the conversation thread it belongs to (required).
*   **`labelIds: string[]`**: An array of labels applied to the message (required).
*   **`snippet: string`**: A short snippet of the message content (required).
*   **`payload: { ... }`**: This is the most complex part, describing the actual content structure of the email.
    *   **`headers: Array<{ name: string; value: string }>`**: An array of email headers (e.g., "From", "To", "Subject", "Date").
    *   **`body: { data?: string; attachmentId?: string; size?: number }`**: Represents the main body part of an email if it's a simple, single-part message.
        *   **`data?: string`**: Base64URL encoded string of the body content.
        *   **`attachmentId?: string`**: If this body part *is* an attachment, its ID.
        *   **`size?: number`**: Size of the body part in bytes.
    *   **`parts?: Array<{ ... }>`**: This is where it gets recursive and complex! Emails are often composed of multiple "parts" (e.g., a plain text version, an HTML version, and various attachments). This array holds those individual parts.
        *   **`mimeType: string`**: The MIME type of this specific part (e.g., "text/plain", "text/html", "image/jpeg").
        *   **`filename?: string`**: If this part is an attachment, its filename.
        *   **`body: { data?: string; attachmentId?: string; size?: number }`**: The content for *this specific part*. It has the same structure as the main `payload.body`.
        *   **`parts?: Array<any>`**: This is the **recursive part**. A part can itself contain *other parts* (e.g., an HTML part might embed images, or a "multipart/alternative" part might contain both text/plain and text/html versions). This `Array<any>` indicates it could be any structure, typically another `GmailMessage.payload.parts` structure.
*   **Simplification of `GmailMessage`:** This interface truly reflects the internal, nested nature of email messages. While complex, it's essential for accurately parsing and handling all types of email content, including those with multiple text formats and various attachments.

#### 8. Processed Gmail Attachment Interface

```typescript
// Gmail Attachment Interface (for processed attachments)
export interface GmailAttachment {
  name: string
  data: Buffer
  mimeType: string
  size: number
}
```

*   **`export interface GmailAttachment`**: This interface defines a **simplified and processed** representation of an attachment. This is what your application would likely work with *after* it has extracted an attachment from the raw `GmailMessage` payload.
*   **`name: string`**: The filename of the attachment.
*   **`data: Buffer`**: The actual binary content of the attachment. `Buffer` is a Node.js class representing raw binary data, commonly used for files and streams.
*   **`mimeType: string`**: The MIME type of the attachment (e.g., "image/png", "application/pdf").
*   **`size: number`**: The size of the attachment in bytes.
*   **Simplification**: Compared to how attachments are nested and encoded within `GmailMessage.payload`, this `GmailAttachment` is a much cleaner, ready-to-use format. It implies that there's some processing logic elsewhere that takes the raw `GmailMessage` and converts its attachment parts into this `GmailAttachment` format.

---

### In Summary: The Data Flow

This file sets up the "language" for your application to talk to Gmail:

1.  **You prepare a `GmailToolParams`** (e.g., `GmailSendParams` with `to`, `subject`, `body`).
2.  **You send it to your Gmail "tool."**
3.  **The tool (internally) might deal with raw `GmailMessage` structures** from the Gmail API.
4.  **The tool processes the response**, potentially extracting raw attachment parts into `GmailAttachment` objects.
5.  **The tool returns a `GmailToolResponse`** which contains structured `output` with `content`, appropriate `metadata` (either `EmailMetadata` or `SearchMetadata`), and any processed `attachments`.

This robust set of type definitions ensures clarity, type-safety, and maintainability for any code interacting with Gmail operations within your project.