This file defines a "tool" for uploading files to Google Drive, intended to be integrated into a larger system (like an AI agent framework or a workflow automation platform). It encapsulates all the necessary information for the system to understand how to interact with the Google Drive API for file uploads.

Let's break down the code:

---

## **Purpose of this File**

This TypeScript file defines a `ToolConfig` object named `uploadTool`. This object acts as a complete blueprint for an "Upload to Google Drive" functionality. It specifies:

1.  **What the tool does**: Uploads a file to Google Drive.
2.  **How to use it**: What parameters it accepts (e.g., file name, content, access token).
3.  **Authentication requirements**: It needs Google Drive OAuth.
4.  **How to make the API request**: The URL, method, headers, and body for the initial file creation.
5.  **How to process the API response**: A complex `transformResponse` function that handles the multi-step Google Drive upload process, including creating the file, uploading content, converting Google Workspace formats, and fetching final metadata.
6.  **What output it produces**: Metadata about the newly uploaded file (ID, links, etc.).

In essence, this file provides a standardized, declarative way to expose a Google Drive upload capability to a system that can interpret `ToolConfig` objects.

---

## **Simplified Complex Logic**

The most complex part of this file is the `transformResponse` function. Google Drive file uploads are not a single API call for all file types, especially for Google Workspace documents (like Docs, Sheets, Slides). Here's a simplified explanation of its logic:

1.  **Step 1: Create a Placeholder File (Metadata Only)**
    *   The `request` property handles the *initial* API call. It tells Google Drive: "Hey, I want to create a file named X, with MIME type Y, in folder Z."
    *   Google Drive responds by creating an empty file and giving it an `ID`. This is just the *metadata* for the file; no content is uploaded yet.

2.  **Step 2: Prepare the Content (Special Handling for Google Sheets)**
    *   The `transformResponse` function first receives the response from Step 1.
    *   It checks if the user wants to upload a Google Sheet (`application/vnd.google-apps.spreadsheet`). If so, it takes the provided content (which might be in various formats) and converts it into a standard CSV format using `handleSheetsFormat`. This is crucial because Google Sheets often prefers CSV for content uploads.
    *   For other file types, the content is used as-is.

3.  **Step 3: Upload the Actual Content (using `PATCH`)**
    *   Now that we have the `fileId` (from Step 1) and the `preparedContent` (from Step 2), a *second* API call is made.
    *   This is a `PATCH` request to a special `/upload/drive/v3/files/{fileId}` endpoint. This call *pours* the actual file content into the empty placeholder file created in Step 1.
    *   Crucially, if you're uploading content that will be *converted* into a Google Workspace format (e.g., a `.xlsx` file becoming a Google Sheet), Google Drive needs to know the *original* MIME type of the content being uploaded, which might be different from the *target* Google Workspace MIME type. The code handles this by mapping `GOOGLE_WORKSPACE_MIME_TYPES` to their `SOURCE_MIME_TYPES` if necessary.

4.  **Step 4: Fix File Name for Google Workspace Conversions (If Needed)**
    *   Sometimes, when you upload a file that gets *converted* into a Google Workspace format (e.g., uploading a `.docx` file and it becomes a Google Doc), Google Drive might reset the file name during the conversion process.
    *   If the target MIME type was a Google Workspace type, a *third* API call (another `PATCH`) is made to explicitly set the `name` of the file again, ensuring it retains the desired filename after conversion.

5.  **Step 5: Get Final File Details**
    *   Finally, after all uploads and potential name fixes, a *fourth* API call is made to retrieve the complete and up-to-date metadata for the newly created and populated file, including important links (`webViewLink`, `webContentLink`).

6.  **Error Handling**: At each step, the code checks if the API call was successful (`response.ok`). If not, it logs the error and throws an exception, preventing further steps from executing with invalid data.

---

## **Line-by-Line Explanation**

### Imports

```typescript
import { createLogger } from '@/lib/logs/console/logger'
import type { GoogleDriveToolParams, GoogleDriveUploadResponse } from '@/tools/google_drive/types'
import {
  GOOGLE_WORKSPACE_MIME_TYPES,
  handleSheetsFormat,
  SOURCE_MIME_TYPES,
} from '@/tools/google_drive/utils'
import type { ToolConfig } from '@/tools/types'
```

