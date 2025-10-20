This TypeScript file defines a sophisticated "tool" designed to fetch the content and comprehensive metadata of a file stored in Google Drive. It's particularly clever because it distinguishes between native Google Workspace files (like Docs, Sheets, Slides) and other file types, handling each appropriately to provide readable content.

This "tool" structure is common in systems that integrate with various APIs, allowing developers to define how an external service (like Google Drive) can be interacted with in a standardized, declarative way.

---

### **Purpose of this File**

The primary purpose of this file is to encapsulate the logic required to:

1.  **Retrieve content from a Google Drive file.**
2.  **Intelligently handle Google Workspace files:** For files like Google Docs or Sheets, it automatically *exports* them into a readable format (e.g., plain text or a specified MIME type) rather than just downloading their proprietary binary format.
3.  **Download content from regular files:** For files like PDFs, images, or standard documents, it directly downloads their content.
4.  **Fetch comprehensive file metadata:** It also retrieves important details about the file, such as its name, MIME type, size, creation/modification times, and links.
5.  **Provide a unified interface:** It defines a clear structure for its input parameters (`fileId`, `accessToken`) and its output (`content`, `metadata`), making it easy for other parts of a system to use this Google Drive functionality.

---

### **Detailed Explanation**

Let's break down the code section by section.

#### **Imports and Logger Setup**

```typescript
import { createLogger } from '@/lib/logs/console/logger'
import type {
  GoogleDriveGetContentResponse,
  GoogleDriveToolParams,
} from '@/tools/google_drive/types'
import { DEFAULT_EXPORT_FORMATS, GOOGLE_WORKSPACE_MIME_TYPES } from '@/tools/google_drive/utils'
import type { ToolConfig } from '@/tools/types'

const logger = createLogger('GoogleDriveGetContentTool')
```

*   **`import { createLogger } from '@/lib/logs/console/logger'`**: This line imports a utility function `createLogger`. This function is likely used to create a logging instance, enabling the tool to output messages (information, warnings, errors) during its execution, which is crucial for debugging and monitoring.
*   **`import type { GoogleDriveGetContentResponse, GoogleDriveToolParams, } from '@/tools/google_drive/types'`**: These are TypeScript type imports.
    *   `GoogleDriveToolParams`: Defines the structure of the input parameters this tool expects (e.g., `fileId`, `accessToken`).
    *   `GoogleDriveGetContentResponse`: Defines the structure of the successful output data this tool will return.
*   **`import { DEFAULT_EXPORT_FORMATS, GOOGLE_WORKSPACE_MIME_TYPES } from '@/tools/google_drive/utils'`**: This line imports constants from a utility file.
    *   `DEFAULT_EXPORT_FORMATS`: An object or map that likely defines default MIME types for exporting different Google Workspace file types (e.g., Google Doc exports to 'text/plain').
    *   `GOOGLE_WORKSPACE_MIME_TYPES`: An array or set containing the specific MIME types that identify native Google Workspace files (e.g., `application/vnd.google-apps.document`).
*   **`import type { ToolConfig } from '@/tools/types'`**: This imports a generic TypeScript type `ToolConfig`. This type likely defines the overall structure that any "tool" (like this one) must adhere to, ensuring consistency across different tool implementations.
*   **`const logger = createLogger('GoogleDriveGetContentTool')`**: Here, a specific logger instance is created for this tool, identified by the name `'GoogleDriveGetContentTool'`. This allows logs from this tool to be easily filtered or identified within a larger system.

#### **Tool Configuration Object (`getContentTool`)**

This is the main export of the file, defining the `getContentTool` object, which is of type `ToolConfig`.

```typescript
export const getContentTool: ToolConfig<GoogleDriveToolParams, GoogleDriveGetContentResponse> = {
  id: 'google_drive_get_content',
  name: 'Get Content from Google Drive',
  description:
    'Get content from a file in Google Drive (exports Google Workspace files automatically)',
  version: '1.0',
```

