This TypeScript file is a comprehensive collection of interfaces and types designed to define the data structures for interacting with the **Google Calendar API** within a specific application context, likely a "tool" or "service" that wraps this API.

It acts as a **contract** for:
1.  **Input Parameters:** What information is required when making different requests to the Google Calendar API (e.g., creating, listing, updating events).
2.  **Output Responses:** What data to expect back from the Google Calendar API calls, both in a simplified "tool" format and in a more direct API format.
3.  **Data Models:** The structure of Google Calendar events and attendees.

By defining these types, the file ensures **type safety**, provides **clear documentation** for developers using the API wrapper, and helps **prevent common errors** by catching incorrect data structures at compile time.

Let's break down each part of the code:

---

### File Structure & Core Concepts

```typescript
import type { ToolResponse } from '@/tools/types'
```
*   **`import type { ToolResponse } from '@/tools/types'`**: This line imports a type named `ToolResponse` from another file located at `@/tools/types`.
    *   **`import type`**: This is a TypeScript-specific import syntax. It signals that `ToolResponse` is *only* used as a type and will be completely removed during compilation to JavaScript. This helps prevent unnecessary code being bundled if the imported type isn't used as a runtime value.
    *   **`ToolResponse`**: This likely represents a generic base structure for responses from any "tool" in the system, providing common properties like `status` or `error` messages. The Google Calendar specific responses will extend this base.

---

### API Base URL Constant

```typescript
export const CALENDAR_API_BASE = 'https://www.googleapis.com/calendar/v3'
```
*   **`export const CALENDAR_API_BASE = 'https://www.googleapis.com/calendar/v3'`**: This declares and exports a constant variable.
    *   **`export`**: Makes this constant available for other files to import and use.
    *   **`const`**: Declares a constant, meaning its value cannot be reassigned after it's initially set.
    *   **`CALENDAR_API_BASE`**: The name of the constant, clearly indicating its purpose.
    *   **`'https://www.googleapis.com/calendar/v3'`**: The string value, which is the base URL for version 3 of the Google Calendar API. This is a common practice to centralize API endpoints for easier management and updates.

---

### Shared Data Models

#### `CalendarAttendee` Interface

```typescript
export interface CalendarAttendee {
  id?: string
  email: string
  displayName?: string
  organizer?: boolean
  self?: boolean
  resource?: boolean
  optional?: boolean
  responseStatus: string
  comment?: string
  additionalGuests?: number
}
```
*   **`export interface CalendarAttendee`**: Defines a TypeScript interface named `CalendarAttendee`. This interface describes the properties of an individual attending a calendar event, matching the structure used by the Google Calendar API.
    *   **`export`**: Makes this interface available for other files.
    *   **`interface`**: A TypeScript construct that defines the shape of an object.
    *   **`id?: string`**: An optional (`?`) property for the attendee's unique ID. It's a string.
    *   **`email: string`**: A required property for the attendee's email address. It's a string.
    *   **`displayName?: string`**: An optional property for the attendee's display name.
    *   **`organizer?: boolean`**: An optional boolean indicating if this attendee is the organizer of the event.
    *   **`self?: boolean`**: An optional boolean indicating if this attendee is the user making the API request.
    *   **`resource?: boolean`**: An optional boolean indicating if this attendee is a resource (e.g., a meeting room) rather than a person.
    *   **`optional?: boolean`**: An optional boolean indicating if the attendee's presence is optional.
    *   **`responseStatus: string`**: A required property indicating the attendee's response status (e.g., "accepted", "declined", "needsAction").
    *   **`comment?: string`**: An optional property for a comment left by the attendee.
    *   **`additionalGuests?: number`**: An optional property for the number of additional guests the attendee is bringing.

---

### Base Parameters for Google Calendar Operations

```typescript
interface BaseGoogleCalendarParams {
  accessToken: string
  calendarId?: string // defaults to 'primary' if not provided
}
```
*   **`interface BaseGoogleCalendarParams`**: Defines a base interface for parameters common to *all* Google Calendar API requests made by this tool.
    *   **`accessToken: string`**: A required property. This is the OAuth 2.0 access token needed to authenticate the request with Google's API.
    *   **`calendarId?: string`**: An optional property. This specifies the ID of the calendar to operate on. If not provided, it typically defaults to the user's primary calendar (`'primary'`).

---

### Operation-Specific Parameter Interfaces

These interfaces define the specific data required for different Google Calendar operations. All of them extend `BaseGoogleCalendarParams` to inherit `accessToken` and `calendarId`.

#### `GoogleCalendarCreateParams` (Create an Event)

