This TypeScript file defines a "Gmail Read" tool, a crucial component in a larger system designed to interact with external services, likely for an AI agent or automated workflow platform. Its primary purpose is to encapsulate all the necessary configuration and logic to **read emails from a user's Gmail account** via the Gmail API.

Essentially, this file acts as a blueprint for a capability:
1.  **Declares what it does:** "Read emails from Gmail."
2.  **Specifies prerequisites:** Requires Google OAuth authentication with specific permissions.
3.  **Defines inputs:** What information it needs from the user (e.g., email ID, folder, whether to include attachments).
4.  **Details how to make the API call:** Constructs the correct URL, HTTP method, and headers.
5.  **Explains how to interpret the API response:** Processes the raw data from Gmail into a structured, readable format.
6.  **Defines outputs:** What information it will return (e.g., email content, metadata, attachments).

This modular approach allows the overarching system to easily integrate new tools and interact with various APIs in a standardized way.

---

### Simplified Complex Logic

The most intricate parts of this file lie within the `request` and `transformResponse` properties. Let's break them down:

#### 1. `request.url` - Building the Right API Query

This section dynamically constructs the URL for the Gmail API based on the input parameters. It handles two main scenarios:

*   **Reading a Specific Email:** If you provide a `messageId`, it directly targets that single email for a full detailed retrieval.
*   **Listing Emails:** If no `messageId` is given, it's a request to *list* emails. Here, it gets smart:
    *   It filters by `unreadOnly` if requested.
    *   It handles both standard Gmail folders (like `INBOX`, `SENT`) and custom user-defined labels. It defaults to `INBOX` if no folder is specified.
    *   It limits the number of messages retrieved (`maxResults`), with a sensible default and maximum to prevent overwhelming the system.

This logic ensures that whether you want one specific email or a list of emails from a certain category, the correct API endpoint is generated.

#### 2. `transformResponse` - Making Sense of Gmail's Data

Gmail's API response can be complex. This section's job is to take that raw JSON and turn it into something useful for the user or the AI agent.

*   **Direct Message Fetch (by ID):** If the initial `request.url` was for a single message, it directly processes that message using an internal utility function (`processMessage`).
*   **Message List Fetch:** This is where it gets more involved:
    *   Gmail's "list messages" endpoint only gives you message *IDs*, not the full email content. So, if `maxResults` is greater than 0, the tool has to make *additional* API calls for each message ID to get its full details.
    *   **Single Message Detail (after listing):** If `maxResults` is 1 (the default), it fetches the details of *only the first message* from the list. This is useful for AI agents that often just need to act on the most relevant single item.
    *   **Multiple Message Summary:** If `maxResults` is greater than 1, it fetches details for *all* requested messages (up to 10). It then uses helper functions (`processMessageForSummary`, `createMessagesSummary`) to generate a concise text summary of these emails, rather than returning all their full contents. This is crucial for presenting digestible information when many emails match.
    *   **Attachment Handling:** If `includeAttachments` is true, it processes each fetched message to extract and include any attachments in the final output.
    *   **Robust Error Handling:** It includes `try...catch` blocks to gracefully handle potential issues during API calls (e.g., network errors, permissions issues) and provides informative error messages.

This transformation logic intelligently adapts to the number of requested emails, provides detailed content for single reads, summarized content for multiple reads, and handles attachments, all while making the data consistent and easy to consume.

---

### Line-by-Line Explanation

