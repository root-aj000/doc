This TypeScript file defines a "tool" configuration for creating events in Google Calendar. It's designed to be used within a larger system (likely an AI agent framework or a similar platform) that needs to interact with external services. This tool specifically encapsulates all the necessary logic and metadata for making a Google Calendar API call to add a new event.

---

### **Purpose of this File**

At its core, this file acts as a blueprint for a specific action: **creating an event in a user's Google Calendar**.

Imagine you have an AI assistant that you can ask to "Schedule a meeting for tomorrow at 10 AM with John about the project kick-off." This file provides the structured definition that tells the AI system:

1.  **What this action is called** and what it does.
2.  **What information it needs** to perform the action (e.g., event title, start time, attendees).
3.  **How to authenticate** with Google Calendar.
4.  **How to construct the actual request** to the Google Calendar API (the URL, HTTP method, headers, and the body of data).
5.  **How to interpret the response** from the Google Calendar API and present it in a useful way.

In essence, it's a self-contained module that allows an application to effortlessly integrate Google Calendar event creation functionality without needing to write the low-level API interaction code repeatedly.

---

### **Detailed Explanation**

Let's break down the code line by line.

```typescript
import {
  CALENDAR_API_BASE,
  type GoogleCalendarApiEventResponse,
  type GoogleCalendarCreateParams,
  type GoogleCalendarCreateResponse,
  type GoogleCalendarEventRequestBody,
} from '@/tools/google_calendar/types'
import type { ToolConfig } from '@/tools/types'
```

*   **`import { ... } from '@/tools/google_calendar/types'`**: This line imports several types and a constant from another file (`types.ts`) related to Google Calendar API interactions.
    *   `CALENDAR_API_BASE`: A string constant representing the base URL for the Google Calendar API (e.g., `https://www.googleapis.com/calendar/v3`).
    *   `type GoogleCalendarApiEventResponse`: A TypeScript type defining the structure of the data expected back from the Google Calendar API *after* an event is created.
    *   `type GoogleCalendarCreateParams`: A type defining the structure of the *input parameters* this tool expects from the calling application (e.g., summary, start time, access token).
    *   `type GoogleCalendarCreateResponse`: A type defining the structure of the *output* this tool will return *after* successfully creating an event.
    *   `type GoogleCalendarEventRequestBody`: A type defining the structure of the data that will be sent in the *request body* to the Google Calendar API to create an event.
*   **`import type { ToolConfig } from '@/tools/types'`**: This imports the `ToolConfig` type, which is a generic interface defining the overall structure for *any* tool within the system. It helps ensure all tools adhere to a consistent contract.

---

```typescript
export const createTool: ToolConfig<GoogleCalendarCreateParams, GoogleCalendarCreateResponse> = {
  id: 'google_calendar_create',
  name: 'Google Calendar Create Event',
  description: 'Create a new event in Google Calendar',
  version: '1.0.0',
```

*   **`export const createTool: ToolConfig<GoogleCalendarCreateParams, GoogleCalendarCreateResponse> = { ... }`**: This line defines and exports a constant variable named `createTool`. It's explicitly typed as `ToolConfig`, taking two generic arguments:
    *   `GoogleCalendarCreateParams`: This specifies the type of parameters the tool *accepts as input*.
    *   `GoogleCalendarCreateResponse`: This specifies the type of data the tool *returns as output*.
    *   The object `{ ... }` then provides the actual configuration for this tool.
*   **`id: 'google_calendar_create'`**: A unique identifier for this specific tool. Useful for programmatic referencing.
*   **`name: 'Google Calendar Create Event'`**: A human-readable name for the tool.
*   **`description: 'Create a new event in Google Calendar'`**: A brief explanation of what the tool does.
*   **`version: '1.0.0'`**: The version number of this tool definition.

---

```typescript
  oauth: {
    required: true,
    provider: 'google-calendar',
    additionalScopes: ['https://www.googleapis.com/auth/calendar'],
  },
```

*   **`oauth: { ... }`**: This object defines the authentication requirements for using this tool, specifically using OAuth 2.0.
    *   **`required: true`**: Indicates that OAuth authentication is absolutely necessary to use this tool.
    *   **`provider: 'google-calendar'`**: Specifies which OAuth provider should be used. This tells the system to use the configured Google Calendar OAuth setup.
    *   **`additionalScopes: ['https://www.googleapis.com/auth/calendar']`**: Defines the specific permissions (scopes) that the user must grant to the application. `'https://www.googleapis.com/auth/calendar'` is the scope required to create, modify, and delete events in the user's calendars.