```typescript
export interface GoogleCalendarCreateParams extends BaseGoogleCalendarParams {
  summary: string
  description?: string
  location?: string
  startDateTime: string
  endDateTime: string
  timeZone?: string
  attendees?: string[] // Array of email addresses
  sendUpdates?: 'all' | 'externalOnly' | 'none'
}
```
*   **`export interface GoogleCalendarCreateParams extends BaseGoogleCalendarParams`**: Defines parameters for creating a new calendar event. It inherits properties from `BaseGoogleCalendarParams`.
    *   **`summary: string`**: Required. The title or short description of the event.
    *   **`description?: string`**: Optional. A longer description of the event.
    *   **`location?: string`**: Optional. The physical location of the event.
    *   **`startDateTime: string`**: Required. The start time of the event, typically in RFC3339 format (e.g., `2023-10-27T10:00:00-07:00`).
    *   **`endDateTime: string`**: Required. The end time of the event, also in RFC3339 format.
    *   **`timeZone?: string`**: Optional. The time zone of the event (e.g., `America/Los_Angeles`).
    *   **`attendees?: string[]`**: Optional. An array of email addresses of people to invite to the event.
    *   **`sendUpdates?: 'all' | 'externalOnly' | 'none'`**: Optional. Controls who receives email updates about the event:
        *   `'all'`: All attendees.
        *   `'externalOnly'`: Only attendees not within the primary domain of the organizer.
        *   `'none'`: No attendees receive updates.

#### `GoogleCalendarListParams` (List Events)

```typescript
export interface GoogleCalendarListParams extends BaseGoogleCalendarParams {
  timeMin?: string // RFC3339 timestamp
  timeMax?: string // RFC3339 timestamp
  maxResults?: number
  singleEvents?: boolean
  orderBy?: 'startTime' | 'updated'
  showDeleted?: boolean
}
```
*   **`export interface GoogleCalendarListParams extends BaseGoogleCalendarParams`**: Defines parameters for listing calendar events.
    *   **`timeMin?: string`**: Optional. An RFC3339 timestamp specifying the earliest (inclusive) event start time to retrieve.
    *   **`timeMax?: string`**: Optional. An RFC3339 timestamp specifying the latest (exclusive) event start time to retrieve.
    *   **`maxResults?: number`**: Optional. The maximum number of events to return.
    *   **`singleEvents?: boolean`**: Optional. If `true`, expands recurring events into individual instances. Defaults to `false`.
    *   **`orderBy?: 'startTime' | 'updated'`**: Optional. The order of the events returned:
        *   `'startTime'`: Order by event start time.
        *   `'updated'`: Order by event last modification time.
    *   **`showDeleted?: boolean`**: Optional. If `true`, includes deleted events in the response. Defaults to `false`.

#### `GoogleCalendarGetParams` (Get a Single Event)

```typescript
export interface GoogleCalendarGetParams extends BaseGoogleCalendarParams {
  eventId: string
}
```
*   **`export interface GoogleCalendarGetParams extends BaseGoogleCalendarParams`**: Defines parameters for retrieving details of a single calendar event.
    *   **`eventId: string`**: Required. The unique identifier of the event to retrieve.

#### `GoogleCalendarUpdateParams` (Update an Event)

```typescript
export interface GoogleCalendarUpdateParams extends BaseGoogleCalendarParams {
  eventId: string
  summary?: string
  description?: string
  location?: string
  startDateTime?: string
  endDateTime?: string
  timeZone?: string
  attendees?: string[]
  sendUpdates?: 'all' | 'externalOnly' | 'none'
}
```
*   **`export interface GoogleCalendarUpdateParams extends BaseGoogleCalendarParams`**: Defines parameters for updating an existing calendar event.
    *   **`eventId: string`**: Required. The unique identifier of the event to update.
    *   The other properties (`summary`, `description`, `location`, `startDateTime`, `endDateTime`, `timeZone`, `attendees`, `sendUpdates`) are all optional. This means you only provide the fields you want to change; omitted fields remain untouched.

#### `GoogleCalendarDeleteParams` (Delete an Event)

```typescript
export interface GoogleCalendarDeleteParams extends BaseGoogleCalendarParams {
  eventId: string
  sendUpdates?: 'all' | 'externalOnly' | 'none'
}
```
*   **`export interface GoogleCalendarDeleteParams extends BaseGoogleCalendarParams`**: Defines parameters for deleting a calendar event.
    *   **`eventId: string`**: Required. The unique identifier of the event to delete.
    *   **`sendUpdates?: 'all' | 'externalOnly' | 'none'`**: Optional. Controls who receives email updates about the event deletion, similar to `CreateParams`.

