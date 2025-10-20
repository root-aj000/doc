This TypeScript file defines a "tool" configuration for interacting with the Google Calendar API. Specifically, it describes how to update an *existing* event in a user's Google Calendar.

This configuration is likely used within a larger application (perhaps an AI agent, an automation platform, or a custom integration engine) that needs to know:
1.  **What this tool does**: Its name, description, and version.
2.  **What inputs it requires**: Parameters like event ID, new summary, dates, attendees, and authentication tokens.
3.  **How to authenticate**: Google OAuth requirements.
4.  **How to make the API calls**: The URLs, HTTP methods, headers.
5.  **How to process the API responses**: Merging updates, handling different data types, and formatting the final output.

---

### **Simplified Explanation of the Complex Logic (`transformResponse` function)**

The `transformResponse` function is the most complex part of this file, but its core logic can be simplified into these steps:

1.  **Fetch the Existing Event**: Google Calendar's API for updating events (`PUT`) typically expects the *entire* event object, not just the fields you want to change. So, the first step is to make a `GET` request to retrieve all the details of the event you intend to update.
2.  **Prepare the Update**: Once the existing event data is retrieved, a copy is made. Then, this copy is selectively updated with only the parameters provided by the user (e.g., if the user only provides a new `summary`, only the `summary` field is changed; other fields remain as they were in the original event). Special care is taken for date/time fields (which must be updated together) and attendees (which are completely replaced).
3.  **Clean Up**: Read-only fields (like `id`, `created`, `updated` timestamps) that Google doesn't allow you to send back in an update request are removed from the prepared event object.
4.  **Send the Update**: Finally, a `PUT` request is sent to the Google Calendar API with the merged and cleaned-up event object.
5.  **Handle Response**: The response from this `PUT` request is processed. If successful, a confirmation message and the updated event's metadata are returned. If an error occurs, an error message is thrown.

---

### **Detailed Line-by-Line Explanation**

```typescript
import {
  CALENDAR_API_BASE,
  type GoogleCalendarToolResponse,
  type GoogleCalendarUpdateParams,
} from '@/tools/google_calendar/types'
import type { ToolConfig } from '@/tools/types'
```

*   **`import { ... } from '@/tools/google_calendar/types'`**: This line imports specific types and constants from a local file (`src/tools/google_calendar/types`).
    *   **`CALENDAR_API_BASE`**: A constant string representing the base URL for the Google Calendar API (e.g., `https://www.googleapis.com/calendar/v3`).
    *   **`type GoogleCalendarToolResponse`**: A TypeScript type definition that describes the structure of the final output returned by this tool.
    *   **`type GoogleCalendarUpdateParams`**: A TypeScript type definition that describes the expected input parameters for updating a Google Calendar event.
*   **`import type { ToolConfig } from '@/tools/types'`**: This imports the `ToolConfig` type, which is a generic type defining the overall structure for any tool within this system. The `type` keyword ensures this import is only for type-checking and doesn't generate any runtime JavaScript code.

---

```typescript
export const updateTool: ToolConfig<GoogleCalendarUpdateParams, GoogleCalendarToolResponse> = {
  id: 'google_calendar_update',
  name: 'Google Calendar Update Event',
  description: 'Update an existing event in Google Calendar',
  version: '1.0.0',
```

*   **`export const updateTool: ToolConfig<GoogleCalendarUpdateParams, GoogleCalendarToolResponse> = { ... }`**: This line defines and exports a constant variable named `updateTool`.
    *   **`export`**: Makes this `updateTool` object available for other files to import.
    *   **`const updateTool`**: Declares a constant variable named `updateTool`.
    *   **`: ToolConfig<GoogleCalendarUpdateParams, GoogleCalendarToolResponse>`**: This is a TypeScript type annotation. It specifies that `updateTool` must conform to the `ToolConfig` interface.
        *   `GoogleCalendarUpdateParams`: The first type argument, indicating the type of parameters this tool expects as input.
        *   `GoogleCalendarToolResponse`: The second type argument, indicating the type of response this tool will return.
