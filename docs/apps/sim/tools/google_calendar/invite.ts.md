This TypeScript code defines a "tool" designed to invite attendees to an *existing* Google Calendar event. It's structured as a `ToolConfig` object, which is a common pattern in systems that integrate various external services or APIs as modular capabilities.

This tool handles the entire workflow:
1.  **Retrieving** the existing event details from Google Calendar.
2.  **Calculating** the final list of attendees, either by replacing all existing attendees or adding new ones, while avoiding duplicates.
3.  **Updating** the event on Google Calendar with the modified attendee list.
4.  **Providing** a clear, human-readable confirmation message and detailed event metadata.

Let's break down each part.

---

## Purpose of this File

This file defines a single, specific **tool**: `google_calendar_invite`.
Its primary purpose is to enable an application (potentially an AI agent, a backend service, or a user interface) to programmatically **add or replace attendees** for an existing event in a Google Calendar.

It abstracts away the complexities of interacting directly with the Google Calendar API, including OAuth authentication, constructing API requests, handling various parameter inputs (like attendee lists and update preferences), managing read-only fields, and processing the API responses. Essentially, it provides a standardized, easy-to-use interface for a very specific Google Calendar operation.

---

## Detailed Explanation

### Imports

```typescript
import {
  CALENDAR_API_BASE,
  type GoogleCalendarInviteParams,
  type GoogleCalendarInviteResponse,
} from '@/tools/google_calendar/types'
import type { ToolConfig } from '@/tools/types'
```

*   `CALENDAR_API_BASE`: This imports a constant string, likely the base URL for the Google Calendar API (e.g., `https://www.googleapis.com/calendar/v3`). This helps keep API endpoints consistent.
*   `type GoogleCalendarInviteParams`: This imports a TypeScript type (interface) that defines the expected structure of the input parameters for this tool. It specifies what information needs to be provided to invite attendees (e.g., event ID, attendee emails).
*   `type GoogleCalendarInviteResponse`: This imports a TypeScript type that defines the expected structure of the output this tool will produce after successfully inviting attendees.
*   `type ToolConfig`: This imports a generic TypeScript type that defines the overall structure for *any* tool in this system. It acts as a blueprint for how tools are configured, including their ID, name, parameters, request details, and response transformation logic. The `<GoogleCalendarInviteParams, GoogleCalendarInviteResponse>` part specifies that *this particular tool* uses `GoogleCalendarInviteParams` for its inputs and `GoogleCalendarInviteResponse` for its outputs.

### Tool Definition: `inviteTool`

```typescript
export const inviteTool: ToolConfig<GoogleCalendarInviteParams, GoogleCalendarInviteResponse> = {
  // ... configuration for the tool ...
}
```

This line exports a constant variable named `inviteTool`. Its type is `ToolConfig` configured for `GoogleCalendarInviteParams` and `GoogleCalendarInviteResponse`, meaning it adheres to the established structure for tools in this system and specifically handles the invitation process.

Let's break down the properties of this `inviteTool` object:

#### `id`, `name`, `description`, `version`

```typescript
  id: 'google_calendar_invite',
  name: 'Google Calendar Invite Attendees',
  description: 'Invite attendees to an existing Google Calendar event',
  version: '1.0.0',
```

These are basic metadata fields that identify and describe the tool:
*   `id`: A unique programmatic identifier for this tool.
*   `name`: A human-readable name for the tool.
*   `description`: A brief explanation of what the tool does.
*   `version`: The version number of this tool's definition.

#### `oauth` Configuration

```typescript
  oauth: {
    required: true,
    provider: 'google-calendar',
    additionalScopes: ['https://www.googleapis.com/auth/calendar'],
  },
```

This section specifies the OAuth (Open Authorization) requirements for this tool:
*   `required: true`: Indicates that user authorization via OAuth is necessary to use this tool.
*   `provider: 'google-calendar'`: Specifies that the authorization should be handled by the 'google-calendar' OAuth provider, meaning it will use Google's authentication system.
*   `additionalScopes: ['https://www.googleapis.com/auth/calendar']`: This is a crucial security setting. It requests specific permissions from the user. The `https://www.googleapis.com/auth/calendar` scope grants full read/write access to the user's calendars, which is necessary to modify events and invite attendees.