#### `GoogleCalendarQuickAddParams` (Quick Add Event)

```typescript
export interface GoogleCalendarQuickAddParams extends BaseGoogleCalendarParams {
  text: string // Natural language text like "Meeting with John tomorrow at 3pm"
  attendees?: string[] // Array of email addresses (comma-separated string also accepted)
  sendUpdates?: 'all' | 'externalOnly' | 'none'
}
```
*   **`export interface GoogleCalendarQuickAddParams extends BaseGoogleCalendarParams`**: Defines parameters for quickly adding an event using natural language text.
    *   **`text: string`**: Required. The natural language description of the event (e.g., "Lunch with Sarah next Tuesday at noon"). Google will parse this to create the event.
    *   **`attendees?: string[]`**: Optional. An array of email addresses for attendees. The comment notes that a comma-separated string is also accepted by the underlying API, but this interface prefers an array for type safety.
    *   **`sendUpdates?: 'all' | 'externalOnly' | 'none'`**: Optional. Controls email updates for this quick-added event.

#### `GoogleCalendarInviteParams` (Invite Attendees to an Event)

```typescript
export interface GoogleCalendarInviteParams extends BaseGoogleCalendarParams {
  eventId: string
  attendees: string[] // Array of email addresses to invite
  sendUpdates?: 'all' | 'externalOnly' | 'none'
  replaceExisting?: boolean // Whether to replace existing attendees or add to them
}
```
*   **`export interface GoogleCalendarInviteParams extends BaseGoogleCalendarParams`**: Defines parameters for inviting attendees to an *existing* event.
    *   **`eventId: string`**: Required. The ID of the event to invite attendees to.
    *   **`attendees: string[]`**: Required. An array of email addresses of the new attendees to invite.
    *   **`sendUpdates?: 'all' | 'externalOnly' | 'none'`**: Optional. Controls email updates.
    *   **`replaceExisting?: boolean`**: Optional. If `true`, the `attendees` array will replace *all* current attendees on the event. If `false` (or omitted), these attendees will be *added* to the existing list.

---

### Union Type for All Google Calendar Tool Parameters

```typescript
export type GoogleCalendarToolParams =
  | GoogleCalendarCreateParams
  | GoogleCalendarListParams
  | GoogleCalendarGetParams
  | GoogleCalendarUpdateParams
  | GoogleCalendarDeleteParams
  | GoogleCalendarQuickAddParams
  | GoogleCalendarInviteParams
```
*   **`export type GoogleCalendarToolParams`**: This is a **union type**.
    *   **`export type`**: Defines a new type that can be used elsewhere.
    *   This type `GoogleCalendarToolParams` can be *any one* of the specific parameter interfaces defined above. This is incredibly useful when you have a function that can accept parameters for *any* Google Calendar operation. The function can then use type narrowing (e.g., checking for specific properties like `eventId` or `text`) to determine which operation to perform.

---

### Tool Response Metadata Interfaces

These interfaces define the structure of the *metadata* returned by the Google Calendar tool wrapper. This metadata is a more structured, application-specific representation of the underlying Google API response.

#### `EventMetadata`

```typescript
interface EventMetadata {
  id: string
  htmlLink: string
  status: string
  summary: string
  description?: string
  location?: string
  start: {
    dateTime?: string
    date?: string
    timeZone?: string
  }
  end: {
    dateTime?: string
    date?: string
    timeZone?: string
  }
  attendees?: CalendarAttendee[]
  creator?: {
    email: string
    displayName?: string
  }
  organizer?: {
    email: string
    displayName?: string
  }
}
```
*   **`interface EventMetadata`**: Defines the metadata structure for a single calendar event, as it would be presented by the tool.
    *   **`id: string`**: The event's unique identifier.
    *   **`htmlLink: string`**: A URL to view the event in Google Calendar.
    *   **`status: string`**: The event's status (e.g., "confirmed", "cancelled").
    *   **`summary: string`**: The event's title.
    *   **`description?: string`**: Optional. The event's description.
    *   **`location?: string`**: Optional. The event's location.
    *   **`start: { ... }`**: An object defining the event's start time.
        *   **`dateTime?: string`**: Optional. The specific date and time (RFC3339).
        *   **`date?: string`**: Optional. The date for all-day events (e.g., `YYYY-MM-DD`).
        *   **`timeZone?: string`**: Optional. The time zone.
    *   **`end: { ... }`**: Similar structure for the event's end time.
    *   **`attendees?: CalendarAttendee[]`**: Optional. An array of `CalendarAttendee` objects for people invited.
    *   **`creator?: { email: string; displayName?: string }`**: Optional. An object describing the creator of the event.
    *   **`organizer?: { email: string; displayName?: string }`**: Optional. An object describing the organizer of the event.

