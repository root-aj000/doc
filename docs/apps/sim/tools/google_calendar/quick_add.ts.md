This TypeScript file defines the configuration for a powerful "Google Calendar Quick Add" tool. Essentially, it's a blueprint for how an application (perhaps an AI assistant or a workflow engine) can interact with the Google Calendar API to create events using natural language.

Let's break down its purpose, logic, and each line of code.

---

## Google Calendar Quick Add Tool Configuration

This file, `quickAddTool.ts`, configures a specific tool designed to interact with the Google Calendar API. Its primary function is to enable the creation of new calendar events from natural language text, like "meeting with John tomorrow at 3pm," and optionally add attendees to that event.

### Purpose of this file

The main goal of this file is to export a `ToolConfig` object. This object acts as a declarative schema or definition for how an external system (like a large language model, a workflow automation tool, or a custom application) should use the "Google Calendar Quick Add" functionality. It specifies:

1.  **Tool Metadata:** Its ID, name, description, and version.
2.  **Authentication Requirements:** That it needs Google Calendar OAuth and specific permissions.
3.  **Input Parameters:** What information the tool expects to receive (e.g., event text, calendar ID, access token, attendees).
4.  **Request Details:** How to construct the HTTP request to the Google Calendar API for adding the event.
5.  **Response Transformation:** How to process the raw API response into a more user-friendly output, including a crucial step to handle adding attendees via a separate API call.
6.  **Output Structure:** What kind of data the tool will return.

In simpler terms, this file tells a system: "Here's how to quickly create a Google Calendar event from text, including how to handle authentication, what inputs you need, how to talk to Google, and what to do with the response."

---

### Simplified Complex Logic: How it Handles Attendees

The most complex part of this file is how it deals with attendees. The Google Calendar API's `quickAdd` endpoint is great for simple text-based event creation, but it doesn't directly support adding attendees in the initial call.

Here's the simplified logic for attendee handling:

1.  **Initial Event Creation:** The tool first uses the `quickAdd` endpoint with the provided natural language text to create a basic event.
2.  **Check for Attendees:** After the initial event is created, it checks if the user *also* requested to add attendees.
3.  **Prepare Attendees:** If attendees are requested, it processes them, converting them into a standardized list of email addresses, whether they were provided as an array or a comma-separated string.
4.  **Update the Event:** If there are valid attendees, the tool then makes a *second* API call. It uses the `PATCH` method on the newly created event to add the attendees. This is a common pattern when an API has a "quick create" endpoint that's feature-limited, and more advanced features require a separate update call.
5.  **Robustness:** It includes error handling for this second update call, ensuring that even if adding attendees fails, the initial event creation is still reported as successful (though a warning is logged).
6.  **Final Output:** It combines the information from either the initial event creation or the updated event (if attendees were successfully added) into a single, comprehensive response.

---

### Line-by-Line Explanation

```typescript
import {
  CALENDAR_API_BASE,
  type GoogleCalendarQuickAddParams,
  type GoogleCalendarQuickAddResponse,
} from '@/tools/google_calendar/types'
import type { ToolConfig } from '@/tools/types'
```
These lines import necessary types and constants from other parts of the application:
*   `CALENDAR_API_BASE`: A string constant representing the base URL for the Google Calendar API (e.g., `https://www.googleapis.com/calendar/v3`). This prevents hardcoding the URL multiple times.
*   `GoogleCalendarQuickAddParams`: A TypeScript type defining the structure of the input parameters for this specific quick-add operation (what data the tool expects to receive).
*   `GoogleCalendarQuickAddResponse`: A TypeScript type defining the structure of the expected output from this operation *after* processing.
*   `ToolConfig`: A generic TypeScript type that defines the overall structure for any tool configuration within this application. It typically takes two type arguments: one for the input parameters and one for the output response.

```typescript
export const quickAddTool: ToolConfig<
  GoogleCalendarQuickAddParams,
  GoogleCalendarQuickAddResponse
> = {
```
This line declares and exports a constant named `quickAddTool`.
*   `export`: Makes this constant available for other files to import and use.
*   `const quickAddTool`: Declares an immutable constant.
*   `: ToolConfig<GoogleCalendarQuickAddParams, GoogleCalendarQuickAddResponse>`: This is a TypeScript type annotation. It specifies that `quickAddTool` must conform to the `ToolConfig` interface, and it uses `GoogleCalendarQuickAddParams` as its input type and `GoogleCalendarQuickAddResponse` as its output type. This provides strong type checking for the entire configuration object.
*   `= { ... }`: Defines the object literal that holds the configuration details.