#### `params` Definition

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
      description: 'Event ID to invite attendees to',
    },
    attendees: {
      type: 'array',
      required: true,
      visibility: 'user-or-llm',
      description: 'Array of attendee email addresses to invite',
    },
    sendUpdates: {
      type: 'string',
      required: false,
      visibility: 'user-only',
      description: 'How to send updates to attendees: all, externalOnly, or none',
    },
    replaceExisting: {
      type: 'boolean',
      required: false,
      visibility: 'user-only',
      description: 'Whether to replace existing attendees or add to them (defaults to false)',
    },
  },
```

This object defines all the input parameters this tool expects. Each parameter has:
*   `type`: The expected data type (e.g., `'string'`, `'array'`, `'boolean'`).
*   `required`: Whether the parameter must be provided (`true`) or is optional (`false`).
*   `visibility`: Controls who can see or provide this parameter.
    *   `'hidden'`: Typically provided by the system, not directly by the user or an LLM (Large Language Model).
    *   `'user-only'`: Meant for direct user input, or in some systems, can be inferred.
    *   `'user-or-llm'`: Can be provided by either a user or an LLM interacting with the tool.
*   `description`: A brief explanation of what the parameter represents.

Let's look at specific parameters:
*   `accessToken`: This is the OAuth token obtained from the Google Calendar provider. It's `required` and `hidden` because the system handles its injection after successful authentication.
*   `calendarId`: The ID of the calendar where the event resides. It's `optional` and defaults to `'primary'` (the user's default calendar) if not provided.
*   `eventId`: The unique identifier of the specific event to which attendees will be invited. This is `required`.
*   `attendees`: This is a `required` array of email addresses. This is the core input for who to invite.
*   `sendUpdates`: An `optional` string that controls notification behavior. Possible values are usually `'all'` (send updates to all attendees), `'externalOnly'` (only to attendees outside the organizer's domain), or `'none'` (no email notifications).
*   `replaceExisting`: An `optional` boolean. If `true`, any new attendees provided will completely replace the current attendee list of the event. If `false` (the default), new attendees will be added to the existing list, avoiding duplicates.

#### `request` Definition (Initial GET request)

```typescript
  request: {
    url: (params: GoogleCalendarInviteParams) => {
      const calendarId = params.calendarId || 'primary'
      return `${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events/${encodeURIComponent(params.eventId)}`
    },
    method: 'GET',
    headers: (params: GoogleCalendarInviteParams) => ({
      Authorization: `Bearer ${params.accessToken}`,
      'Content-Type': 'application/json',
    }),
  },
```

This section defines the initial HTTP request that the tool will make. Crucially, this is a **`GET` request** to fetch the *current* state of the event. This is necessary because the Google Calendar API's `PUT` (update) method requires sending the *entire* event object, not just the fields you want to change. So, we first retrieve the existing event, modify it, and then send the complete, updated object back.

*   `url`: A function that dynamically constructs the URL for the `GET` request.
    *   `const calendarId = params.calendarId || 'primary'`: If `params.calendarId` is not provided, it defaults to `'primary'`.
    *   `return `${CALENDAR_API_BASE}/calendars/${encodeURIComponent(calendarId)}/events/${encodeURIComponent(params.eventId)}``: This builds the full URL. `encodeURIComponent` is used to safely encode parts of the URL (like `calendarId` or `eventId`) that might contain special characters.
*   `method: 'GET'`: Specifies that this is a GET request to retrieve data.
*   `headers`: A function that dynamically constructs the HTTP headers for the request.
    *   `Authorization: `Bearer ${params.accessToken}``: This is the standard way to send an OAuth access token, allowing the API to authenticate the request on behalf of the user.
    *   `'Content-Type': 'application/json'`: Specifies that the request body (though empty for a GET) expects and sends JSON.

#### `transformResponse` Logic (The Core)

```typescript
  transformResponse: async (response: Response, params) => {
    // ... complex logic ...
  },
