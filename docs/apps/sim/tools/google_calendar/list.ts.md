This TypeScript file defines a *configuration object* for a specific "tool" designed to interact with the Google Calendar API. Specifically, this tool's purpose is to **list events from a Google Calendar**.

It's structured in a way that suggests it's part of a larger framework or platform. This framework likely consumes `ToolConfig` objects, handles their execution (making API requests), manages authentication, and processes the results according to the definitions provided in this file.

Let's break down the code in detail.

---

### **Purpose of this File**

The primary purpose of this file is to *declare* and *configure* a "Google Calendar List Events" tool. This configuration includes:
1.  **Metadata:** Basic information about the tool (ID, name, description, version).
2.  **Authentication Requirements:** Specifies that Google OAuth is needed and what specific permissions (scopes) are required.
3.  **Input Parameters:** Defines what information the tool expects to receive from the user or another system to perform its function (e.g., `calendarId`, `timeMin`).
4.  **Request Construction Logic:** How to translate the input parameters into an actual HTTP request (URL, method, headers) to the Google Calendar API.
5.  **Response Transformation Logic:** How to take the raw JSON response from the Google Calendar API and convert it into a standardized, usable output format for the platform.
6.  **Output Description:** Describes the structure of the data the tool will return.

In essence, it's a blueprint for a micro-service or function within a larger application that needs to interact with Google Calendar.

---

### **Detailed Explanation**

```typescript
import {
  CALENDAR_API_BASE,
  type GoogleCalendarApiEventResponse,
  type GoogleCalendarApiListResponse,
  type GoogleCalendarListParams,
  type GoogleCalendarListResponse,
} from '@/tools/google_calendar/types'
import type { ToolConfig } from '@/tools/types'
```
*   **`import { ... } from '@/tools/google_calendar/types'`**: This line imports various TypeScript types and a constant (`CALENDAR_API_BASE`) from a local module that likely defines data structures specific to interacting with the Google Calendar API.
    *   `CALENDAR_API_BASE`: A string constant representing the base URL for the Google Calendar API (e.g., `https://www.googleapis.com/calendar/v3`).
    *   `GoogleCalendarApiEventResponse`: The TypeScript type that describes the structure of a *single event* object returned directly by the Google Calendar API.
    *   `GoogleCalendarApiListResponse`: The TypeScript type for the *entire response body* when listing events from the Google Calendar API. It's a container for multiple `GoogleCalendarApiEventResponse` objects, along with pagination and sync tokens.
    *   `GoogleCalendarListParams`: The TypeScript type defining the structure of the *input parameters* this specific tool expects. This ensures type safety when defining and using the `params` property later.
    *   `GoogleCalendarListResponse`: The TypeScript type for the *final, transformed output* that this tool will produce. This is the standardized response structure of the tool itself, not the raw API response.
*   **`import type { ToolConfig } from '@/tools/types'`**: This imports the core `ToolConfig` type from a general `tools` module. This type defines the overall structure that all tools in the platform must adhere to. The `type` keyword ensures this is only imported for type checking and doesn't generate any runtime JavaScript code.

---

```typescript
export const listTool: ToolConfig<GoogleCalendarListParams, GoogleCalendarListResponse> = {
  id: 'google_calendar_list',
  name: 'Google Calendar List Events',
  description: 'List events from Google Calendar',
  version: '1.0.0',
```
*   **`export const listTool: ToolConfig<GoogleCalendarListParams, GoogleCalendarListResponse> = { ... }`**: This declares and exports a constant variable named `listTool`.
    *   `export`: Makes this `listTool` object available for other files to import and use.
    *   `const`: Declares a constant, meaning its value cannot be reassigned after initialization.
    *   `listTool`: The name of our tool's configuration object.
    *   `: ToolConfig<GoogleCalendarListParams, GoogleCalendarListResponse>`: This is a type annotation. It specifies that `listTool` must conform to the `ToolConfig` interface. The `<GoogleCalendarListParams, GoogleCalendarListResponse>` part uses TypeScript generics, indicating that:
        *   `GoogleCalendarListParams` is the type for the *input parameters* this specific tool expects.
        *   `GoogleCalendarListResponse` is the type for the *final output* this specific tool will produce.
*   **`id: 'google_calendar_list'`**: A unique identifier for this tool within the larger system.
*   **`name: 'Google Calendar List Events'`**: A human-readable name for the tool, likely used in UIs or logs.
*   **`description: 'List events from Google Calendar'`**: A brief explanation of what the tool does.
*   **`version: '1.0.0'`**: The version of this tool definition.