*   **`export const getContentTool: ToolConfig<GoogleDriveToolParams, GoogleDriveGetContentResponse> = { ... }`**: This declares and exports a constant `getContentTool`. It's explicitly typed as `ToolConfig`, specifying that it takes `GoogleDriveToolParams` as input and returns `GoogleDriveGetContentResponse`.
*   **`id: 'google_drive_get_content'`**: A unique identifier for this specific tool.
*   **`name: 'Get Content from Google Drive'`**: A human-readable name for the tool.
*   **`description: 'Get content from a file in Google Drive (exports Google Workspace files automatically)'`**: A brief explanation of what the tool does, highlighting its key feature of handling Google Workspace files.
*   **`version: '1.0'`**: The version of this tool configuration.

#### **OAuth Configuration**

```typescript
  oauth: {
    required: true,
    provider: 'google-drive',
    additionalScopes: ['https://www.googleapis.com/auth/drive.file'],
  },
```

*   **`oauth: { ... }`**: This section defines the OAuth (Open Authorization) requirements for using this tool.
*   **`required: true`**: Indicates that authentication is mandatory to use this tool.
*   **`provider: 'google-drive'`**: Specifies that the OAuth provider is Google Drive.
*   **`additionalScopes: ['https://www.googleapis.com/auth/drive.file']`**: This is a critical security setting. It requests specific permissions (called "scopes") from the user's Google account. `https://www.googleapis.com/auth/drive.file` means the tool needs permission to view and manage files that it has opened or created with the user.

#### **Parameters Definition**

```typescript
  params: {
    accessToken: {
      type: 'string',
      required: true,
      visibility: 'hidden',
      description: 'The access token for the Google Drive API',
    },
    fileId: {
      type: 'string',
      required: true,
      visibility: 'user-only',
      description: 'The ID of the file to get content from',
    },
    mimeType: {
      type: 'string',
      required: false,
      visibility: 'hidden',
      description: 'The MIME type to export Google Workspace files to (optional)',
    },
  },
```

*   **`params: { ... }`**: Defines the input parameters the tool expects. Each parameter is an object describing its properties.
    *   **`accessToken`**:
        *   `type: 'string'`: The expected data type.
        *   `required: true`: This parameter *must* be provided.
        *   `visibility: 'hidden'`: This parameter should not be exposed directly to the end-user (it's likely managed by the system integrating the tool).
        *   `description`: Explains its purpose.
    *   **`fileId`**:
        *   `type: 'string'`: The expected data type.
        *   `required: true`: This parameter *must* be provided.
        *   `visibility: 'user-only'`: This parameter is expected to be provided by the end-user.
        *   `description`: Explains its purpose.
    *   **`mimeType`**:
        *   `type: 'string'`: The expected data type.
        *   `required: false`: This parameter is optional.
        *   `visibility: 'hidden'`: Not typically exposed to the end-user.
        *   `description`: Explains its purpose – allows specifying a custom export format for Google Workspace files.

#### **Request Definition**

```typescript
  request: {
    url: (params) =>
      `https://www.googleapis.com/drive/v3/files/${params.fileId}?fields=id,name,mimeType`,
    method: 'GET',
    headers: (params) => ({
      Authorization: `Bearer ${params.accessToken}`,
    }),
  },
```

*   **`request: { ... }`**: This section defines how the *initial* HTTP request is made to the Google Drive API to get basic metadata about the file.
    *   **`url: (params) => ...`**: A function that constructs the API endpoint URL dynamically using the provided `params`.
        *   `` `https://www.googleapis.com/drive/v3/files/${params.fileId}?fields=id,name,mimeType` ``: This is the Google Drive API endpoint for getting file information. It uses string interpolation to insert the `fileId`. The `?fields=id,name,mimeType` part is crucial for optimization; it tells the API to return *only* these specific fields (ID, name, MIME type), which are sufficient for the initial check.
    *   **`method: 'GET'`**: The HTTP method used for the request.
    *   **`headers: (params) => ({ ... })`**: A function that constructs the HTTP headers for the request.
        *   `Authorization: \`Bearer ${params.accessToken}\``: This sets the `Authorization` header, including the `Bearer` token, which is essential for authenticating the request with the Google Drive API.