```

This is the most complex part of the tool. It's an `async` function that takes the `Response` from the *initial `GET` request* (defined above) and the original `params` provided to the tool. Its responsibility is to:
1.  Parse the existing event data.
2.  Determine the final list of attendees based on `params.attendees` and `params.replaceExisting`.
3.  Construct a new `PUT` request to update the event with the new attendees.
4.  Handle the response from the `PUT` request.
5.  Format the final output for the tool.

Let's break it down step-by-step:

```typescript
    const existingEvent = await response.json()
```

*   `const existingEvent = await response.json()`: This line parses the JSON body of the `Response` object from the initial `GET` request, converting it into a JavaScript object (`existingEvent`) that represents the current state of the Google Calendar event.

```typescript
    // Validate required fields exist
    if (!existingEvent.start || !existingEvent.end || !existingEvent.summary) {
      throw new Error('Existing event is missing required fields (start, end, or summary)')
    }
```

*   This block performs a basic validation. The Google Calendar API's `PUT` method requires the `start`, `end`, and `summary` fields to be present in the event object for an update. If any of these are missing from the `existingEvent`, it throws an error, indicating that the retrieved event is malformed or incomplete, preventing a failed update.

```typescript
    // Process new attendees - handle both string and array formats
    let newAttendeeList: string[] = []

    if (params?.attendees) {
      if (Array.isArray(params.attendees)) {
        // Already an array from block processing
        newAttendeeList = params.attendees.filter(
          (email: string) => email && email.trim().length > 0
        )
      } else if (
        typeof (params.attendees as any) === 'string' &&
        (params.attendees as any).trim().length > 0
      ) {
        // Fallback: process comma-separated string if block didn't convert it
        newAttendeeList = (params.attendees as any)
          .split(',')
          .map((email: string) => email.trim())
          .filter((email: string) => email.length > 0)
      }
    }
```

*   This section processes the `attendees` input from the tool's parameters. It's designed to be flexible:
    *   `let newAttendeeList: string[] = []`: Initializes an empty array to store the processed email addresses.
    *   `if (params?.attendees)`: Checks if the `attendees` parameter was provided.
    *   `if (Array.isArray(params.attendees))`: If `attendees` is already an array (which is the preferred format), it filters out any empty or whitespace-only email strings.
    *   `else if (typeof (params.attendees as any) === 'string' ...)`: This is a fallback. If `attendees` was provided as a string (e.g., "email1@example.com, email2@example.com"), it:
        *   `.split(',')`: Splits the string by commas into an array.
        *   `.map((email: string) => email.trim())`: Trims whitespace from each email.
        *   `.filter((email: string) => email.length > 0)`: Removes any resulting empty strings.
    *   The `(params.attendees as any)` type assertion is used here to temporarily bypass TypeScript's strict type checking, allowing the code to treat `params.attendees` as a string even though its declared type might be an array or a union type. This is often done when handling potentially inconsistent input formats.

```typescript
    // Calculate final attendees list
    const existingAttendees = existingEvent.attendees || []
    let finalAttendees: Array<any> = []

    // Handle replaceExisting properly - check for both boolean true and string "true"
    const shouldReplace =
      params?.replaceExisting === true || (params?.replaceExisting as any) === 'true'

    if (shouldReplace) {
      // Replace all attendees with just the new ones
      finalAttendees = newAttendeeList.map((email: string) => ({
        email,
        responseStatus: 'needsAction',
      }))
    } else {
      // Add to existing attendees (preserve all existing ones)

      // Start with ALL existing attendees - preserve them completely
      finalAttendees = [...existingAttendees]

      // Get set of existing emails for duplicate checking (case-insensitive)
      const existingEmails = new Set(
        existingAttendees.map((attendee: any) => attendee.email?.toLowerCase() || '')
      )

      // Add only new attendees that don't already exist
      for (const newEmail of newAttendeeList) {
        const emailLower = newEmail.toLowerCase()
        if (!existingEmails.has(emailLower)) {
          finalAttendees.push({
            email: newEmail,
            responseStatus: 'needsAction',
          })
        }
      }
    }