```typescript
// Imports necessary TypeScript types from local modules.
// '@/tools/gmail/types': Defines the structure of Gmail-related data, like attachments, parameters for reading, and the final tool response.
import type { GmailAttachment, GmailReadParams, GmailToolResponse } from '@/tools/gmail/types'
// '@/tools/gmail/utils': Imports utility functions specific to Gmail operations, such as creating message summaries and processing individual messages.
import {
  createMessagesSummary,
  GMAIL_API_BASE,
  processMessage,
  processMessageForSummary,
} from '@/tools/gmail/utils'
// '@/tools/types': Imports the base type for defining a tool configuration.
import type { ToolConfig } from '@/tools/types'

// Defines the 'gmailReadTool' as a constant, adhering to the 'ToolConfig' interface.
// 'GmailReadParams' specifies the input parameters this tool expects.
// 'GmailToolResponse' specifies the output structure this tool will produce.
export const gmailReadTool: ToolConfig<GmailReadParams, GmailToolResponse> = {
  // Unique identifier for this tool.
  id: 'gmail_read',
  // Human-readable name for the tool.
  name: 'Gmail Read',
  // A brief description of what the tool does.
  description: 'Read emails from Gmail',
  // Version of the tool.
  version: '1.0.0',

  // Configuration for OAuth authentication.
  oauth: {
    // Specifies that authentication is mandatory for this tool.
    required: true,
    // The provider responsible for handling the OAuth flow (e.g., Google).
    provider: 'google-email',
    // Additional OAuth scopes (permissions) required from the user.
    // 'https://www.googleapis.com/auth/gmail.labels': Allows access to manage (but not read) labels. Used to identify labels.
    // 'https://www.googleapis.com/auth/gmail.readonly': Grants permission to read email metadata and content.
    additionalScopes: [
      'https://www.googleapis.com/auth/gmail.labels',
      'https://www.googleapis.com/auth/gmail.readonly',
    ],
  },

  // Defines the input parameters this tool accepts.
  params: {
    // Parameter for the OAuth access token.
    accessToken: {
      type: 'string', // Data type is string.
      required: true, // This parameter is mandatory.
      visibility: 'hidden', // Should not be shown to the user in a UI, as it's an internal token.
      description: 'Access token for Gmail API', // Explanation of the parameter.
    },
    // Optional parameter to read a specific message by its ID.
    messageId: {
      type: 'string',
      required: false, // Not mandatory, as the tool can list messages too.
      visibility: 'user-only', // Can be exposed to the user in a UI.
      description: 'ID of the message to read',
    },
    // Optional parameter to specify a folder/label to read emails from.
    folder: {
      type: 'string',
      required: false,
      visibility: 'user-only',
      description: 'Folder/label to read emails from',
    },
    // Optional boolean to retrieve only unread messages.
    unreadOnly: {
      type: 'boolean',
      required: false,
      visibility: 'user-only',
      description: 'Only retrieve unread messages',
    },
    // Optional parameter to limit the number of messages retrieved when listing.
    maxResults: {
      type: 'number',
      required: false,
      visibility: 'user-only',
      description: 'Maximum number of messages to retrieve (default: 1, max: 10)',
    },
    // Optional boolean to include email attachments in the output.
    includeAttachments: {
      type: 'boolean',
      required: false,
      visibility: 'user-only',
      description: 'Download and include email attachments',
    },
  },

  // Defines how to make the HTTP request to the Gmail API.
  request: {
    // A function that constructs the API URL based on the provided parameters.
    url: (params) => {
      // If a specific messageId is provided, construct the URL to fetch that message directly.
      if (params.messageId) {
        // Appends '?format=full' to get the complete message content, including body and headers.
        return `${GMAIL_API_BASE}/messages/${params.messageId}?format=full`
      }

      // If no messageId, construct the URL to list messages.
      const url = new URL(`${GMAIL_API_BASE}/messages`)

      // Initialize an array to build Gmail API query parameters (e.g., 'is:unread', 'in:inbox').
      const queryParams = []

      // Add 'is:unread' to the query if 'unreadOnly' is true.
      if (params.unreadOnly) {
        queryParams.push('is:unread')
      }

      // Handle the 'folder' parameter.
      if (params.folder) {
        // Check if it's a known system label (case-insensitive for comparison, then toLowerCase for API).
        if (['INBOX', 'SENT', 'DRAFT', 'TRASH', 'SPAM'].includes(params.folder.toUpperCase())) {
          queryParams.push(`in:${params.folder.toLowerCase()}`) // Use 'in:' for system labels.
        } else {
          // Otherwise, assume it's a user-defined label.
          queryParams.push(`label:${params.folder}`) // Use 'label:' for custom labels.
        }
      } else {
        // If no folder is specified, default to 'in:inbox'.
        queryParams.push('in:inbox')
      }

      // If any query parameters were built, append them to the URL's search string with 'q='.
      if (queryParams.length > 0) {
        url.searchParams.append('q', queryParams.join(' ')) // Join multiple query parts with a space.
      }

      // Set 'maxResults' for the list operation. Default to 1, with a maximum of 10.
      const maxResults = params.maxResults ? Math.min(params.maxResults, 10) : 1
      url.searchParams.append('maxResults', maxResults.toString())

      // Return the final URL string.
      return url.toString()
    },
    // The HTTP method to use for the request.
    method: 'GET',
    // A function that constructs the HTTP headers, including the Authorization token.
    headers: (params: GmailReadParams) => ({
      Authorization: `Bearer ${params.accessToken}`, // Uses the provided access token.
      'Content-Type': 'application/json', // Indicates the request expects JSON.
    }),
  },

  // Defines how to transform the raw HTTP response into the tool's defined output format.
  transformResponse: async (response: Response, params?: GmailReadParams) => {
    // Parse the JSON body of the API response.
    const data = await response.json()

    // --- Scenario 1: Fetching a single message directly by ID ---
    if (params?.messageId) {
      // Use the utility function 'processMessage' to format the single message data.
      return await processMessage(data, params)
    }

    // --- Scenario 2: Listing messages (initial API call returns message IDs) ---
    if (data.messages && Array.isArray(data.messages)) {
      // If the list is empty, return a message indicating no emails were found.
      if (data.messages.length === 0) {
        return {
          success: true,
          output: {
            content: 'No messages found in the selected folder.',
            metadata: {
              results: [], // Empty array for search results metadata.
            },
          },
        }
      }

      // Determine the actual max results to fetch, defaulting to 1, max 10.
      const maxResults = params?.maxResults ? Math.min(params.maxResults, 10) : 1

      // --- Sub-scenario 2a: Only interested in the first message ---
      if (maxResults === 1) {
        try {
          // Get the ID of the first message.
          const messageId = data.messages[0].id
          // Fetch the full details of this single message.
          const messageResponse = await fetch(
            `${GMAIL_API_BASE}/messages/${messageId}?format=full`,
            {
              headers: {
                Authorization: `Bearer ${params?.accessToken || ''}`, // Use access token for this secondary fetch.
                'Content-Type': 'application/json',
              },
            }
          )

          // Check if the secondary fetch was successful.
          if (!messageResponse.ok) {
            const errorData = await messageResponse.json()
            throw new Error(errorData.error?.message || 'Failed to fetch message details')
          }

          // Parse the detailed message and process it.
          const message = await messageResponse.json()
          return await processMessage(message, params)
        } catch (error: any) {
          // Handle errors during the secondary fetch of a single message.
          return {
            success: true, // Still a "success" for the tool, but reports an issue fetching details.
            output: {
              content: `Found messages but couldn't retrieve details: ${error.message || 'Unknown error'}`,
              metadata: {
                results: data.messages.map((msg: any) => ({
                  id: msg.id,
                  threadId: msg.threadId,
                })),
              },
            },
          }
        }
      } else {
        // --- Sub-scenario 2b: Fetching details for multiple messages ---
        try {
          // Create an array of Promises, each fetching the full details of a message.
          const messagePromises = data.messages.slice(0, maxResults).map(async (msg: any) => {
            const messageResponse = await fetch(
              `${GMAIL_API_BASE}/messages/${msg.id}?format=full`,
              {
                headers: {
                  Authorization: `Bearer ${params?.accessToken || ''}`,
                  'Content-Type': 'application/json',
                },
              }
            )

            if (!messageResponse.ok) {
              throw new Error(`Failed to fetch details for message ${msg.id}`)
            }

            return await messageResponse.json()
          })

          // Wait for all message detail fetches to complete.
          const messages = await Promise.all(messagePromises)

          // Process each fetched message into a summary-friendly format using 'processMessageForSummary'.
          const summaryMessages = messages.map(processMessageForSummary)

          // Initialize an array to hold all attachments found across all messages.
          const allAttachments: GmailAttachment[] = []
          if (params?.includeAttachments) {
            // If attachments are requested, iterate through messages to extract them.
            for (const msg of messages) {
              try {
                // Process each message (using the full 'processMessage' to get attachments).
                const processedResult = await processMessage(msg, params)
                // If attachments are present, add them to the 'allAttachments' array.
                if (
                  processedResult.output.attachments &&
                  processedResult.output.attachments.length > 0
                ) {
                  allAttachments.push(...processedResult.output.attachments)
                }
              } catch (error: any) {
                console.error(`Error processing message ${msg.id} for attachments:`, error)
              }
            }
          }

          // Return a summary of the multiple messages.
          return {
            success: true,
            output: {
              content: createMessagesSummary(summaryMessages), // Use utility to create a text summary.
              metadata: {
                results: summaryMessages.map((msg) => ({
                  id: msg.id,
                  threadId: msg.threadId,
                  subject: msg.subject,
                  from: msg.from,
                  date: msg.date,
                })),
              },
              attachments: allAttachments, // Include collected attachments.
            },
          }
        } catch (error: any) {
          // Handle errors during fetching details for multiple messages.
          return {
            success: true,
            output: {
              content: `Found ${data.messages.length} messages but couldn't retrieve all details: ${error.message || 'Unknown error'}`,
              metadata: {
                results: data.messages.map((msg: any) => ({
                  id: msg.id,
                  threadId: msg.threadId,
                })),
              },
              attachments: [],
            },
          }
        }
      }
    }

    // --- Fallback: For unexpected response formats from the initial Gmail API call ---
    return {
      success: true,
      output: {
        content: 'Unexpected response format from Gmail API',
        metadata: {
          results: [],
        },
      },
    }
  },

  // Defines the expected output structure of the tool, useful for documentation or UI generation.
  outputs: {
    // The main text content, typically the email body or a summary.
    content: { type: 'string', description: 'Text content of the email' },
    // Structured data about the email(s).
    metadata: { type: 'json', description: 'Metadata of the email' },
    // An array of files, representing attachments.
    attachments: { type: 'file[]', description: 'Attachments of the email' },
  },
}
```