#### `ListMetadata`

```typescript
interface ListMetadata {
  nextPageToken?: string
  nextSyncToken?: string
  events: EventMetadata[]
  timeZone: string
}
```
*   **`interface ListMetadata`**: Defines the metadata structure for a *list* of calendar events.
    *   **`nextPageToken?: string`**: Optional. A token to retrieve the next page of results if the list is paginated.
    *   **`nextSyncToken?: string`**: Optional. A token used for incremental synchronization of events.
    *   **`events: EventMetadata[]`**: Required. An array containing multiple `EventMetadata` objects.
    *   **`timeZone: string`**: The time zone used for the list of events.

---

### Google Calendar Tool Response Interfaces

These interfaces define the full response shape from the tool wrapper, combining the generic `ToolResponse` with specific output `metadata`.

#### `GoogleCalendarToolResponse` (Generic Tool Response)

```typescript
export interface GoogleCalendarToolResponse extends ToolResponse {
  output: {
    content: string
    metadata: EventMetadata | ListMetadata
  }
}
```
*   **`export interface GoogleCalendarToolResponse extends ToolResponse`**: This is a generic interface for *any* Google Calendar tool response. It extends the base `ToolResponse` (imported at the top) and adds a specific `output` structure.
    *   **`output: { ... }`**: This object holds the actual data returned by the tool.
        *   **`content: string`**: A human-readable string summarizing the result of the operation (e.g., "Event created successfully").
        *   **`metadata: EventMetadata | ListMetadata`**: The structured data, which can be either a single `EventMetadata` object (for create, get, update, delete) or a `ListMetadata` object (for listing events). This uses a union type.

#### Specific Operation Response Interfaces

These provide more precise type definitions for each operation's response, narrowing down the `metadata` type.

```typescript
export interface GoogleCalendarCreateResponse extends ToolResponse {
  output: {
    content: string
    metadata: EventMetadata
  }
}

export interface GoogleCalendarListResponse extends ToolResponse {
  output: {
    content: string
    metadata: ListMetadata
  }
}

export interface GoogleCalendarGetResponse extends ToolResponse {
  output: {
    content: string
    metadata: EventMetadata
  }
}

export interface GoogleCalendarQuickAddResponse extends ToolResponse {
  output: {
    content: string
    metadata: EventMetadata
  }
}

export interface GoogleCalendarUpdateResponse extends ToolResponse {
  output: {
    content: string
    metadata: EventMetadata
  }
}

export interface GoogleCalendarInviteResponse extends ToolResponse {
  output: {
    content: string
    metadata: EventMetadata
  }
}
```
*   These interfaces all follow the same pattern: they extend `ToolResponse` and define an `output` object with `content: string`. The key difference is that the `metadata` property is now specifically typed to `EventMetadata` or `ListMetadata` depending on the operation, providing more granular type safety.
    *   `CreateResponse`, `GetResponse`, `QuickAddResponse`, `UpdateResponse`, `InviteResponse` all specify `metadata: EventMetadata` because they deal with a single event.
    *   `ListResponse` specifies `metadata: ListMetadata` because it deals with a collection of events.

---

### Google Calendar API Raw Response Interfaces

These interfaces directly mirror the expected structure of responses from the *raw* Google Calendar API, which might differ slightly from the tool's simplified `EventMetadata`. These are crucial for the internal implementation of the tool that interacts directly with Google's API.

#### `GoogleCalendarEvent`

```typescript
export interface GoogleCalendarEvent {
  id: string
  status: string
  htmlLink: string
  created: string
  updated: string
  summary: string
  description?: string
  location?: string
  start: {
    dateTime?: string
    date?: string
    timeZone?: string
  }
  end: {
    dateTime?: string
    date?: string
    timeZone?: string
  }
  attendees?: CalendarAttendee[]
  creator?: {
    email: string
    displayName?: string
  }
  organizer?: {
    email: string
    displayName?: string
  }
  reminders?: {
    useDefault: boolean
    overrides?: Array<{
      method: string
      minutes: number
    }>
  }
}
```
*   **`export interface GoogleCalendarEvent`**: This interface represents a complete Google Calendar event object returned by the API. It includes all the fields that `EventMetadata` has, plus additional API-specific fields like `created`, `updated`, and `reminders`.
    *   **`created: string`**: The creation timestamp of the event.
    *   **`updated: string`**: The last modification timestamp of the event.
    *   **`reminders?: { ... }`**: Optional. An object detailing event reminders.
        *   **`useDefault: boolean`**: Whether to use the default reminders for the calendar.
        *   **`overrides?: Array<{ method: string; minutes: number }>`**: Optional. Specific reminder settings (e.g., method: "email", minutes: 30).