```

This is the core logic for determining the final list of attendees:
*   `const existingAttendees = existingEvent.attendees || []`: Safely gets the `attendees` array from the `existingEvent`. If it doesn't exist, it defaults to an empty array.
*   `let finalAttendees: Array<any> = []`: Initializes an array to hold the final list of attendees.
*   `const shouldReplace = params?.replaceExisting === true || (params?.replaceExisting as any) === 'true'`: Determines if existing attendees should be replaced. It checks for both a boolean `true` and a string `"true"` to be robust against different input origins (e.g., a form field sending "true" as a string).
*   **`if (shouldReplace)`:** If `replaceExisting` is true:
    *   `finalAttendees = newAttendeeList.map(...)`: The `finalAttendees` list is simply the `newAttendeeList`, transformed into the Google Calendar API's required attendee object format `{ email: '...', responseStatus: 'needsAction' }`. `needsAction` indicates the attendee has been invited but hasn't responded yet.
*   **`else` (add to existing):** If `replaceExisting` is false (or not provided):
    *   `finalAttendees = [...existingAttendees]`: Starts by including all existing attendees to preserve them.
    *   `const existingEmails = new Set(...)`: Creates a `Set` of existing attendee email addresses (converted to lowercase). A `Set` is highly efficient for checking if an item already exists. This prevents inviting someone already on the list.
    *   `for (const newEmail of newAttendeeList)`: Iterates through each email in the `newAttendeeList`.
    *   `const emailLower = newEmail.toLowerCase()`: Converts the new email to lowercase for case-insensitive comparison.
    *   `if (!existingEmails.has(emailLower))`: Checks if this new email is *not* already in the `existingEmails` set.
    *   `finalAttendees.push(...)`: If it's a truly new attendee, it's added to `finalAttendees` in the required Google Calendar object format.

```typescript
    // Use the complete existing event object and only modify the attendees field
    // This is crucial because the Google Calendar API update method "does not support patch semantics
    // and always updates the entire event resource" according to the documentation
    const updatedEvent = {
      ...existingEvent, // Start with the complete existing event to preserve all fields
      attendees: finalAttendees, // Only modify the attendees field
    }
```

*   `const updatedEvent = { ...existingEvent, attendees: finalAttendees }`: This is a critical step. It creates a *new* event object for the update. It uses the spread syntax (`...existingEvent`) to copy all properties from the `existingEvent` object and then explicitly overwrites the `attendees` property with the `finalAttendees` list calculated above. This is necessary because the Google Calendar API's `PUT` method does not support partial updates (patches); it requires the entire event resource to be sent.

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

*   `readOnlyFields.forEach((field) => { delete updatedEvent[field] })`: This block defines a list of fields that are typically read-only or managed by the Google Calendar API itself (e.g., event ID, creation timestamp, creator details). Attempting to send these fields in an update request can cause errors. Therefore, this code iterates through `readOnlyFields` and `deletes` each corresponding property from the `updatedEvent` object before sending it.

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

*   This section constructs the URL for the *second* HTTP request – the `PUT` request to update the event.
    *   `const calendarId = params?.calendarId || 'primary'`: Again, handles the default `calendarId`.
    *   `const queryParams = new URLSearchParams()`: Creates an instance of `URLSearchParams` to easily manage URL query parameters.
    *   `if (params?.sendUpdates !== undefined) { queryParams.append('sendUpdates', params.sendUpdates) }`: If the `sendUpdates` parameter was provided, it's added as a query parameter (e.g., `?sendUpdates=all`).
    *   `const queryString = queryParams.toString()`: Converts the `URLSearchParams` object into a URL-encoded string (e.g., `sendUpdates=all`).
    *   `const putUrl = ...`: Constructs the final `PUT` URL, appending the `queryString` if it's not empty.

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
```

*   `const putResponse = await fetch(...)`: This is where the actual update happens. It makes a `PUT` request to the `putUrl`.
    *   `method: 'PUT'`: Specifies the HTTP method for updating a resource.
    *   `headers`: Includes the `Authorization` token and sets `Content-Type` to `application/json` since we're sending a JSON body.
    *   `body: JSON.stringify(updatedEvent)`: The `updatedEvent` object (which includes all existing fields and the modified attendees list, with read-only fields removed) is converted to a JSON string and sent as the request body.

```typescript
    // Handle the PUT response
    if (!putResponse.ok) {
      const errorData = await putResponse.json()
      throw new Error(errorData.error?.message || 'Failed to invite attendees to calendar event')
    }
```