*   **`id: 'google_calendar_update'`**: A unique identifier for this specific tool.
*   **`name: 'Google Calendar Update Event'`**: A human-readable name for the tool.
*   **`description: 'Update an existing event in Google Calendar'`**: A brief explanation of what the tool does.
*   **`version: '1.0.0'`**: The version of this tool configuration.

---

```typescript
  oauth: {
    required: true,
    provider: 'google-calendar',
    additionalScopes: ['https://www.googleapis.com/auth/calendar'],
  },
```

*   **`oauth: { ... }`**: This section defines the OAuth (Open Authorization) requirements for using this tool.
    *   **`required: true`**: Indicates that an OAuth access token is mandatory for this tool to function.
    *   **`provider: 'google-calendar'`**: Specifies which OAuth provider should be used to obtain the access token.
    *   **`additionalScopes: ['https://www.googleapis.com/auth/calendar']`**: An array of additional permissions (scopes) required from the user during the OAuth consent process. `'https://www.googleapis.com/auth/calendar'` grants full read/write access to the user's calendars, which is necessary for updating events.

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
      description: 'Event ID to update',
    },
    summary: {
      type: 'string',
      required: false,
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
      required: false,
      visibility: 'user-or-llm',
      description: 'Start date and time (RFC3339 format, e.g., 2025-06-03T10:00:00-08:00)',
    },
    endDateTime: {
      type: 'string',
      required: false,
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
      type: 'array',
      required: false,
      visibility: 'user-or-llm',
      description: 'Array of attendee email addresses (replaces all existing attendees)',
    },
    sendUpdates: {
      type: 'string',
      required: false,
      visibility: 'user-only',
      description: 'How to send updates to attendees: all, externalOnly, or none',
    },
  },
```

*   **`params: { ... }`**: This object defines all the input parameters that can be passed to this tool. Each parameter has a specific structure:
    *   **`type`**: The data type of the parameter (e.g., `'string'`, `'array'`).
    *   **`required`**: A boolean indicating if this parameter must always be provided.
    *   **`visibility`**: Controls who can provide or see this parameter.
        *   `'hidden'`: Typically for system-level parameters like access tokens, not exposed to the end-user or an LLM.
        *   `'user-only'`: Can be provided by the end-user, but an LLM might not be prompted to generate it directly.
        *   `'user-or-llm'`: Can be provided by either the end-user or an LLM, often for core event details.
    *   **`description`**: A human-readable description of the parameter's purpose and expected format.

    Let's look at some key parameters:
    *   **`accessToken`**: The OAuth token obtained from the `google-calendar` provider. It's `required` and `hidden` because it's sensitive and handled internally.
    *   **`calendarId`**: The ID of the calendar to update the event in. It's `optional` and defaults to `'primary'` (the user's main calendar) if not provided.
    *   **`eventId`**: **Crucial** and `required`, this is the unique identifier of the event to be updated.
    *   **`summary`, `description`, `location`**: Common event details, all `optional`.
    *   **`startDateTime`, `endDateTime`**: The start and end times of the event. They should be in [RFC3339 format](https://www.ietf.org/rfc/rfc3339.txt). Both are `optional` but are often updated together.
    *   **`timeZone`**: The time zone for the event, `optional`.
    *   **`attendees`**: An array (or comma-separated string) of email addresses. **Important**: It explicitly states this `replaces all existing attendees`.
    *   **`sendUpdates`**: An `optional` parameter to control how notifications are sent to attendees when an event is updated. Values like `'all'`, `'externalOnly'`, or `'none'` are typical.

---

```typescript
  request: {
    url: (params: GoogleCalendarUpdateParams) => {
      const calendarId = params.calendarId || 'primary'
      return `${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events/${encodeURIComponent(params.eventId)}`
    },
    method: 'GET',
    headers: (params: GoogleCalendarUpdateParams) => ({
      Authorization: `Bearer ${params.accessToken}`,
      'Content-Type': 'application/json',
    }),
  },