#### `GoogleCalendarEventRequestBody`

```typescript
export interface GoogleCalendarEventRequestBody {
  summary: string
  description?: string
  location?: string
  start: {
    dateTime: string
    timeZone?: string
  }
  end: {
    dateTime: string
    timeZone?: string
  }
  attendees?: Array<{
    email: string
  }>
}
```
*   **`export interface GoogleCalendarEventRequestBody`**: Defines the minimal structure for the *body* of a request to *create or update* an event via the Google Calendar API.
    *   Notice `dateTime` is required within `start` and `end` here, as an event *must* have a start/end time.
    *   `attendees` here is simplified to just an array of objects containing `email`, as that's typically all that's sent when creating or updating. Other attendee properties are usually generated by Google.

#### `GoogleCalendarApiEventResponse`

```typescript
export interface GoogleCalendarApiEventResponse {
  id: string
  status: string
  htmlLink: string
  created?: string
  updated?: string
  summary: string
  description?: string
  location?: string
  start: {
    dateTime?: string
    date?: string
    timeZone?: string
  }
  end: {
    dateTime?: string
    date?: string
    timeZone?: string
  }
  attendees?: CalendarAttendee[]
  creator?: {
    email: string
    displayName?: string
  }
  organizer?: {
    email: string
    displayName?: string
  }
  reminders?: {
    useDefault: boolean
    overrides?: Array<{
      method: string
      minutes: number
    }>
  }
}
```
*   **`export interface GoogleCalendarApiEventResponse`**: Similar to `GoogleCalendarEvent`, this represents the actual response structure for a single event from the Google API. The primary difference here is that `created` and `updated` are marked as optional (`?`). This might reflect that these fields could be absent in certain scenarios or for specific event types from the API.

#### `GoogleCalendarApiListResponse`

```typescript
export interface GoogleCalendarApiListResponse {
  kind: string
  etag: string
  summary: string
  description?: string
  updated: string
  timeZone: string
  accessRole: string
  defaultReminders: Array<{
    method: string
    minutes: number
  }>
  nextPageToken?: string
  nextSyncToken?: string
  items: GoogleCalendarApiEventResponse[]
}
```
*   **`export interface GoogleCalendarApiListResponse`**: This interface defines the complete structure of a response when listing events directly from the Google Calendar API.
    *   **`kind: string`**: The type of the resource (e.g., "calendar#events").
    *   **`etag: string`**: An opaque string that changes whenever the resource changes. Used for optimistic locking.
    *   **`summary: string`**: The calendar's summary (e.g., "My Calendar").
    *   **`description?: string`**: Optional. The calendar's description.
    *   **`updated: string`**: The last modification timestamp of the calendar.
    *   **`timeZone: string`**: The time zone of the calendar.
    *   **`accessRole: string`**: The user's access role to the calendar (e.g., "owner", "writer").
    *   **`defaultReminders: Array<{ method: string; minutes: number }>`**: An array of default reminder settings for events on this calendar.
    *   **`nextPageToken?: string`**: Optional. Token for fetching the next page of results.
    *   **`nextSyncToken?: string`**: Optional. Token for incremental synchronization.
    *   **`items: GoogleCalendarApiEventResponse[]`**: Required. An array of `GoogleCalendarApiEventResponse` objects, representing the events found.

---

### Union Type for All Google Calendar API Responses

```typescript
export type GoogleCalendarResponse =
  | GoogleCalendarCreateResponse
  | GoogleCalendarListResponse
  | GoogleCalendarGetResponse
  | GoogleCalendarQuickAddResponse
  | GoogleCalendarInviteResponse
  | GoogleCalendarUpdateResponse
```
*   **`export type GoogleCalendarResponse`**: This is another **union type**, similar to `GoogleCalendarToolParams`.
    *   It represents the possible *high-level tool responses* (not the raw API responses) for any of the Google Calendar operations. This can be used in functions that might return the result of *any* Google Calendar operation, allowing for flexible type checking.

---

### Conclusion

This file serves as a robust type definition library for working with the Google Calendar API. It distinguishes between:
1.  **Request Parameters**: What you send to the API.
2.  **Tool-Level Responses**: Simplified, application-specific data structures returned by the wrapper tool.
3.  **Raw API Responses**: Detailed data structures directly mirroring Google's API, used for internal parsing and mapping.

This level of detail dramatically improves the maintainability, readability, and reliability of any TypeScript code interacting with the Google Calendar service.