This TypeScript code defines a "tool" configuration for interacting with the Google Calendar API. Specifically, it configures a tool to **retrieve a single event** from a Google Calendar.

Imagine you're building a platform that needs to integrate with various services. Instead of writing custom code for each service every time, you define a standardized "tool" interface. This file is one such definition, telling your platform *how* to call the Google Calendar API to get an event, *what inputs* it needs, and *what outputs* it will provide.

---

### **Purpose of this File**

The primary purpose of this file is to **declare a reusable, self-contained configuration for fetching a specific event from Google Calendar**. It acts as a blueprint for a "Google Calendar Get Event" tool, specifying:

1.  **Identity:** Its unique ID, name, description, and version.
2.  **Authentication:** That it requires OAuth with Google Calendar and the specific permissions needed.
3.  **Input Parameters:** What information the user or system needs to provide to use this tool (e.g., event ID, calendar ID).
4.  **API Request:** How to construct the actual HTTP request to Google's API (URL, method, headers).
5.  **Response Transformation:** How to process the raw response from Google's API into a standardized, clean output for the platform.
6.  **Output Description:** What the tool will return as a result.

This structured approach makes it easier to integrate, manage, and use various external services within a larger application or platform.

---

### **Simplified Explanation of Complex Logic**

The most "complex" parts are often related to dynamic behavior:

1.  **Dynamic URL Construction (`request.url`):** The tool doesn't just call a fixed URL. It dynamically builds the URL based on the `calendarId` and `eventId` provided by the user. It also ensures these IDs are safely encoded for web URLs using `encodeURIComponent`. If no `calendarId` is given, it intelligently defaults to `'primary'`, which typically refers to the user's main calendar.
2.  **Dynamic Headers (`request.headers`):** Similarly, the `Authorization` header, essential for secure API calls, is built dynamically using the `accessToken` provided by the user, ensuring the request is authenticated.
3.  **Response Transformation (`transformResponse`):** The Google Calendar API sends back a lot of data in a specific format (`GoogleCalendarApiEventResponse`). This tool takes that raw data, extracts the most relevant pieces, and repackages them into a more user-friendly, standardized output format (`GoogleCalendarGetResponse`). This step is crucial for abstracting away API-specific details from the end-user of the tool.

---

### **Line-by-Line Explanation**

```typescript
import {
  CALENDAR_API_BASE,
  type GoogleCalendarApiEventResponse,
  type GoogleCalendarGetParams,
  type GoogleCalendarGetResponse,
} from '@/tools/google_calendar/types'
import type { ToolConfig } from '@/tools/types'
```
*   **`import ... from '@/tools/google_calendar/types'`**: This line imports various types and constants specifically related to Google Calendar integration from a local `types` file.
    *   `CALENDAR_API_BASE`: A constant string representing the base URL for the Google Calendar API (e.g., `https://www.googleapis.com/calendar/v3`).
    *   `type GoogleCalendarApiEventResponse`: A TypeScript type definition that describes the structure of the *raw* event data returned directly by the Google Calendar API.
    *   `type GoogleCalendarGetParams`: A TypeScript type definition that describes the structure of the *input parameters* expected by this specific tool (e.g., `accessToken`, `calendarId`, `eventId`).
    *   `type GoogleCalendarGetResponse`: A TypeScript type definition that describes the structure of the *final output* this tool will produce after processing the API response.
*   **`import type { ToolConfig } from '@/tools/types'`**: This line imports the core `ToolConfig` type from a more general `types` file. `ToolConfig` is a generic type that defines the overall structure for *any* tool configuration within the platform, making sure all tools adhere to a consistent interface.

---

```typescript
export const getTool: ToolConfig<GoogleCalendarGetParams, GoogleCalendarGetResponse> = {
```
*   **`export const getTool`**: This declares a constant variable named `getTool` and makes it available for other files to import and use. This `getTool` object holds the entire configuration for our specific Google Calendar event retrieval tool.
*   **`: ToolConfig<GoogleCalendarGetParams, GoogleCalendarGetResponse>`**: This is a TypeScript type annotation. It specifies that `getTool` must conform to the `ToolConfig` interface. The `<GoogleCalendarGetParams, GoogleCalendarGetResponse>` part are generics, telling `ToolConfig` what specific types to expect for the tool's input parameters and its final output, respectively.

---

```typescript
  id: 'google_calendar_get',
  name: 'Google Calendar Get Event',
  description: 'Get a specific event from Google Calendar',
  version: '1.0.0',
```
*   **`id: 'google_calendar_get'`**: A unique identifier string for this tool within the platform.
*   **`name: 'Google Calendar Get Event'`**: A human-readable name for the tool, often used in user interfaces.
*   **`description: 'Get a specific event from Google Calendar'`**: A brief explanation of what the tool does.
*   **`version: '1.0.0'`**: The version number of this tool configuration, useful for managing updates and compatibility.

---

```typescript
  oauth: {
    required: true,
    provider: 'google-calendar',
    additionalScopes: ['https://www.googleapis.com/auth/calendar'],
  },
```
*   **`oauth`**: This object defines the OAuth (Open Authorization) requirements for this tool.
    *   **`required: true`**: Indicates that this tool *requires* OAuth authentication to function. Without an access token obtained via OAuth, it cannot be used.
    *   **`provider: 'google-calendar'`**: Specifies which OAuth provider to use. This tells the platform to use the pre-configured Google Calendar OAuth flow.
    *   **`additionalScopes: ['https://www.googleapis.com/auth/calendar']`**: An array of specific OAuth "scopes" (permissions) that the tool needs. `'https://www.googleapis.com/auth/calendar'` grants permission to view and manage events on calendars that the user has access to.

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
    eventId: {
      type: 'string',
      required: true,
      visibility: 'user-only',
      description: 'Event ID to retrieve',
    },
  },