*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a utility function `createLogger` from a local logging library. This is used to create a logger instance for this specific tool, allowing for structured logging of its operations.
*   `import type { GoogleDriveToolParams, GoogleDriveUploadResponse } from '@/tools/google_drive/types'`: Imports TypeScript type definitions for the input parameters (`GoogleDriveToolParams`) and the expected output (`GoogleDriveUploadResponse`) of this tool. Using `type` ensures these imports are only used for type checking and don't contribute to the runtime bundle size.
*   `import { GOOGLE_WORKSPACE_MIME_TYPES, handleSheetsFormat, SOURCE_MIME_TYPES, } from '@/tools/google_drive/utils'`: Imports several utilities specific to Google Drive:
    *   `GOOGLE_WORKSPACE_MIME_TYPES`: An array containing MIME types for Google's native document formats (e.g., Google Docs, Google Sheets).
    *   `handleSheetsFormat`: A function to process and potentially convert content specifically for Google Sheets.
    *   `SOURCE_MIME_TYPES`: A mapping from Google Workspace MIME types back to common source MIME types (e.g., `application/vnd.google-apps.spreadsheet` maps to `text/csv` for content upload).
*   `import type { ToolConfig } from '@/tools/types'`: Imports the base `ToolConfig` type, which this `uploadTool` constant will conform to. This type defines the expected structure for any tool within the system.

### Logger Initialization

```typescript
const logger = createLogger('GoogleDriveUploadTool')
```

*   `const logger = createLogger('GoogleDriveUploadTool')`: Creates a logger instance specifically named `'GoogleDriveUploadTool'`. This helps in tracing logs back to their source when debugging.

### Tool Configuration (`uploadTool`)

```typescript
export const uploadTool: ToolConfig<GoogleDriveToolParams, GoogleDriveUploadResponse> = {
  // ... configuration properties ...
}
```

*   `export const uploadTool: ToolConfig<GoogleDriveToolParams, GoogleDriveUploadResponse> = { ... }`: Declares and exports a constant `uploadTool`. It's explicitly typed as `ToolConfig`, specifying that it takes `GoogleDriveToolParams` as input and returns `GoogleDriveUploadResponse` as output. This ensures type safety and consistency with the overall tool framework.

#### Basic Tool Metadata

```typescript
  id: 'google_drive_upload',
  name: 'Upload to Google Drive',
  description: 'Upload a file to Google Drive',
  version: '1.0',
```

*   `id: 'google_drive_upload'`: A unique identifier for this tool within the system.
*   `name: 'Upload to Google Drive'`: A human-readable name for the tool, used in user interfaces or documentation.
*   `description: 'Upload a file to Google Drive'`: A brief explanation of what the tool does.
*   `version: '1.0'`: The version number of this tool definition.

#### OAuth Configuration

```typescript
  oauth: {
    required: true,
    provider: 'google-drive',
    additionalScopes: ['https://www.googleapis.com/auth/drive.file'],
  },
```

*   `oauth: { ... }`: Specifies the OAuth requirements for this tool.
*   `required: true`: Indicates that OAuth authentication is mandatory for this tool to function.
*   `provider: 'google-drive'`: Identifies the OAuth provider to use (in this case, Google Drive). This helps the system route authentication requests correctly.
*   `additionalScopes: ['https://www.googleapis.com/auth/drive.file']`: Specifies additional Google API scopes required. `drive.file` grants permission to read, write, and delete only the files that this application creates or opens, which is appropriate for an upload tool.

#### Parameters Definition

```typescript
  params: {
    accessToken: { /* ... */ },
    fileName: { /* ... */ },
    content: { /* ... */ },
    mimeType: { /* ... */ },
    folderSelector: { /* ... */ },
    folderId: { /* ... */ },
  },
```