#### **Response Transformation (`transformResponse`) - The Core Logic**

This is the most complex part, an asynchronous function that processes the API response from the initial metadata request and then performs further actions (exporting or downloading) based on the file type.

```typescript
  transformResponse: async (response: Response, params?: GoogleDriveToolParams) => {
    try {
      if (!response.ok) {
        const errorDetails = await response.json().catch(() => ({}))
        logger.error('Failed to get file metadata', {
          status: response.status,
          statusText: response.statusText,
          error: errorDetails,
        })
        throw new Error(errorDetails.error?.message || 'Failed to get file metadata')
      }

      const metadata = await response.json()
      const fileId = metadata.id
      const mimeType = metadata.mimeType
      const authHeader = `Bearer ${params?.accessToken || ''}`

      let content: string
```

*   **`transformResponse: async (response: Response, params?: GoogleDriveToolParams) => { ... }`**: An asynchronous function that takes the `Response` object from the initial API call and the original `params` as input.
*   **`try { ... } catch (error: any) { ... }`**: A `try...catch` block wraps the entire logic for robust error handling. If any unhandled error occurs, it's caught, logged, and re-thrown.
*   **`if (!response.ok) { ... }`**: Checks if the initial HTTP response was successful (`response.ok` is `true` for 2xx status codes).
    *   **`const errorDetails = await response.json().catch(() => ({}))`**: If the response was not OK, it tries to parse the response body as JSON to get more error details. The `.catch(() => ({}))` ensures it doesn't fail if the error response isn't valid JSON.
    *   **`logger.error(...)`**: Logs the error with status code, status text, and any detailed error message.
    *   **`throw new Error(...)`**: Throws a new error, including the message from the API error details or a generic one.
*   **`const metadata = await response.json()`**: If the initial response was OK, it parses the response body as JSON to get the basic file metadata (ID, name, MIME type).
*   **`const fileId = metadata.id`**, **`const mimeType = metadata.mimeType`**: Extracts the `fileId` and `mimeType` from the received metadata.
*   **`const authHeader = \`Bearer ${params?.accessToken || ''}\``**: Reconstructs the `Authorization` header for subsequent API calls, using the access token from the original `params`. The `|| ''` is a safeguard if `accessToken` is somehow missing.
*   **`let content: string`**: Declares a variable `content` which will store the extracted file content.

---

#### **Complex Logic: Handling Google Workspace Files vs. Regular Files**

This is the core decision-making part of the tool.

```typescript
      if (GOOGLE_WORKSPACE_MIME_TYPES.includes(mimeType)) {
        const exportFormat = params?.mimeType || DEFAULT_EXPORT_FORMATS[mimeType] || 'text/plain'
        logger.info('Exporting Google Workspace file', {
          fileId,
          mimeType,
          exportFormat,
        })

        const exportResponse = await fetch(
          `https://www.googleapis.com/drive/v3/files/${fileId}/export?mimeType=${encodeURIComponent(exportFormat)}`,
          {
            headers: {
              Authorization: authHeader,
            },
          }
        )

        if (!exportResponse.ok) {
          const exportError = await exportResponse.json().catch(() => ({}))
          logger.error('Failed to export file', {
            status: exportResponse.status,
            statusText: exportResponse.statusText,
            error: exportError,
          })
          throw new Error(exportError.error?.message || 'Failed to export Google Workspace file')
        }

        content = await exportResponse.text()
      } else {
        logger.info('Downloading regular file', {
          fileId,
          mimeType,
        })

        const downloadResponse = await fetch(
          `https://www.googleapis.com/drive/v3/files/${fileId}?alt=media`,
          {
            headers: {
              Authorization: authHeader,
            },
          }
        )

        if (!downloadResponse.ok) {
          const downloadError = await downloadResponse.json().catch(() => ({}))
          logger.error('Failed to download file', {
            status: downloadResponse.status,
            statusText: downloadResponse.statusText,
            error: downloadError,
          })
          throw new Error(downloadError.error?.message || 'Failed to download file')
        }

        content = await downloadResponse.text()
      }