---

```typescript
  oauth: {
    required: true,
    provider: 'google-calendar',
    additionalScopes: ['https://www.googleapis.com/auth/calendar'],
  },
```
*   **`oauth: { ... }`**: This section specifies the authentication requirements for this tool. It indicates that the tool needs to use OAuth 2.0 to access Google Calendar.
    *   **`required: true`**: Means that this tool *must* be authenticated via OAuth to function.
    *   **`provider: 'google-calendar'`**: Identifies the OAuth provider. The platform presumably has a way to handle authentication flows for different providers (e.g., Google, GitHub, etc.).
    *   **`additionalScopes: ['https://www.googleapis.com/auth/calendar']`**: This is crucial. It specifies the OAuth scopes (permissions) that the user must grant. `https://www.googleapis.com/auth/calendar` is the scope required to view and manage events on Google Calendar. The platform would use this to request the necessary permissions from the user during the OAuth consent flow.

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
    timeMin: {
      type: 'string',
      required: false,
      visibility: 'user-or-llm',
      description: 'Lower bound for events (RFC3339 timestamp, e.g., 2025-06-03T00:00:00Z)',
    },
    timeMax: {
      type: 'string',
      required: false,
      visibility: 'user-or-llm',
      description: 'Upper bound for events (RFC3339 timestamp, e.g., 2025-06-04T00:00:00Z)',
    },
    orderBy: {
      type: 'string',
      required: false,
      visibility: 'hidden',
      description: 'Order of events returned (startTime or updated)',
    },
    showDeleted: {
      type: 'boolean',
      required: false,
      visibility: 'hidden',
      description: 'Include deleted events',
    },
  },
```
*   **`params: { ... }`**: This object defines all the input parameters that the tool expects. Each key is the parameter name, and its value is an object describing the parameter. This section adheres to the `GoogleCalendarListParams` type.
    *   **`accessToken`**:
        *   `type: 'string'`: The parameter is a string.
        *   `required: true`: This parameter *must* be provided for the tool to work.
        *   `visibility: 'hidden'`: This parameter is typically managed by the platform (e.g., after a successful OAuth flow) and should not be exposed directly to a user or even a large language model (LLM) for manual input. It's an internal detail.
        *   `description: 'Access token for Google Calendar API'`: Explains its purpose.
    *   **`calendarId`**:
        *   `type: 'string'`: String value.
        *   `required: false`: This parameter is optional. If not provided, the code will default to `'primary'`.
        *   `visibility: 'user-only'`: This parameter can be provided by a direct user of the platform (e.g., through a UI form), but an LLM (if this is part of an AI agent system) should not be prompted to generate it.
        *   `description: 'Calendar ID (defaults to primary)'`: Explains its purpose and default behavior.
    *   **`timeMin`, `timeMax`**: These define a time range for filtering events.
        *   `type: 'string'`: Expects an RFC3339 formatted timestamp string.
        *   `required: false`: Optional.
        *   `visibility: 'user-or-llm'`: This parameter can be provided by a direct user *or* by an LLM. This suggests an AI agent could infer and provide these timestamps based on a user's natural language request (e.g., "list my events for tomorrow").
        *   `description`: Provides format guidance.
    *   **`orderBy`**:
        *   `type: 'string'`: String value.
        *   `required: false`: Optional.
        *   `visibility: 'hidden'`: Like `accessToken`, this is an internal or advanced option not meant for direct user/LLM input.
        *   `description`: Specifies possible values (`startTime` or `updated`).
    *   **`showDeleted`**:
        *   `type: 'boolean'`: Boolean value.
        *   `required: false`: Optional.
        *   `visibility: 'hidden'`: Hidden from direct user/LLM input.
        *   `description`: Indicates whether to include events marked as deleted.

---

```typescript
  request: {
    url: (params: GoogleCalendarListParams) => {
      const calendarId = params.calendarId || 'primary'
      const queryParams = new URLSearchParams()

      if (params.timeMin) queryParams.append('timeMin', params.timeMin)
      if (params.timeMax) queryParams.append('timeMax', params.timeMax)
      queryParams.append('singleEvents', 'true')
      if (params.orderBy) queryParams.append('orderBy', params.orderBy)
      if (params.showDeleted !== undefined)
        queryParams.append('showDeleted', params.showDeleted.toString())

      const queryString = queryParams.toString()
      return `${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events${queryString ? `?${queryString}` : ''}`
    },
    method: 'GET',
    headers: (params: GoogleCalendarListParams) => ({
      Authorization: `Bearer ${params.accessToken}`,
      'Content-Type': 'application/json',
    }),
  },