*   `params: { ... }`: Defines the input parameters the tool expects. Each parameter has a type, requirement status, visibility, and description.

    *   `accessToken`:
        *   `type: 'string'`: Expects a string.
        *   `required: true`: Must be provided.
        *   `visibility: 'hidden'`: Should not be directly exposed to the end-user or an LLM, typically managed by the system.
        *   `description: 'The access token for the Google Drive API'`: Explains its purpose.
    *   `fileName`:
        *   `type: 'string'`: Expects a string.
        *   `required: true`: Must be provided.
        *   `visibility: 'user-or-llm'`: Can be provided by a human user or an LLM.
        *   `description: 'The name of the file to upload'`: Explains its purpose.
    *   `content`:
        *   `type: 'string'`: Expects a string (representing the file's content).
        *   `required: true`: Must be provided.
        *   `visibility: 'user-or-llm'`: Can be provided by a human user or an LLM.
        *   `description: 'The content of the file to upload'`: Explains its purpose.
    *   `mimeType`:
        *   `type: 'string'`: Expects a string.
        *   `required: false`: Optional. If not provided, `text/plain` will be assumed.
        *   `visibility: 'hidden'`: Typically managed internally or inferred, not directly by a user/LLM.
        *   `description: 'The MIME type of the file to upload'`: Explains its purpose.
    *   `folderSelector`:
        *   `type: 'string'`: Expects a string (e.g., a folder name or path).
        *   `required: false`: Optional.
        *   `visibility: 'user-only'`: Primarily for human users (e.g., via a UI dropdown).
        *   `description: 'Select the folder to upload the file to'`: Explains its purpose.
    *   `folderId`:
        *   `type: 'string'`: Expects a string (the actual Google Drive folder ID).
        *   `required: false`: Optional.
        *   `visibility: 'hidden'`: Intended for internal system use, not direct user input.
        *   `description: 'The ID of the folder to upload the file to (internal use)'`: Explains its purpose.

#### Request Configuration

```typescript
  request: {
    url: 'https://www.googleapis.com/drive/v3/files?supportsAllDrives=true',
    method: 'POST',
    headers: (params) => ({
      Authorization: `Bearer ${params.accessToken}`,
      'Content-Type': 'application/json',
    }),
    body: (params) => {
      const metadata: {
        name: string | undefined
        mimeType: string
        parents?: string[]
      } = {
        name: params.fileName,
        mimeType: params.mimeType || 'text/plain',
      }

      const parentFolderId = params.folderSelector || params.folderId
      if (parentFolderId && parentFolderId.trim() !== '') {
        metadata.parents = [parentFolderId]
      }

      return metadata
    },
  },
```

*   `request: { ... }`: Defines how the initial API request should be constructed.
    *   `url: 'https://www.googleapis.com/drive/v3/files?supportsAllDrives=true'`: The base URL for creating new files in Google Drive. `supportsAllDrives=true` is for shared drives compatibility.
    *   `method: 'POST'`: Specifies that an HTTP POST request should be used to create the file.
    *   `headers: (params) => ({ ... })`: A function that generates the HTTP headers for the request, using the provided `params`.
        *   `Authorization: \`Bearer ${params.accessToken}\``: Sets the `Authorization` header with the provided access token, authenticating the request.
        *   `'Content-Type': 'application/json'`: Indicates that the request body will be in JSON format.
    *   `body: (params) => { ... }`: A function that generates the HTTP request body (the file metadata) from the provided `params`.
        *   `const metadata: { name: string | undefined; mimeType: string; parents?: string[]; } = { ... }`: Defines an object `metadata` to hold the file's properties.
        *   `name: params.fileName`: Sets the file's name from the input parameters.
        *   `mimeType: params.mimeType || 'text/plain'`: Sets the file's MIME type, defaulting to `text/plain` if not specified.
        *   `const parentFolderId = params.folderSelector || params.folderId`: Determines the parent folder ID, prioritizing `folderSelector` if available, otherwise using `folderId`.
        *   `if (parentFolderId && parentFolderId.trim() !== '') { metadata.parents = [parentFolderId] }`: If a valid `parentFolderId` is found, it's added to the `parents` array in the metadata, indicating where the file should be created.
        *   `return metadata`: The constructed metadata object is returned as the request body.

#### Response Transformation (`transformResponse`)

```typescript
  transformResponse: async (response: Response, params?: GoogleDriveToolParams) => {
    try {
      // ... detailed logic for handling response and subsequent requests ...
    } catch (error: any) {
      logger.error('Error in upload transformation', {
        error: error.message,
        stack: error.stack,
      })
      throw error
    }
  },
```

*   `transformResponse: async (response: Response, params?: GoogleDriveToolParams) => { ... }`: This is an `async` function that processes the initial API response (from the `request` step) and performs subsequent actions, ultimately returning the tool's output. It takes the raw `Response` object and the original `params` as input.
*   `try { ... } catch (error: any) { ... }`: A `try...catch` block wraps the entire transformation logic to catch and log any errors that occur during the complex multi-step process.

**Inside `transformResponse`'s `try` block:**

1.  **Initial File Creation Response Handling**
    ```typescript
      const data = await response.json()

      if (!response.ok) {
        logger.error('Failed to create file in Google Drive', {
          status: response.status,
          statusText: response.statusText,
          data,
        })
        throw new Error(data.error?.message || 'Failed to create file in Google Drive')
      }
    ```
    *   `const data = await response.json()`: Parses the JSON body of the initial response from Google Drive. This response should contain the `id` of the newly created (but empty) file.
    *   `if (!response.ok)`: Checks if the HTTP response status indicates success (e.g., 2xx).
    *   `logger.error(...)`: If not successful, an error is logged with relevant details (status, status text, and the response data).
    *   `throw new Error(...)`: An error is thrown, halting execution and providing a user-friendly error message.

2.  **Extracting Key Information**
    ```typescript
      const fileId = data.id
      const requestedMimeType = params?.mimeType || 'text/plain'
      const authHeader =
        response.headers.get('Authorization') || `Bearer ${params?.accessToken || ''}`
    ```
    *   `const fileId = data.id`: Extracts the `id` of the newly created file from the response data.
    *   `const requestedMimeType = params?.mimeType || 'text/plain'`: Retrieves the MIME type that was requested by the user, defaulting to `text/plain`.
    *   `const authHeader = response.headers.get('Authorization') || \`Bearer ${params?.accessToken || ''}\``: Attempts to get the `Authorization` header from the *current* response. If not present (which it usually won't be from Google's response), it reconstructs it using the original `accessToken` from `params`. This header is needed for subsequent API calls.

3.  **Content Preparation (Google Sheets Specifics)**
    ```typescript
      let preparedContent: string | undefined =
        typeof params?.content === 'string' ? (params?.content as string) : undefined

      if (requestedMimeType === 'application/vnd.google-apps.spreadsheet' && params?.content) {
        const { csv, rowCount, columnCount } = handleSheetsFormat(params.content as unknown)
        if (csv !== undefined) {
          preparedContent = csv
          logger.info('Prepared CSV content for Google Sheets upload', {
            fileId,
            fileName: params?.fileName,
            rowCount,
            columnCount,
          })
        }
      }
    ```
    *   `let preparedContent: string | undefined = ...`: Initializes `preparedContent` with the original `params.content` if it's a string, otherwise `undefined`. This variable will hold the content in the format suitable for upload.
    *   `if (requestedMimeType === 'application/vnd.google-apps.spreadsheet' && params?.content)`: Checks if the target file type is a Google Sheet *and* if content was provided.
    *   `const { csv, rowCount, columnCount } = handleSheetsFormat(params.content as unknown)`: Calls the `handleSheetsFormat` utility. This function is responsible for taking potentially varied input for a sheet and converting it to CSV. It returns the CSV content along with row/column counts.
    *   `if (csv !== undefined)`: If `handleSheetsFormat` successfully produced CSV content.
    *   `preparedContent = csv`: Updates `preparedContent` to be the CSV string.
    *   `logger.info(...)`: Logs that content was prepared for Google Sheets upload, including details like `rowCount` and `columnCount`.

4.  **Determine Upload MIME Type**
    ```typescript
      const uploadMimeType = GOOGLE_WORKSPACE_MIME_TYPES.includes(requestedMimeType)
        ? SOURCE_MIME_TYPES[requestedMimeType] || 'text/plain'
        : requestedMimeType
    ```
    *   `const uploadMimeType = ...`: This line determines the correct `Content-Type` header for the *content upload* (PATCH) request.
    *   `GOOGLE_WORKSPACE_MIME_TYPES.includes(requestedMimeType)`: Checks if the `requestedMimeType` (the final desired format, e.g., Google Sheet) is one of Google's native Workspace formats.
    *   `? SOURCE_MIME_TYPES[requestedMimeType] || 'text/plain'`: If it *is* a Google Workspace type, it looks up the corresponding *source* MIME type in `SOURCE_MIME_TYPES` (e.g., if target is `application/vnd.google-apps.spreadsheet`, the source might be `text/csv`). If no specific source type is found, it defaults to `text/plain`. This is crucial for Google Drive to correctly understand how to convert the uploaded content.
    *   `: requestedMimeType`: If it's *not* a Google Workspace type (e.g., a plain `.pdf` file), the `uploadMimeType` is simply the `requestedMimeType` itself.

    ```typescript
      logger.info('Uploading content to file', {
        fileId,
        fileName: params?.fileName,
        requestedMimeType,
        uploadMimeType,
      })
    ```
    *   `logger.info(...)`: Logs that content upload is starting, showing the `fileId`, `fileName`, the originally `requestedMimeType`, and the `uploadMimeType` determined for the actual content transfer.

5.  **Content Upload (PATCH Request)**
    ```typescript
      const uploadResponse = await fetch(
        `https://www.googleapis.com/upload/drive/v3/files/${fileId}?uploadType=media&supportsAllDrives=true`,
        {
          method: 'PATCH',
          headers: {
            Authorization: authHeader,
            'Content-Type': uploadMimeType,
          },
          body: preparedContent !== undefined ? preparedContent : params?.content || '',
        }
      )

      if (!uploadResponse.ok) {
        const uploadError = await uploadResponse.json()
        logger.error('Failed to upload content to file', {
          status: uploadResponse.status,
          statusText: uploadResponse.statusText,
          error: uploadError,
        })
        throw new Error(uploadError.error?.message || 'Failed to upload content to file')
      }
    ```
    *   `const uploadResponse = await fetch(...)`: Makes the second API call to actually upload the file content.
    *   `` `https://www.googleapis.com/upload/drive/v3/files/${fileId}?uploadType=media&supportsAllDrives=true` ``: The specific Google Drive endpoint for uploading file *content*. `uploadType=media` signifies a simple upload of file data.
    *   `method: 'PATCH'`: Uses the PATCH method to update the existing (empty) file with content.
    *   `headers: { Authorization: authHeader, 'Content-Type': uploadMimeType }`: Sets the `Authorization` and the determined `Content-Type` for the content being uploaded.
    *   `body: preparedContent !== undefined ? preparedContent : params?.content || ''`: The actual content to upload. It prioritizes `preparedContent` (especially for Sheets), falling back to the original `params.content`, or an empty string if neither is available.
    *   `if (!uploadResponse.ok) { ... }`: Checks for success of the content upload. If it fails, an error is logged and thrown, similar to the initial creation step.