---

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
    summary: {
      type: 'string',
      required: true,
      visibility: 'user-or-llm',
      description: 'Event title/summary',
    },
    description: {
      type: 'string',
      required: false,
      visibility: 'user-or-llm',
      description: 'Event description',
    },
    location: {
      type: 'string',
      required: false,
      visibility: 'user-or-llm',
      description: 'Event location',
    },
    startDateTime: {
      type: 'string',
      required: true,
      visibility: 'user-or-llm',
      description: 'Start date and time (RFC3339 format, e.g., 2025-06-03T10:00:00-08:00)',
    },
    endDateTime: {
      type: 'string',
      required: true,
      visibility: 'user-or-llm',
      description: 'End date and time (RFC3339 format, e.g., 2025-06-03T11:00:00-08:00)',
    },
    timeZone: {
      type: 'string',
      required: false,
      visibility: 'user-or-llm',
      description: 'Time zone (e.g., America/Los_Angeles)',
    },
    attendees: {
      type: 'array', // Note: The internal logic handles both string and array for flexibility
      required: false,
      visibility: 'user-or-llm',
      description: 'Array of attendee email addresses',
    },
    sendUpdates: {
      type: 'string',
      required: false,
      visibility: 'user-only',
      description: 'How to send updates to attendees: all, externalOnly, or none',
    },
  },
```

*   **`params: { ... }`**: This object defines all the input parameters (arguments) that the tool expects to receive to perform its action. Each parameter has a schema:
    *   **`type`**: The expected data type (e.g., `'string'`, `'array'`).
    *   **`required`**: A boolean indicating if this parameter *must* be provided.
    *   **`visibility`**: Specifies who can provide or see this parameter:
        *   `'hidden'`: The parameter is typically handled internally by the system (e.g., an access token) and not exposed to the end-user or an LLM.
        *   `'user-only'`: The parameter is typically provided by the end-user directly (e.g., selecting a specific calendar or setting an update preference).
        *   `'user-or-llm'`: The parameter can be provided by either the user or inferred by an LLM (Large Language Model) from the user's natural language request.
    *   **`description`**: A human-readable explanation of the parameter's purpose and expected format.

    Let's look at specific parameters:
    *   **`accessToken`**:
        *   `required: true`, `visibility: 'hidden'`. This is crucial for authentication and is managed by the system, not directly by the user or AI.
    *   **`calendarId`**:
        *   `required: false`, `visibility: 'user-only'`. If not provided, the `request` logic will default to the user's primary calendar.
    *   **`summary`**:
        *   `required: true`, `visibility: 'user-or-llm'`. The event title, a mandatory piece of information.
    *   **`description`, `location`, `timeZone`**:
        *   `required: false`, `visibility: 'user-or-llm'`. Optional details that can be provided. `timeZone` uses standard IANA time zone IDs (e.g., "America/Los_Angeles").
    *   **`startDateTime`, `endDateTime`**:
        *   `required: true`, `visibility: 'user-or-llm'`. These are mandatory and must be in **RFC3339 format** (a standard for date and time, including time zone offset, like `2025-06-03T10:00:00-08:00`). This is a common and precise format for APIs.
    *   **`attendees`**:
        *   `type: 'array'`, `required: false`, `visibility: 'user-or-llm'`. Expected to be an array of email addresses, but the `body` function demonstrates flexibility in handling comma-separated strings as well.
    *   **`sendUpdates`**:
        *   `required: false`, `visibility: 'user-only'`. Controls how Google Calendar notifies attendees (e.g., `'all'`, `'externalOnly'`, `'none'`).

---

```typescript
  request: {
    url: (params: GoogleCalendarCreateParams) => {
      const calendarId = params.calendarId || 'primary'
      const queryParams = new URLSearchParams()

      if (params.sendUpdates !== undefined) {
        queryParams.append('sendUpdates', params.sendUpdates)
      }

      const queryString = queryParams.toString()
      const finalUrl = `${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events${queryString ? `?${queryString}` : ''}`

      return finalUrl
    },
    method: 'POST',
    headers: (params: GoogleCalendarCreateParams) => ({
      Authorization: `Bearer ${params.accessToken}`,
      'Content-Type': 'application/json',
    }),
    body: (params: GoogleCalendarCreateParams): GoogleCalendarEventRequestBody => {
      const eventData: GoogleCalendarEventRequestBody = {
        summary: params.summary,
        start: {
          dateTime: params.startDateTime,
        },
        end: {
          dateTime: params.endDateTime,
        },
      }

      if (params.description) {
        eventData.description = params.description
      }

      if (params.location) {
        eventData.location = params.location
      }

      if (params.timeZone) {
        eventData.start.timeZone = params.timeZone
        eventData.end.timeZone = params.timeZone
      }

      // Handle both string and array cases for attendees
      let attendeeList: string[] = []
      if (params.attendees) {
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
      }

      if (attendeeList.length > 0) {
        eventData.attendees = attendeeList.map((email: string) => ({ email }))
      }

      return eventData
    },
  },