```
*   **`request: { ... }`**: This object defines how to construct the actual HTTP request to the Google Calendar API.
    *   **`url: (params: GoogleCalendarListParams) => { ... }`**: This is a function that takes the tool's input parameters (`params`) and returns the full URL for the API request.
        *   **`const calendarId = params.calendarId || 'primary'`**: If `params.calendarId` is not provided (is `null`, `undefined`, or an empty string), it defaults to `'primary'`, which typically refers to the user's main calendar in Google Calendar.
        *   **`const queryParams = new URLSearchParams()`**: Creates a new instance of `URLSearchParams`. This built-in browser/Node.js class is used to easily construct URL query strings (e.g., `?key1=value1&key2=value2`).
        *   **`if (params.timeMin) queryParams.append('timeMin', params.timeMin)`**: If `timeMin` is provided in the input parameters, it's added as a query parameter to the URL.
        *   **`if (params.timeMax) queryParams.append('timeMax', params.timeMax)`**: Same for `timeMax`.
        *   **`queryParams.append('singleEvents', 'true')`**: This is a mandatory Google Calendar API parameter. It tells the API to expand recurring events into individual occurrences rather than just returning the recurring event master.
        *   **`if (params.orderBy) queryParams.append('orderBy', params.orderBy)`**: If `orderBy` is provided, it's added.
        *   **`if (params.showDeleted !== undefined) queryParams.append('showDeleted', params.showDeleted.toString())`**: If `showDeleted` is provided (even if `false`), it's added. It explicitly converts the boolean `true` or `false` to a string `'true'` or `'false'`, as required for URL query parameters.
        *   **`const queryString = queryParams.toString()`**: Converts the `URLSearchParams` object into a properly formatted query string (e.g., `timeMin=...&singleEvents=true`).
        *   **`return `${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events${queryString ? `?${queryString}` : ''}``**: This line constructs the final URL using a template literal:
            *   `${CALENDAR_API_BASE}`: The base API URL.
            *   `/calendars/`: Standard path segment.
            *   `${encodeURIComponent(calendarId)}`: The `calendarId` (e.g., 'primary' or a specific ID) is included in the URL path. `encodeURIComponent` is crucial here to ensure that any special characters in the `calendarId` (though unlikely for 'primary') are properly escaped for the URL.
            *   `/events`: The endpoint to list events for a specific calendar.
            *   `${queryString ? `?${queryString}` : ''}`: This is a conditional addition. If `queryString` is not empty (i.e., there are query parameters), it prepends a `?` and appends the `queryString`. Otherwise, it adds nothing.
    *   **`method: 'GET'`**: Specifies that this API request will use the HTTP GET method, which is standard for retrieving data.
    *   **`headers: (params: GoogleCalendarListParams) => ({ ... })`**: This is a function that takes the `params` and returns an object of HTTP headers for the request.
        *   **`Authorization: `Bearer ${params.accessToken}``**: This is the standard way to send an OAuth 2.0 access token. The `Bearer` scheme indicates that the token grants bearer access, meaning whoever possesses the token can use it. `params.accessToken` is the token obtained from the OAuth flow.
        *   **`'Content-Type': 'application/json'`**: While not strictly necessary for a GET request without a body, it's good practice to specify that the client expects/sends JSON, especially in a common header function.

---

```typescript
  transformResponse: async (response: Response) => {
    const data: GoogleCalendarApiListResponse = await response.json()
    const events = data.items || []
    const eventsCount = events.length

    return {
      success: true,
      output: {
        content: `Found ${eventsCount} event${eventsCount !== 1 ? 's' : ''}`,
        metadata: {
          nextPageToken: data.nextPageToken,
          nextSyncToken: data.nextSyncToken,
          timeZone: data.timeZone,
          events: events.map((event: GoogleCalendarApiEventResponse) => ({
            id: event.id,
            htmlLink: event.htmlLink,
            status: event.status,
            summary: event.summary || 'No title',
            description: event.description,
            location: event.location,
            start: event.start,
            end: event.end,
            attendees: event.attendees,
            creator: event.creator,
            organizer: event.organizer,
          })),
        },
      },
    }
  },