```typescript
  id: 'google_calendar_quick_add',
  name: 'Google Calendar Quick Add',
  description: 'Create events from natural language text',
  version: '1.0.0',
```
These are basic metadata fields for the tool:
*   `id`: A unique identifier for this tool within the system.
*   `name`: A human-readable name for the tool, often used in UIs.
*   `description`: A brief explanation of what the tool does, useful for documentation or for LLMs to understand its capabilities.
*   `version`: The version number of this tool configuration.

```typescript
  oauth: {
    required: true,
    provider: 'google-calendar',
    additionalScopes: ['https://www.googleapis.com/auth/calendar'],
  },
```
This section specifies the OAuth (Open Authorization) requirements for using this tool:
*   `required: true`: Indicates that authentication is mandatory to use this tool.
*   `provider: 'google-calendar'`: Specifies which OAuth provider to use (e.g., Google's OAuth service for Calendar).
*   `additionalScopes: ['https://www.googleapis.com/auth/calendar']`: This is crucial. It requests specific permissions from the user. `https://www.googleapis.com/auth/calendar` grants full read/write access to all calendars that the authenticated user can access, which is necessary for creating and modifying events.

```typescript
  params: {
    accessToken: {
      type: 'string',
      required: true,
      visibility: 'hidden',
      description: 'Access token for Google Calendar API',
    },
    calendarId: {
      type: 'string',
      required: false,
      visibility: 'user-only',
      description: 'Calendar ID (defaults to primary)',
    },
    text: {
      type: 'string',
      required: true,
      visibility: 'user-or-llm',
      description:
        'Natural language text describing the event (e.g., "Meeting with John tomorrow at 3pm")',
    },
    attendees: {
      type: 'array',
      required: false,
      visibility: 'user-or-llm',
      description: 'Array of attendee email addresses (comma-separated string also accepted)',
    },
    sendUpdates: {
      type: 'string',
      required: false,
      visibility: 'user-only',
      description: 'How to send updates to attendees: all, externalOnly, or none',
    },
  },
```
This `params` section defines the input parameters the tool expects. Each parameter has a schema:
*   `accessToken`:
    *   `type: 'string'`: It expects a string value.
    *   `required: true`: It's a mandatory parameter.
    *   `visibility: 'hidden'`: This suggests it's an internal parameter, likely passed by the system after OAuth, not directly provided by a user or LLM.
    *   `description`: Explains its purpose.
*   `calendarId`:
    *   `type: 'string'`: String value.
    *   `required: false`: Optional.
    *   `visibility: 'user-only'`: Implies it might be a direct user input, not typically inferred by an LLM.
    *   `description`: Explains it can default to the user's primary calendar.
*   `text`:
    *   `type: 'string'`: String value.
    *   `required: true`: Mandatory.
    *   `visibility: 'user-or-llm'`: This is the core input; it can come from a direct user prompt or be generated by an LLM.
    *   `description`: Provides an example and clarifies its purpose.
*   `attendees`:
    *   `type: 'array'`: Expects an array (though the description notes a comma-separated string is also accepted, which will be handled in `transformResponse`).
    *   `required: false`: Optional.
    *   `visibility: 'user-or-llm'`: Can be provided by user or LLM.
    *   `description`: Explains what kind of input is expected for attendees.
*   `sendUpdates`:
    *   `type: 'string'`: String value.
    *   `required: false`: Optional.
    *   `visibility: 'user-only'`: Direct user input, not typically inferred by an LLM.
    *   `description`: Specifies the allowed values (`all`, `externalOnly`, `none`) for sending notifications.

```typescript
  request: {
    url: (params: GoogleCalendarQuickAddParams) => {
      const calendarId = params.calendarId || 'primary'
      const queryParams = new URLSearchParams()

      queryParams.append('text', params.text)

      if (params.sendUpdates !== undefined) {
        queryParams.append('sendUpdates', params.sendUpdates)
      }

      return `${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events/quickAdd?${queryParams.toString()}`
    },
    method: 'POST',
    headers: (params: GoogleCalendarQuickAddParams) => ({
      Authorization: `Bearer ${params.accessToken}`,
      'Content-Type': 'application/json',
    }),
  },
```
This `request` section defines how to construct the initial HTTP request to the Google Calendar API:
*   `url`: This is a function that dynamically generates the API endpoint URL based on the provided `params`.
    *   `params: GoogleCalendarQuickAddParams`: Specifies the type of the input parameters for this function.
    *   `const calendarId = params.calendarId || 'primary'`: Sets the `calendarId`. If `params.calendarId` is not provided (is `undefined` or `null`), it defaults to `'primary'`, which refers to the user's default calendar.
    *   `const queryParams = new URLSearchParams()`: Creates a new `URLSearchParams` object, a utility for easily building URL query strings.
    *   `queryParams.append('text', params.text)`: Adds the natural language event description (`params.text`) as a query parameter named `text`. This is required by the `quickAdd` endpoint.
    *   `if (params.sendUpdates !== undefined)`: Conditionally adds the `sendUpdates` parameter if it was provided.
    *   `return `${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events/quickAdd?${queryParams.toString()}``: This line constructs the final URL using a template literal:
        *   `${CALENDAR_API_BASE}`: The base URL for the Google Calendar API.
        *   `/calendars/${encodeURIComponent(calendarId)}`: Specifies the calendar to interact with. `encodeURIComponent` is crucial to properly escape any special characters in the calendar ID (though `primary` doesn't need it, other IDs might).
        *   `/events/quickAdd`: The specific endpoint for quickly adding events.
        *   `?${queryParams.toString()}`: Appends the generated query string (e.g., `?text=Meeting%20with%20John`) to the URL.
*   `method: 'POST'`: Specifies that the HTTP method for this request should be `POST`, which is used for creating resources.
*   `headers`: This is a function that returns an object containing HTTP headers for the request.
    *   `Authorization: `Bearer ${params.accessToken}``: This is the standard way to send an OAuth 2.0 access token. `Bearer` indicates the type of token, followed by the actual token. This authenticates the request.
    *   `'Content-Type': 'application/json'`: Specifies that the request body (if any, though `quickAdd` uses query params) is expected to be JSON. This is a common practice for many REST APIs.

```typescript
  transformResponse: async (response: Response, params) => {
    const data = await response.json()

    // Handle attendees if provided
    let finalEventData = data
    if (params?.attendees) {
      let attendeeList: string[] = []
      const attendees = params.attendees as string | string[]

      if (Array.isArray(attendees)) {
        attendeeList = attendees.filter((email: string) => email && email.trim().length > 0)
      } else if (typeof attendees === 'string' && attendees.trim().length > 0) {
        // Convert comma-separated string to array
        attendeeList = attendees
          .split(',')
          .map((email: string) => email.trim())
          .filter((email: string) => email.length > 0)
      }

      if (attendeeList.length > 0) {
        try {
          // Update the event with attendees
          const calendarId = params.calendarId || 'primary'
          const eventId = data.id

          // Prepare update data
          const updateData = {
            attendees: attendeeList.map((email: string) => ({ email })),
          }

          // Build update URL with sendUpdates if specified
          const updateQueryParams = new URLSearchParams()
          if (params.sendUpdates !== undefined) {
            updateQueryParams.append('sendUpdates', params.sendUpdates)
          }

          const updateUrl = `${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events/${eventId}${updateQueryParams.toString() ? `?${updateQueryParams.toString()}` : ''}`

          // Make the update request
          const updateResponse = await fetch(updateUrl, {
            method: 'PATCH',
            headers: {
              Authorization: `Bearer ${params.accessToken}`,
              'Content-Type': 'application/json',
            },
            body: JSON.stringify(updateData),
          })

          if (updateResponse.ok) {
            finalEventData = await updateResponse.json()
          } else {
            // If update fails, we still return the original event but log the error
            console.warn(
              'Failed to add attendees to quick-added event:',
              await updateResponse.text()
            )
          }
        } catch (error) {
          // If attendee update fails, we still return the original event
          console.warn('Error adding attendees to quick-added event:', error)
        }
      }
    }

    return {
      success: true,
      output: {
        content: `Event "${finalEventData?.summary || 'Untitled'}" created successfully ${finalEventData?.attendees?.length ? ` with ${finalEventData.attendees.length} attendee(s)` : ''}`,
        metadata: {
          id: finalEventData.id,
          htmlLink: finalEventData.htmlLink,
          status: finalEventData.status,
          summary: finalEventData.summary,
          description: finalEventData.description,
          location: finalEventData.location,
          start: finalEventData.start,
          end: finalEventData.end,
          attendees: finalEventData.attendees,
          creator: finalEventData.creator,
          organizer: finalEventData.organizer,
        },
      },
    }
  },
```
This `transformResponse` function is executed *after* the initial `POST` request to `quickAdd` completes. It's responsible for processing the API's raw response and returning a structured output. This is where the complex logic for attendee handling resides.

*   `async (response: Response, params)`: An asynchronous function that receives the raw `Response` object from the initial `quickAdd` call and the original `params` provided to the tool. `async` keyword is used because it will perform potentially two `fetch` calls (`quickAdd` and then `PATCH` for attendees).
*   `const data = await response.json()`: Parses the JSON body of the initial `quickAdd` API response. `data` now holds the details of the newly created (but potentially attendee-less) event.
*   `let finalEventData = data`: Initializes `finalEventData` with the data from the initial event creation. This variable will be updated if attendees are successfully added.
*   `if (params?.attendees)`: Checks if the original `params` included an `attendees` property. The `?` is optional chaining, ensuring it doesn't error if `params` itself is null/undefined.
*   `let attendeeList: string[] = []`: Declares an empty array to store processed attendee email addresses.
*   `const attendees = params.attendees as string | string[]`: Type assertion to inform TypeScript that `params.attendees` can be either a string or an array of strings, even if the `params` schema declared it as an array (this allows for runtime flexibility).
*   `if (Array.isArray(attendees)) { ... }`: If `attendees` is already an array, it filters out any empty or whitespace-only email strings.
*   `else if (typeof attendees === 'string' && attendees.trim().length > 0) { ... }`: If `attendees` is a non-empty string, it assumes it's a comma-separated list of emails.
    *   `.split(',')`: Splits the string into an array by commas.
    *   `.map((email: string) => email.trim())`: Trims whitespace from each email.
    *   `.filter((email: string) => email.length > 0)`: Filters out any empty strings that might result from extra commas or just whitespace.
*   `if (attendeeList.length > 0)`: Proceeds to update the event only if valid attendees were found after processing.
*   `try { ... } catch (error) { ... }`: A `try-catch` block is used to gracefully handle potential errors during the attendee update process.
    *   `const calendarId = params.calendarId || 'primary'`: Re-determines the calendar ID.
    *   `const eventId = data.id`: Extracts the `id` of the newly created event from the `data` obtained in the first API call.
    *   `const updateData = { attendees: attendeeList.map((email: string) => ({ email }))}`: Prepares the JSON payload for the `PATCH` request. Google Calendar API expects an array of objects, where each object has an `email` property.
    *   `const updateQueryParams = new URLSearchParams()`: Creates a new `URLSearchParams` object for the update request's query parameters.
    *   `if (params.sendUpdates !== undefined) { updateQueryParams.append('sendUpdates', params.sendUpdates) }`: Conditionally adds the `sendUpdates` parameter to the update request's query string.
    *   `const updateUrl = `${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events/${eventId}${updateQueryParams.toString() ? `?${updateQueryParams.toString()}` : ''}``: Constructs the URL for the `PATCH` request. It targets the specific event by its `eventId`. It conditionally appends query parameters if `updateQueryParams` are present.
    *   `const updateResponse = await fetch(updateUrl, { ... })`: Makes the HTTP `PATCH` request to update the event with attendees.
        *   `method: 'PATCH'`: Specifies the HTTP method for updating a resource.
        *   `headers`: Includes the `Authorization` token and `Content-Type`.
        *   `body: JSON.stringify(updateData)`: Sends the `updateData` (the attendees list) as a JSON string in the request body.
    *   `if (updateResponse.ok) { finalEventData = await updateResponse.json() }`: If the `PATCH` request is successful (`updateResponse.ok` is true, meaning HTTP status 2xx), it parses the response and updates `finalEventData` to include the attendees.
    *   `else { console.warn(...) }`: If the `PATCH` request fails, it logs a warning but crucially, it *doesn't throw an error*. This means the initial event creation is still considered successful, but the attendees might not have been added.
    *   `catch (error) { console.warn(...) }`: Catches any network errors or other exceptions during the `PATCH` request. Similar to the `else` block, it logs a warning but allows the function to complete, returning the original event data.
*   `return { success: true, output: { ... } }`: This is the final structured output of the `transformResponse` function, which is what the calling system will receive.
    *   `success: true`: Indicates that the tool execution was generally successful.
    *   `output`: An object containing the final result, split into a human-readable `content` message and detailed `metadata`.
    *   `content`: A formatted string message confirming the event creation, including the event summary and optionally the number of attendees. Uses optional chaining (`?.`) to safely access properties.
    *   `metadata`: An object containing key details of the created/updated event extracted from `finalEventData`. This provides structured data for programmatic use (e.g., event ID, link, start/end times).

```typescript
  outputs: {
    content: {
      type: 'string',
      description: 'Event creation confirmation message from natural language',
    },
    metadata: { type: 'json', description: 'Created event metadata including parsed details' },
  },
}
```
This `outputs` section defines the expected structure and description of the data returned by the tool:
*   `content`:
    *   `type: 'string'`: Specifies that the `content` field in the output will be a string.
    *   `description`: Explains what this string represents.
*   `metadata`:
    *   `type: 'json'`: Specifies that the `metadata` field will be a JSON object.
    *   `description`: Explains that it contains the detailed parsed information about the created event.

---

In summary, this `quickAddTool` configuration file is a comprehensive and robust definition for a Google Calendar integration. It handles authentication, flexible input parsing, making API calls (including a multi-step process for attendee handling), and structuring the output for easy consumption by an integrating system.