```

*   **`request: { ... }`**: This object defines how to construct the actual HTTP request to the Google Calendar API.
    *   **`url: (params: GoogleCalendarCreateParams) => { ... }`**: A function that dynamically generates the API endpoint URL based on the provided `params`.
        *   `const calendarId = params.calendarId || 'primary'`: If `calendarId` is not provided in the input, it defaults to `'primary'`, which refers to the user's main calendar.
        *   `const queryParams = new URLSearchParams()`: Initializes an object to help build URL query parameters.
        *   `if (params.sendUpdates !== undefined) { queryParams.append('sendUpdates', params.sendUpdates) }`: If `sendUpdates` is provided, it's added as a query parameter (e.g., `?sendUpdates=all`).
        *   `const queryString = queryParams.toString()`: Converts the `URLSearchParams` object into a URL-encoded string (e.g., `sendUpdates=all`).
        *   `const finalUrl = ...`: Constructs the full URL.
            *   `${CALENDAR_API_BASE}`: Starts with the base API URL.
            *   `/calendars/${encodeURIComponent(calendarId)}/events`: Appends the path for event creation within a specific calendar. `encodeURIComponent` ensures that the `calendarId` is correctly formatted for a URL, especially if it contains special characters.
            *   `${queryString ? `?${queryString}` : ''}`: Conditionally adds the query string if it exists, prefixed with a `?`.
        *   `return finalUrl`: Returns the complete URL.
    *   **`method: 'POST'`**: Specifies that this request will use the HTTP POST method, which is standard for creating new resources.
    *   **`headers: (params: GoogleCalendarCreateParams) => ({ ... })`**: A function that generates the HTTP headers for the request.
        *   `Authorization: \`Bearer ${params.accessToken}\``: This is the critical authentication header. It uses the `accessToken` obtained via OAuth to prove the user's identity and authorization. The `Bearer` scheme is a common way to send access tokens.
        *   `'Content-Type': 'application/json'`: Specifies that the request body will be in JSON format.
    *   **`body: (params: GoogleCalendarCreateParams): GoogleCalendarEventRequestBody => { ... }`**: A function that constructs the request body, which will be sent as JSON to the Google Calendar API. It takes the tool `params` and returns an object conforming to `GoogleCalendarEventRequestBody`.
        *   `const eventData: GoogleCalendarEventRequestBody = { ... }`: Initializes the base event object with mandatory fields.
            *   `summary: params.summary`: The event title.
            *   `start: { dateTime: params.startDateTime }`: The start time, using the RFC3339 format.
            *   `end: { dateTime: params.endDateTime }`: The end time, using the RFC3339 format.
        *   `if (params.description) { eventData.description = params.description }`: Conditionally adds the description if provided.
        *   `if (params.location) { eventData.location = params.location }`: Conditionally adds the location if provided.
        *   `if (params.timeZone) { eventData.start.timeZone = params.timeZone; eventData.end.timeZone = params.timeZone }`: If a time zone is specified, it's applied to both the start and end times.
        *   **Simplified `attendees` logic explanation:** This is the most complex part of the `body` function.
            *   The `params.attendees` input can sometimes be a single string (e.g., "john@example.com, jane@example.com") or an array of strings (`["john@example.com", "jane@example.com"]`). The code handles both.
            *   `let attendeeList: string[] = []`: Initializes an empty array to hold cleaned email addresses.
            *   `if (params.attendees)`: Checks if any attendees were provided.
            *   `const attendees = params.attendees as string | string[]`: Type assertion to tell TypeScript that `params.attendees` could be either a string or a string array.
            *   `if (Array.isArray(attendees))`: If it's already an array, it filters out any empty or whitespace-only email strings.
            *   `else if (typeof attendees === 'string' && attendees.trim().length > 0)`: If it's a non-empty string:
                *   `.split(',')`: Splits the string by commas.
                *   `.map((email: string) => email.trim())`: Trims whitespace from each resulting email.
                *   `.filter((email: string) => email.length > 0)`: Filters out any empty strings that might result from extra commas (e.g., "a,,b").
            *   `if (attendeeList.length > 0) { eventData.attendees = attendeeList.map((email: string) => ({ email })) }`: If there are valid attendees, it maps each email string into an object of the format `{ email: "email@example.com" }`, which is what the Google Calendar API expects for attendees.
        *   `return eventData`: Returns the final constructed request body.