```

*   **`request: { ... }`**: This section describes the initial HTTP request that needs to be made. As explained in the simplified logic, this *first* request is a `GET` to fetch the existing event details before modification.
    *   **`url: (params: GoogleCalendarUpdateParams) => { ... }`**: A function that constructs the URL for the request. It takes the tool's parameters as input.
        *   **`const calendarId = params.calendarId || 'primary'`**: If `params.calendarId` is provided, use it; otherwise, default to `'primary'`.
        *   **``${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events/${encodeURIComponent(params.eventId)}``**: This uses template literals to build the URL.
            *   `CALENDAR_API_BASE`: The base API URL.
            *   `/calendars/${encodeURIComponent(calendarId)}`: Appends the calendar ID. `encodeURIComponent` is used to safely include the ID in a URL (e.g., if it contains special characters).
            *   `/events/${encodeURIComponent(params.eventId)}`: Appends the event ID, also URL-encoded.
    *   **`method: 'GET'`**: Specifies that this is an HTTP GET request, used to retrieve data.
    *   **`headers: (params: GoogleCalendarUpdateParams) => ({ ... })`**: A function that returns an object containing the HTTP headers for the request.
        *   **`Authorization: `Bearer ${params.accessToken}``**: This is the standard way to send an OAuth 2.0 access token. The token is prefixed with "Bearer ".
        *   **`'Content-Type': 'application/json'`**: Although this is a GET request and typically doesn't have a body, setting `Content-Type` to `application/json` is common practice for APIs that primarily deal with JSON.

---

```typescript
  transformResponse: async (response: Response, params) => {
    const existingEvent = await response.json()

    // Start with the complete existing event to preserve all fields
    const updatedEvent = { ...existingEvent }
```

*   **`transformResponse: async (response: Response, params) => { ... }`**: This is the main function responsible for processing the API responses and orchestrating the actual update.
    *   **`async`**: Indicates that this function performs asynchronous operations (like `await`).
    *   **`(response: Response, params)`**: It takes two arguments:
        *   `response`: The `Response` object from the *initial `GET` request*.
        *   `params`: The input parameters originally provided to the tool.
    *   **`const existingEvent = await response.json()`**: Reads the body of the initial `GET` response and parses it as JSON. This `existingEvent` object now contains all current details of the event from Google Calendar.
    *   **`const updatedEvent = { ...existingEvent }`**: Creates a *shallow copy* of the `existingEvent` object using the spread syntax (`...`). This is crucial because we want to modify `updatedEvent` without altering `existingEvent` directly, and we also want to retain all original fields not explicitly updated by the `params`.

---

```typescript
    // Apply updates only for fields that are provided and not empty
    if (
      params?.summary !== undefined &&
      params?.summary !== null &&
      params?.summary.trim() !== ''
    ) {
      updatedEvent.summary = params.summary
    }
    // ... similar blocks for description, location
```

*   **`if (params?.summary !== undefined && params?.summary !== null && params?.summary.trim() !== '') { ... }`**: This block (and similar ones for `description` and `location`) checks if a new value for `summary` (or other fields) was provided in the `params`.
    *   **`params?.summary`**: Uses optional chaining (`?`) because `params` itself might be `null` or `undefined` in some contexts (though unlikely here), and `summary` is an optional parameter.
    *   **`!== undefined && params?.summary !== null`**: Ensures the parameter was actually passed (not just missing) and is not `null`.
    *   **`params?.summary.trim() !== ''`**: Further ensures that if a string was provided, it's not just an empty string or whitespace.
    *   **`updatedEvent.summary = params.summary`**: If all conditions pass, the `summary` field in our `updatedEvent` object is set to the new value from `params`.

---

```typescript
    // Only update times if both start and end are provided (Google Calendar requires both)
    const hasStartTime =
      params?.startDateTime !== undefined &&
      params?.startDateTime !== null &&
      params?.startDateTime.trim() !== ''
    const hasEndTime =
      params?.endDateTime !== undefined &&
      params?.endDateTime !== null &&
      params?.endDateTime.trim() !== ''

    if (hasStartTime && hasEndTime) {
      updatedEvent.start = {
        dateTime: params.startDateTime,
      }
      updatedEvent.end = {
        dateTime: params.endDateTime,
      }
      if (params?.timeZone) {
        updatedEvent.start.timeZone = params.timeZone
        updatedEvent.end.timeZone = params.timeZone
      }
    }
```

*   **`const hasStartTime = ...` / `const hasEndTime = ...`**: These lines define boolean flags to check if `startDateTime` and `endDateTime` were provided and are not empty, using the same robust checking logic as for `summary`.
*   **`if (hasStartTime && hasEndTime) { ... }`**: This conditional block ensures that event start and end times are *only* updated if *both* `startDateTime` and `endDateTime` are present. Google Calendar API often requires both to be specified when updating times.
    *   **`updatedEvent.start = { dateTime: params.startDateTime, }`**: The `start` property of the event is an object in Google Calendar, containing a `dateTime` field.
    *   **`updatedEvent.end = { dateTime: params.endDateTime, }`**: Similarly for the `end` property.
    *   **`if (params?.timeZone) { ... }`**: If a `timeZone` is also provided in the `params`, it's applied to both the `start` and `end` time objects.

---

```typescript
    // Handle attendees update - this replaces all existing attendees
    if (params?.attendees !== undefined && params?.attendees !== null) {
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

      // Replace all attendees with the new list
      if (attendeeList.length > 0) {
        updatedEvent.attendees = attendeeList.map((email: string) => ({
          email,
          responseStatus: 'needsAction',
        }))
      } else {
        // If empty attendee list is provided, remove all attendees
        updatedEvent.attendees = []
      }
    }
```

*   **`if (params?.attendees !== undefined && params?.attendees !== null) { ... }`**: This block handles updates to attendees. It's triggered if `params.attendees` was explicitly provided (even if it's an empty array or string).
    *   **`let attendeeList: string[] = []`**: Initializes an empty array to hold cleaned-up attendee email addresses.
    *   **`if (params.attendees) { ... }`**: Checks if `params.attendees` is truthy (not `null`, `undefined`, or empty string/array initially).
        *   **`const attendees = params.attendees as string | string[]`**: Casts `params.attendees` to its expected union type.
        *   **`if (Array.isArray(attendees))`**: If `attendees` is already an array:
            *   **`attendeeList = attendees.filter((email: string) => email && email.trim().length > 0)`**: Filters out any `null`, `undefined`, or empty string emails from the array.
        *   **`else if (typeof attendees === 'string' && attendees.trim().length > 0)`**: If `attendees` is a non-empty string:
            *   **`attendeeList = attendees.split(',').map(...).filter(...)`**: Splits the comma-separated string into an array, `trim`s whitespace from each email, and then filters out any empty results. This handles cases where `attendees` might be `user1@example.com, user2@example.com`.
    *   **`if (attendeeList.length > 0) { ... }`**: If, after processing, `attendeeList` contains actual email addresses:
        *   **`updatedEvent.attendees = attendeeList.map((email: string) => ({ email, responseStatus: 'needsAction', }))`**: Maps each email string into an object format expected by Google Calendar (`{ email: "...", responseStatus: "needsAction" }`). `'needsAction'` is a common default status for newly added attendees, indicating they haven't responded yet.
    *   **`else { updatedEvent.attendees = [] }`**: If `attendeeList` is empty (meaning an empty array or string was provided for attendees, or it resulted in no valid emails), the `attendees` field in `updatedEvent` is set to an empty array. This effectively *removes all attendees* from the event.

---

```typescript
    // Remove read-only fields that shouldn't be included in updates
    const readOnlyFields = [
      'id',
      'etag',
      'kind',
      'created',
      'updated',
      'htmlLink',
      'iCalUID',
      'sequence',
      'creator',
      'organizer',
    ]
    readOnlyFields.forEach((field) => {
      delete updatedEvent[field]
    })
```

*   **`const readOnlyFields = [ ... ]`**: Defines an array of field names that are managed by Google Calendar and cannot be updated by the client (they are "read-only").
*   **`readOnlyFields.forEach((field) => { delete updatedEvent[field] })`**: Loops through each field in `readOnlyFields` and removes that property from the `updatedEvent` object. This prevents sending these fields back to the API in the `PUT` request, which could otherwise cause an error.

---

```typescript
    // Construct PUT URL with query parameters
    const calendarId = params?.calendarId || 'primary'
    const queryParams = new URLSearchParams()
    if (params?.sendUpdates !== undefined) {
      queryParams.append('sendUpdates', params.sendUpdates)
    }

    const queryString = queryParams.toString()
    const putUrl = `${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events/${encodeURIComponent(params?.eventId || '')}${queryString ? `?${queryString}` : ''}`
```

*   **`const calendarId = params?.calendarId || 'primary'`**: Re-determines the `calendarId`, defaulting to 'primary' if not provided.
*   **`const queryParams = new URLSearchParams()`**: Creates a new `URLSearchParams` object, a utility for building URL query strings.
*   **`if (params?.sendUpdates !== undefined) { queryParams.append('sendUpdates', params.sendUpdates) }`**: If the `sendUpdates` parameter was provided, it's added as a query parameter to the `queryParams` object.
*   **`const queryString = queryParams.toString()`**: Converts the `queryParams` object into a URL-encoded query string (e.g., `sendUpdates=all`).
*   **`const putUrl = `${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events/${encodeURIComponent(params?.eventId || '')}${queryString ? `?${queryString}` : ''}``**: Constructs the final URL for the `PUT` request. It's similar to the `GET` URL but conditionally appends the `queryString` if it exists, prefixed with `?`. `params?.eventId || ''` is used for robustness, though `eventId` is required.

---

```typescript
    // Send PUT request to update the event
    const putResponse = await fetch(putUrl, {
      method: 'PUT',
      headers: {
        Authorization: `Bearer ${params?.accessToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(updatedEvent),
    })

    if (!putResponse.ok) {
      const errorData = await putResponse.json()
      throw new Error(errorData.error?.message || 'Failed to update calendar event')
    }

    const data = await putResponse.json()
```

*   **`const putResponse = await fetch(putUrl, { ... })`**: Makes the actual HTTP `PUT` request to update the event.
    *   **`method: 'PUT'`**: Specifies the HTTP PUT method, used for updating existing resources.
    *   **`headers: { ... }`**: Similar headers as the `GET` request, including `Authorization` and `Content-Type`.
    *   **`body: JSON.stringify(updatedEvent)`**: The `updatedEvent` object (which now contains the merged updates and no read-only fields) is converted into a JSON string and sent as the request body.
*   **`if (!putResponse.ok) { ... }`**: Checks if the `PUT` request was successful (HTTP status code 2xx).
    *   **`const errorData = await putResponse.json()`**: If not successful, it tries to parse the error response body as JSON.
    *   **`throw new Error(errorData.error?.message || 'Failed to update calendar event')`**: Throws an `Error` object, using the error message from the API if available, or a generic message.
*   **`const data = await putResponse.json()`**: If the `PUT` request was successful, this parses the JSON response body into the `data` variable. This `data` object now contains the fully updated event details from Google Calendar.

---

```typescript
    return {
      success: true,
      output: {
        content: `Event "${data.summary}" updated successfully`,
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

*   **`return { ... }`**: Returns the final result of the tool's execution.
    *   **`success: true`**: Indicates that the operation was successful.
    *   **`output: { ... }`**: An object containing the actual output.
        *   **`content: `Event "${data.summary}" updated successfully``**: A human-friendly confirmation message, using the `summary` from the updated event data.
        *   **`metadata: { ... }`**: An object containing key details of the updated event, extracted from the `data` received from Google Calendar. This metadata can be used by the calling application for further processing or display.

---

```typescript
  outputs: {
    content: { type: 'string', description: 'Event update confirmation message' },
    metadata: {
      type: 'json',
      description: 'Updated event metadata including ID, status, and details',
    },
  },
}
```

*   **`outputs: { ... }`**: This section describes the expected structure and types of the output that this tool will produce, mirroring the `return` statement in `transformResponse`. This serves as documentation for consumers of this tool.
    *   **`content: { ... }`**: Describes the `content` field of the output.
        *   **`type: 'string'`**: Indicates it's a string.
        *   **`description: 'Event update confirmation message'`**: Explains its purpose.
    *   **`metadata: { ... }`**: Describes the `metadata` field of the output.
        *   **`type: 'json'`**: Indicates it's a JSON object.
        *   **`description: 'Updated event metadata including ID, status, and details'`**: Explains its purpose.

This comprehensive configuration allows an external system to understand and utilize the Google Calendar Update Event tool effectively, from input parameters and authentication to the intricate details of making and processing API calls.