```
*   **`transformResponse: async (response: Response) => { ... }`**: This is an asynchronous function responsible for processing the raw HTTP response received from the Google Calendar API and transforming it into the tool's standardized output format (`GoogleCalendarListResponse`).
    *   **`async`**: Indicates that this function will perform asynchronous operations (like `await response.json()`).
    *   **`(response: Response)`**: It takes a standard Web `Response` object as its input, which contains the raw HTTP response from the API call.
    *   **`const data: GoogleCalendarApiListResponse = await response.json()`**: Reads the body of the `Response` object as JSON and parses it into a JavaScript object. The type assertion `: GoogleCalendarApiListResponse` tells TypeScript what structure to expect.
    *   **`const events = data.items || []`**: The Google Calendar API typically returns an array of events in a property called `items`. This line extracts that array. The `|| []` is a defensive programming technique: if `data.items` is `null` or `undefined` (e.g., if no events are found or an API error occurred that didn't prevent JSON parsing), `events` will be initialized as an empty array, preventing errors later.
    *   **`const eventsCount = events.length`**: Calculates the number of events found.
    *   **`return { ... }`**: This returns the final, standardized output object as defined by `GoogleCalendarListResponse`.
        *   **`success: true`**: Indicates that the tool execution itself was successful (i.e., the API call completed and the response was processed, even if no events were found).
        *   **`output: { ... }`**: This is the main payload of the tool's output.
            *   **`content: `Found ${eventsCount} event${eventsCount !== 1 ? 's' : ''}``**: A human-readable string summarizing the result. It correctly handles pluralization ("event" vs. "events"). This `content` is likely intended for display to a user or as a summary for an LLM.
            *   **`metadata: { ... }`**: This object contains the structured data from the API response, transformed and curated.
                *   **`nextPageToken: data.nextPageToken`**: Used for pagination if there are more results than returned in a single response.
                *   **`nextSyncToken: data.nextSyncToken`**: Used for incremental synchronization of calendar data.
                *   **`timeZone: data.timeZone`**: The time zone of the calendar.
                *   **`events: events.map(...)`**: This is a key transformation step. It iterates over each raw `event` object (`GoogleCalendarApiEventResponse`) received from the Google API and *maps* it to a new, simplified event object. This is good practice for several reasons:
                    *   **Data pruning:** Only relevant fields are kept, reducing the payload size and complexity for downstream consumers.
                    *   **Standardization:** Ensures the output format is consistent, regardless of potential variations in the raw API response.
                    *   **Defaults:** `summary: event.summary || 'No title'` provides a default "No title" if an event's summary (title) is missing from the API response.
                    *   It extracts fields like `id`, `htmlLink`, `status`, `summary`, `description`, `location`, `start`, `end`, `attendees`, `creator`, and `organizer`.

---

```typescript
  outputs: {
    content: { type: 'string', description: 'Summary of found events count' },
    metadata: {
      type: 'json',
      description: 'List of events with pagination tokens and event details',
    },
  },
}
```
*   **`outputs: { ... }`**: This section provides a description of the tool's expected output structure, mirroring what's returned by `transformResponse`. This is useful for documentation, UI generation, or for an LLM to understand what data it will receive.
    *   **`content: { type: 'string', description: 'Summary of found events count' }`**: Describes the `content` field of the output as a string containing a summary.
    *   **`metadata: { type: 'json', description: 'List of events with pagination tokens and event details' }`**: Describes the `metadata` field of the output. It indicates it's a JSON object containing the detailed event list and other relevant tokens.

---

### **Simplified Complex Logic**

The `listTool` object, while seemingly large, is essentially a **declarative recipe** for making an API call and handling its results. Instead of writing imperative code that directly performs these steps every time, this configuration *describes* how the steps should be done. A generic "tool executor" can then read this recipe and execute it.

1.  **Input Parameters (`params`):** It's like defining the ingredients for a dish. Each ingredient has a type, whether it's essential or optional, and who is allowed to add it (user, AI, or hidden).
2.  **Request Construction (`request`):** This is the "cooking instructions" part.
    *   The `url` function builds the precise URL for the Google Calendar API. It intelligently adds query parameters only if they are provided, handles a default calendar, and ensures URL parts are correctly encoded.
    *   The `headers` function adds necessary authentication information (the `Bearer` token) to the request.
3.  **Response Transformation (`transformResponse`):** This is the "plating and serving" part.
    *   It takes the raw response (the "cooked meal") and processes it.
    *   It parses the JSON, extracts the key information (the event list), and formats it into a clean, concise, and standardized output.
    *   It also adds a human-friendly summary (`content`) and keeps important meta-information (like pagination tokens) in `metadata`. This separation allows a user to quickly see a summary, while a system or AI can delve into the structured `metadata`.

By using this `ToolConfig` pattern, the system gains modularity, reusability, and makes it easier to manage and understand how different tools interact with external APIs.