```

*   **`if (GOOGLE_WORKSPACE_MIME_TYPES.includes(mimeType)) { ... }`**: This condition checks if the file's `mimeType` is present in the `GOOGLE_WORKSPACE_MIME_TYPES` array (e.g., if it's a Google Doc, Sheet, or Slide).
    *   **`const exportFormat = params?.mimeType || DEFAULT_EXPORT_FORMATS[mimeType] || 'text/plain'`**: Determines the target MIME type for exporting the Google Workspace file.
        *   It first tries to use a `mimeType` provided in the tool's input `params`.
        *   If not provided, it falls back to `DEFAULT_EXPORT_FORMATS[mimeType]`, which maps Google Workspace MIME types to common export formats (e.g., `application/vnd.google-apps.document` might map to `text/plain` or `application/vnd.openxmlformats-officedocument.wordprocessingml.document`).
        *   If neither is available, it defaults to `'text/plain'`.
    *   **`logger.info('Exporting Google Workspace file', { ... })`**: Logs an informational message about the export process.
    *   **`const exportResponse = await fetch(...)`**: Makes a `fetch` request to the Google Drive API's `/export` endpoint.
        *   `` `https://www.googleapis.com/drive/v3/files/${fileId}/export?mimeType=${encodeURIComponent(exportFormat)}` ``: This URL is specifically for exporting Google Workspace files. The `mimeType` query parameter specifies the desired output format. `encodeURIComponent` ensures the MIME type is correctly URL-encoded.
        *   `headers: { Authorization: authHeader }`: Includes the authentication header.
    *   **`if (!exportResponse.ok) { ... }`**: Error handling for the export request, similar to the initial metadata request. Logs the error and throws an exception if the export fails.
    *   **`content = await exportResponse.text()`**: If the export is successful, the content is read as plain text.
*   **`else { ... }`**: This block executes if the file is *not* a Google Workspace file (e.g., a PDF, JPEG, TXT file).
    *   **`logger.info('Downloading regular file', { ... })`**: Logs an informational message about downloading a regular file.
    *   **`const downloadResponse = await fetch(...)`**: Makes a `fetch` request to directly download the file content.
        *   `` `https://www.googleapis.com/drive/v3/files/${fileId}?alt=media` ``: This URL is used to download the raw content of a file. The `?alt=media` query parameter signals to the API that the actual file content should be returned.
        *   `headers: { Authorization: authHeader }`: Includes the authentication header.
    *   **`if (!downloadResponse.ok) { ... }`**: Error handling for the download request, similar to previous error checks.
    *   **`content = await downloadResponse.text()`**: If the download is successful, the content is read as plain text.

---

#### **Fetching Full Metadata (Best Effort)**

```typescript
      const metadataResponse = await fetch(
        `https://www.googleapis.com/drive/v3/files/${fileId}?fields=id,name,mimeType,webViewLink,webContentLink,size,createdTime,modifiedTime,parents`,
        {
          headers: {
            Authorization: authHeader,
          },
        }
      )

      if (!metadataResponse.ok) {
        logger.warn('Failed to get full metadata, using partial metadata', {
          status: metadataResponse.status,
          statusText: metadataResponse.statusText,
        })
      } else {
        const fullMetadata = await metadataResponse.json()
        Object.assign(metadata, fullMetadata)
      }