*   This block handles potential errors from the `PUT` request.
    *   `if (!putResponse.ok)`: Checks if the HTTP response status code indicates success (e.g., 200-299). If not, it's an error.
    *   `const errorData = await putResponse.json()`: Attempts to parse an error message from the response body.
    *   `throw new Error(...)`: Throws a new `Error` with a message derived from the API error or a generic fallback.

```typescript
    const data = await putResponse.json()
    const totalAttendees = data.attendees?.length || 0
```

*   `const data = await putResponse.json()`: If the `PUT` request was successful, this parses the JSON response from the Google Calendar API, which contains the updated event details.
*   `const totalAttendees = data.attendees?.length || 0`: Extracts the total number of attendees from the updated event data.

```typescript
    // Calculate how many new attendees were actually added
    let newAttendeesAdded = 0

    if (shouldReplace) {
      newAttendeesAdded = newAttendeeList.length
    } else {
      // Count how many of the new emails weren't already in the existing list
      const existingEmails = new Set(
        existingAttendees.map((attendee: any) => attendee.email?.toLowerCase() || '')
      )
      newAttendeesAdded = newAttendeeList.filter(
        (email) => !existingEmails.has(email.toLowerCase())
      ).length
    }
```

*   This section calculates how many *new* attendees were actually added as a result of the operation, which is useful for the final user message.
    *   `if (shouldReplace)`: If attendees were replaced, all attendees from `newAttendeeList` are considered "newly added" in the context of the *new event state*.
    *   `else`: If adding to existing attendees, it re-uses the `existingEmails` set (or reconstructs it) to count only those in `newAttendeeList` that were *not* present in the original event.

```typescript
    // Improved messaging about email delivery
    let baseMessage: string
    if (shouldReplace) {
      baseMessage = `Successfully updated event "${data.summary}" with ${totalAttendees} attendee${totalAttendees !== 1 ? 's' : ''}`
    } else {
      if (newAttendeesAdded > 0) {
        baseMessage = `Successfully added ${newAttendeesAdded} new attendee${newAttendeesAdded !== 1 ? 's' : ''} to event "${data.summary}" (total: ${totalAttendees})`
      } else {
        baseMessage = `No new attendees added to event "${data.summary}" - all specified attendees were already invited (total: ${totalAttendees})`
      }
    }

    const emailNote =
      params?.sendUpdates !== 'none'
        ? ` Email invitations are being sent asynchronously - delivery may take a few minutes and depends on recipients' Google Calendar settings.`
        : ` No email notifications will be sent as requested.`

    const content = baseMessage + emailNote
```

*   This block constructs the final human-readable `content` message for the tool's output.
    *   `baseMessage`: Varies based on whether attendees were replaced or added, and how many new ones were actually integrated. It uses conditional pluralization (`attendee${totalAttendees !== 1 ? 's' : ''}`).
    *   `emailNote`: Provides important context about email delivery, informing the user if emails are being sent (and their asynchronous nature) or if they were explicitly suppressed (`sendUpdates: 'none'`).
    *   `const content = baseMessage + emailNote`: Concatenates the base message and the email notification note.

```typescript
    return {
      success: true,
      output: {
        content,
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

*   Finally, the `transformResponse` function returns the structured output of the tool.
    *   `success: true`: Indicates the operation completed successfully.
    *   `output`: An object containing the primary results:
        *   `content`: The human-readable message generated above.
        *   `metadata`: An object containing key details from the *updated* Google Calendar event, useful for programmatic follow-up or displaying to the user. This includes the event ID, link, status, summary, and the final list of attendees, among others.

#### `outputs` Definition

```typescript
  outputs: {
    content: {
      type: 'string',
      description: 'Attendee invitation confirmation message with email delivery status',
    },
    metadata: {
      type: 'json',
      description: 'Updated event metadata including attendee list and details',
    },
  },
}
```

This section formally defines the structure and types of the output that this tool will produce. This is useful for systems that need to understand and process the tool's results.
*   `content`: A `string` containing the confirmation message.
*   `metadata`: A `json` object containing the detailed information about the updated event.

---

In summary, this `inviteTool` is a robust and well-designed component for managing event attendees in Google Calendar. It encapsulates complex API interactions, handles various input scenarios, ensures data integrity, and provides clear, actionable feedback.