6.  **Post-Conversion Name Update (for Google Workspace files)**
    ```typescript
      if (GOOGLE_WORKSPACE_MIME_TYPES.includes(requestedMimeType)) {
        logger.info('Updating file name to ensure it persists after conversion', {
          fileId,
          fileName: params?.fileName,
        })

        const updateNameResponse = await fetch(
          `https://www.googleapis.com/drive/v3/files/${fileId}?supportsAllDrives=true`,
          {
            method: 'PATCH',
            headers: {
              Authorization: authHeader,
              'Content-Type': 'application/json',
            },
            body: JSON.stringify({
              name: params?.fileName,
            }),
          }
        )

        if (!updateNameResponse.ok) {
          logger.warn('Failed to update filename after conversion, but content was uploaded', {
            status: updateNameResponse.status,
            statusText: updateNameResponse.statusText,
          })
        }
      }
    ```
    *   `if (GOOGLE_WORKSPACE_MIME_TYPES.includes(requestedMimeType))`: This block only executes if the file was intended to be a Google Workspace document (e.g., a Google Doc, Sheet, or Slide), as these are the types that sometimes "lose" their names during conversion.
    *   `logger.info(...)`: Logs the intent to update the filename.
    *   `const updateNameResponse = await fetch(...)`: Makes a *third* API call (a `PATCH`) to update the file's metadata, specifically its `name`.
    *   `` `https://www.googleapis.com/drive/v3/files/${fileId}?supportsAllDrives=true` ``: The standard file metadata update endpoint.
    *   `method: 'PATCH'`: Uses the PATCH method to update existing properties.
    *   `headers: { Authorization: authHeader, 'Content-Type': 'application/json' }`: Sets the authentication and indicates the body is JSON.
    *   `body: JSON.stringify({ name: params?.fileName })`: The JSON body contains only the `name` property to be updated to the original desired filename.
    *   `if (!updateNameResponse.ok) { ... }`: Checks for success of this name update.
    *   `logger.warn(...)`: If the name update fails, it logs a *warning* (not an error that throws), because the primary goal (content upload) was successful. The file exists and has content, even if the name might be generic.