```
*   **`params`**: This object defines the input parameters that the tool expects when it's invoked. Each property within `params` is a parameter definition.
    *   **`accessToken`**:
        *   **`type: 'string'`**: The parameter must be a string.
        *   **`required: true`**: This parameter *must* be provided for the tool to work.
        *   **`visibility: 'hidden'`**: This parameter is typically handled internally by the platform (e.g., automatically injected after an OAuth flow) and should not be directly exposed to end-users.
        *   **`description: 'Access token for Google Calendar API'`**: A short explanation of the parameter's purpose.
    *   **`calendarId`**:
        *   **`type: 'string'`**: The parameter must be a string.
        *   **`required: false`**: This parameter is optional. If not provided, the tool will use a default value (as seen in the `request.url` section).
        *   **`visibility: 'user-only'`**: This parameter can be provided by the user of the tool.
        *   **`description: 'Calendar ID (defaults to primary)'`**: Explains its purpose and the default behavior.
    *   **`eventId`**:
        *   **`type: 'string'`**: The parameter must be a string.
        *   **`required: true`**: This parameter is mandatory; you cannot retrieve an event without its ID.
        *   **`visibility: 'user-only'`**: This parameter can be provided by the user of the tool.
        *   **`description: 'Event ID to retrieve'`**: Explains its purpose.

---

```typescript
  request: {
    url: (params: GoogleCalendarGetParams) => {
      const calendarId = params.calendarId || 'primary'
      return `${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events/${encodeURIComponent(params.eventId)}`
    },
    method: 'GET',
    headers: (params: GoogleCalendarGetParams) => ({
      Authorization: `Bearer ${params.accessToken}`,
      'Content-Type': 'application/json',
    }),
  },
```
*   **`request`**: This object defines how to construct the actual HTTP request to the external API.
    *   **`url: (params: GoogleCalendarGetParams) => { ... }`**: This is a function that dynamically generates the URL for the API request. It takes the tool's input `params` as an argument.
        *   **`const calendarId = params.calendarId || 'primary'`**: This line gets the `calendarId` from the input `params`. If `params.calendarId` is undefined or an empty string (falsy), it defaults to the string `'primary'`, which in Google Calendar typically refers to the user's default calendar.
        *   **`return `${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events/${encodeURIComponent(params.eventId)}``**: This constructs the full API URL using a template literal.
            *   `${CALENDAR_API_BASE}`: The base URL for the Google Calendar API.
            *   `/calendars/${encodeURIComponent(calendarId)}`: Appends the calendar ID, ensuring it's safely encoded for use in a URL (e.g., if it contains special characters).
            *   `/events/${encodeURIComponent(params.eventId)}`: Appends the event ID, also safely encoded.
    *   **`method: 'GET'`**: Specifies the HTTP method to use for the request, which is `GET` for retrieving data.
    *   **`headers: (params: GoogleCalendarGetParams) => ({ ... })`**: This is a function that dynamically generates the HTTP headers for the request, also taking the `params` as input.
        *   **`Authorization: `Bearer ${params.accessToken}``**: This sets the `Authorization` header, which is crucial for authenticating the request. It uses a "Bearer" token scheme, where the `accessToken` obtained via OAuth is included.
        *   **`'Content-Type': 'application/json'`**: This header indicates that the client expects to receive a JSON response. (While not strictly necessary for a GET request which has no body, it's often good practice to include it).

---

```typescript
  transformResponse: async (response: Response) => {
    const data: GoogleCalendarApiEventResponse = await response.json()

    return {
      success: true,
      output: {
        content: `Retrieved event "${data.summary}"`,
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
*   **`transformResponse: async (response: Response) => { ... }`**: This is an asynchronous function responsible for taking the raw HTTP `Response` from the API call and transforming it into the tool's standardized output format (`GoogleCalendarGetResponse`).
    *   **`const data: GoogleCalendarApiEventResponse = await response.json()`**: This line parses the body of the raw HTTP `response` object as JSON. The result is awaited because parsing JSON is an asynchronous operation. The `data` variable is typed as `GoogleCalendarApiEventResponse`, indicating its expected structure from the Google API.
    *   **`return { ... }`**: This returns the final, structured output of the tool.
        *   **`success: true`**: A boolean flag indicating that the operation was successful.
        *   **`output: { ... }`**: This object contains the actual results of the tool's execution.
            *   **`content: `Retrieved event "${data.summary}"``**: A user-friendly string message summarizing the outcome, using the event's `summary` from the API response.
            *   **`metadata: { ... }`**: A structured JSON object containing key details extracted from the raw API response (`data`). This makes the most relevant information easily accessible without exposing the entire, potentially verbose, API response. Each property (e.g., `id`, `htmlLink`, `status`, `summary`, etc.) directly maps or selects a field from the `data` object.

---

```typescript
  outputs: {
    content: { type: 'string', description: 'Event retrieval confirmation message' },
    metadata: {
      type: 'json',
      description: 'Event details including ID, status, times, and attendees',
    },
  },
}
```
*   **`outputs`**: This object defines the structure and description of the data that this tool will return. This is useful for documentation and for other parts of the platform that consume the tool's output.
    *   **`content`**:
        *   **`type: 'string'`**: The `content` field will be a string.
        *   **`description: 'Event retrieval confirmation message'`**: Explains what this string represents.
    *   **`metadata`**:
        *   **`type: 'json'`**: The `metadata` field will be a JSON object.
        *   **`description: 'Event details including ID, status, times, and attendees'`**: Explains what kind of structured data is contained within the `metadata` object.

---

In essence, this file provides a robust, standardized, and self-documenting way for an application to get a Google Calendar event, abstracting away the specifics of the Google API call itself.