```

*   **`const metadataResponse = await fetch(...)`**: After getting the file content, another `fetch` request is made. This time, it's to retrieve a more comprehensive set of metadata fields about the file.
    *   `?fields=id,name,mimeType,webViewLink,webContentLink,size,createdTime,modifiedTime,parents`: This specifies a longer list of desired metadata fields, including links, size, timestamps, and parent folders.
*   **`if (!metadataResponse.ok) { ... }`**: Checks if fetching the full metadata was successful.
    *   **`logger.warn('Failed to get full metadata, using partial metadata', { ... })`**: If it fails, a warning is logged, but the process *continues*. This is a "best-effort" approach; the tool will still return content and whatever partial metadata it initially got, rather than failing entirely.
*   **`else { ... }`**: If fetching full metadata is successful.
    *   **`const fullMetadata = await metadataResponse.json()`**: Parses the full metadata.
    *   **`Object.assign(metadata, fullMetadata)`**: Merges the `fullMetadata` into the `metadata` object that was created from the initial API call. This effectively updates `metadata` with richer information.

---

#### **Return Value and Final Error Handling**

```typescript
      return {
        success: true,
        output: {
          content,
          metadata: {
            id: metadata.id,
            name: metadata.name,
            mimeType: metadata.mimeType,
            webViewLink: metadata.webViewLink,
            webContentLink: metadata.webContentLink,
            size: metadata.size,
            createdTime: metadata.createdTime,
            modifiedTime: metadata.modifiedTime,
            parents: metadata.parents,
          },
        },
      }
    } catch (error: any) {
      logger.error('Error in transform response', {
        error: error.message,
        stack: error.stack,
      })
      throw error
    }
  },
```

*   **`return { success: true, output: { content, metadata: { ... } } }`**: If everything succeeds, the function returns an object indicating success. The `output` property contains:
    *   **`content`**: The text content of the file (either exported or downloaded).
    *   **`metadata`**: An object containing all the collected metadata fields. Notice how it explicitly lists the fields to be returned, ensuring a consistent output structure.
*   **`catch (error: any) { ... }`**: This is the outer `catch` block for the entire `transformResponse` function.
    *   **`logger.error('Error in transform response', { ... })`**: Logs any unexpected errors that occurred during the process, including the error message and stack trace.
    *   **`throw error`**: Re-throws the error, allowing the calling system to handle it further.

---

#### **Outputs Definition**

```typescript
  outputs: {
    content: {
      type: 'string',
      description: 'File content as text (Google Workspace files are exported)',
    },
    metadata: {
      type: 'json',
      description: 'File metadata including ID, name, MIME type, and links',
    },
  },
}
```

*   **`outputs: { ... }`**: This section declaratively describes the structure of the data that the tool will return upon successful execution. This is useful for systems that need to understand the tool's capabilities without running it.
    *   **`content`**:
        *   `type: 'string'`: The content is returned as a string.
        *   `description`: Clarifies that Google Workspace files are exported.
    *   **`metadata`**:
        *   `type: 'json'`: The metadata is returned as a JSON object.
        *   `description`: Lists the key fields included in the metadata.

---

### **Simplifying Complex Logic**

The most complex part of this file is the `transformResponse` function, specifically how it handles different file types. Here's a simplified breakdown:

1.  **Initial Check (Metadata):** The tool first makes a quick API call to Google Drive just to get the `mimeType` of the requested file. This is crucial for deciding the next step. If this initial call fails, it immediately stops and reports an error.
2.  **Smart File Type Detection:**
    *   **"Is this a Google Workspace file?"** It checks if the `mimeType` matches any known Google Workspace types (like a Google Doc, Sheet, etc.).
    *   **If YES (Google Workspace File):**
        *   It figures out the best format to *export* the file to (e.g., convert a Google Doc into plain text or HTML). It can use a user-specified format, a predefined default, or fall back to plain text.
        *   It then makes a *second* API call to Google Drive's `/export` endpoint, specifically asking for the file in that chosen export format.
        *   If the export fails, it reports an error.
        *   If successful, it captures the exported text content.
    *   **If NO (Regular File):**
        *   It makes a *second* API call to Google Drive's `/download` endpoint, asking for the raw file content.
        *   If the download fails, it reports an error.
        *   If successful, it captures the raw text content (assuming it's a text-like file, or it will be a string representation of the binary).
3.  **Getting Full Details (Optional but Recommended):** After getting the file content, it makes a *third* API call to get a broader range of file information (like web links, size, dates, parent folders). If this call fails, it's treated as a warning, and the tool continues, using the partial metadata it already has.
4.  **Final Output:** Finally, it combines the extracted file content and all the collected metadata into a single, structured output.

In essence, it's a multi-step process with conditional logic and robust error handling to ensure it gets the right content in the right format, along with all relevant details, regardless of the Google Drive file type.