7.  **Fetch Final File Metadata**
    ```typescript
      const finalFileResponse = await fetch(
        `https://www.googleapis.com/drive/v3/files/${fileId}?supportsAllDrives=true&fields=id,name,mimeType,webViewLink,webContentLink,size,createdTime,modifiedTime,parents`,
        {
          headers: {
            Authorization: authHeader,
          },
        }
      )

      const finalFile = await finalFileResponse.json()
    ```
    *   `const finalFileResponse = await fetch(...)`: Makes a *fourth* and final API call to retrieve the complete metadata for the file.
    *   `` `https://www.googleapis.com/drive/v3/files/${fileId}?supportsAllDrives=true&fields=id,name,mimeType,webViewLink,webContentLink,size,createdTime,modifiedTime,parents` ``: The URL for fetching file metadata. The `fields` parameter is crucial for efficiency, requesting only the specific fields needed for the output.
    *   `headers: { Authorization: authHeader }`: Authentication header.
    *   `const finalFile = await finalFileResponse.json()`: Parses the JSON response, which contains the comprehensive metadata.

8.  **Return Tool Output**
    ```typescript
      return {
        success: true,
        output: {
          file: {
            id: finalFile.id,
            name: finalFile.name,
            mimeType: finalFile.mimeType,
            webViewLink: finalFile.webViewLink,
            webContentLink: finalFile.webContentLink,
            size: finalFile.size,
            createdTime: finalFile.createdTime,
            modifiedTime: finalFile.modifiedTime,
            parents: finalFile.parents,
          },
        },
      }
    ```
    *   `return { ... }`: Returns the final structured output of the tool, conforming to the `GoogleDriveUploadResponse` type.
    *   `success: true`: Indicates the operation completed successfully.
    *   `output: { file: { ... } }`: Contains a `file` object with all the requested metadata fields from `finalFile`, providing useful information about the uploaded file.

**Outside `transformResponse`'s `try` block:**

```typescript
    } catch (error: any) {
      logger.error('Error in upload transformation', {
        error: error.message,
        stack: error.stack,
      })
      throw error
    }
```

*   `catch (error: any)`: Catches any errors thrown within the `try` block.
*   `logger.error(...)`: Logs the error message and stack trace for debugging purposes.
*   `throw error`: Re-throws the error, allowing the calling system to handle the failure.

#### Outputs Definition

```typescript
  outputs: {
    file: { type: 'json', description: 'Uploaded file metadata including ID, name, and links' },
  },
}
```

*   `outputs: { ... }`: Defines the structure and description of the data returned by the tool.
*   `file: { type: 'json', description: '...' }`: Specifies that the tool will output a `file` object, which is of type `json` (meaning a structured object), along with its description. This corresponds to the `output.file` object returned by `transformResponse`.

---

This detailed breakdown covers the purpose, simplified logic, and a line-by-line explanation of the entire Google Drive upload tool configuration.