---

```typescript
  transformResponse: async (response: Response) => {
    const data: GoogleCalendarApiEventResponse = await response.json()

    return {
      success: true,
      output: {
        content: `Event "${data.summary}" created successfully`,
        metadata: {
          id: data.id,
          htmlLink: data.htmlLink,
          status: data.status,
          summary: data.summary,
          description: data.description,
          location: data.location,
          start: data.start,
          end: data.end,
          attendees: data.attendees,
          creator: data.creator,
          organizer: data.organizer,
        },
      },
    }
  },
```

*   **`transformResponse: async (response: Response) => { ... }`**: This asynchronous function takes the raw HTTP `Response` object received from the Google Calendar API and transforms it into a standardized output format for the calling application.
    *   `const data: GoogleCalendarApiEventResponse = await response.json()`: Parses the JSON body of the API response into a JavaScript object, asserting its type as `GoogleCalendarApiEventResponse`.
    *   `return { success: true, output: { ... } }`: Returns a structured object indicating the success of the operation and the actual output.
        *   `success: true`: Indicates the API call was successful.
        *   `output: { ... }`: The main output object.
            *   `content: \`Event "${data.summary}" created successfully\``: A user-friendly string message confirming the event creation, using the event's summary from the API response.
            *   `metadata: { ... }`: A detailed object containing key information about the newly created event, directly extracted from the Google Calendar API response. This includes the `id` (unique event identifier), `htmlLink` (URL to view the event in Google Calendar), `status`, `summary`, `description`, `location`, `start`, `end`, `attendees`, `creator`, and `organizer`. This metadata is valuable for further actions or detailed logging.

---

```typescript
  outputs: {
    content: { type: 'string', description: 'Event creation confirmation message' },
    metadata: {
      type: 'json',
      description: 'Created event metadata including ID, status, and details',
    },
  },
}
```

*   **`outputs: { ... }`**: This object defines the *schema* of the data that this tool will return upon successful execution. This is useful for the consuming application to understand what kind of data it will receive.
    *   **`content`**:
        *   `type: 'string'`: Specifies that the `content` part of the output will be a string.
        *   `description: 'Event creation confirmation message'`: Describes what this string represents.
    *   **`metadata`**:
        *   `type: 'json'`: Specifies that the `metadata` part of the output will be a JSON object.
        *   `description: 'Created event metadata including ID, status, and details'`: Describes the contents of the metadata.

---

### **Simplified Complex Logic: `attendees` Handling**

The most intricate part of this code is how it handles the `attendees` parameter within the `body` function. The Google Calendar API expects attendees as an array of objects like `[{ email: "email1" }, { email: "email2" }]`. However, users or LLMs might provide attendee information in various ways:

1.  **As an actual array of strings**: `["john@example.com", "jane@example.com"]`
2.  **As a comma-separated string**: `"john@example.com, jane@example.com"`
3.  **With potential extra spaces or empty entries**.

This code elegantly handles all these scenarios:

1.  **It first checks if `attendees` exist at all.**
2.  **It then checks if it's already an array.** If so, it cleans up the array by removing any empty or purely whitespace emails.
3.  **If it's not an array, it assumes it might be a string.** It checks if the string is non-empty after trimming whitespace.
4.  **If it's a non-empty string**, it splits the string by commas, trims whitespace from each resulting email, and then filters out any empty email strings that might have resulted from multiple commas (e.g., "a,,b" would produce "a" and "b", discarding the empty middle).
5.  **Finally, if any valid attendees remain**, it transforms the cleaned list of email strings into the format Google Calendar expects: an array of objects where each object has an `email` property.

This flexible handling makes the tool more robust and user-friendly, as it can gracefully accept different input formats for attendees.