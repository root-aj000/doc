This TypeScript file defines API endpoints for managing organizations within a Next.js application. Specifically, it provides functionality to:

1.  **Retrieve Organization Details (`GET /api/organizations/[id]`):** Fetches an organization's basic information, settings, and optionally, its seat-management data and analytics, based on the requesting user's permissions.
2.  **Update Organization Settings or Seat Count (`PUT /api/organizations/[id]`):** Allows an administrator or owner to modify an organization's name, slug, logo, or adjust its total seat count.

It leverages Drizzle ORM for database interactions, Next.js API Routes for handling HTTP requests, and includes robust authentication and authorization checks to ensure secure access.

---

### **Detailed Code Explanation**

First, let's break down the imports and global setup.

```typescript
import { db } from '@sim/db'
import { member, organization } from '@sim/db/schema'
import { and, eq, ne } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import { getSession } from '@/lib/auth'
import {
  getOrganizationSeatAnalytics,
  getOrganizationSeatInfo,
  updateOrganizationSeats,
} from '@/lib/billing/validation/seat-management'
import { createLogger } from '@/lib/logs/console/logger'
```

*   **`import { db } from '@sim/db'`**: Imports the Drizzle ORM database client instance. This `db` object is used to perform all database operations (e.g., `select`, `update`).
*   **`import { member, organization } from '@sim/db/schema'`**: Imports the Drizzle schema definitions for the `member` and `organization` tables. These are used to reference the actual database tables and their columns in queries.
*   **`import { and, eq, ne } from 'drizzle-orm'`**: Imports utility functions from Drizzle ORM for building complex query conditions:
    *   `and`: Combines multiple conditions with a logical AND.
    *   `eq`: Checks for equality (e.g., `column === value`).
    *   `ne`: Checks for inequality (e.g., `column !== value`).
*   **`import { type NextRequest, NextResponse } from 'next/server'`**: Imports types and classes from Next.js for handling server-side requests and responses:
    *   `NextRequest`: Represents the incoming HTTP request, providing access to URL, headers, body, etc.
    *   `NextResponse`: Used to construct and send HTTP responses, including JSON data and status codes.
*   **`import { getSession } from '@/lib/auth'`**: Imports a helper function to retrieve the user's session information, which contains authentication details.
*   **`import { ... } from '@/lib/billing/validation/seat-management'`**: Imports functions related to managing organization seats:
    *   `getOrganizationSeatAnalytics`: Fetches detailed analytics about seat usage.
    *   `getOrganizationSeatInfo`: Fetches basic information about seat allocation.
    *   `updateOrganizationSeats`: Handles the logic for changing the total number of seats for an organization.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a utility to create a logger instance for structured logging.

```typescript
const logger = createLogger('OrganizationAPI')
```

*   **`const logger = createLogger('OrganizationAPI')`**: Initializes a logger specifically for this API file. The string `'OrganizationAPI'` acts as a context, making it easier to identify logs originating from this part of the application. This is crucial for debugging and monitoring.

---

### **`GET /api/organizations/[id]` Endpoint**

This function handles `GET` requests to retrieve information about a specific organization.

```typescript
/**
 * GET /api/organizations/[id]
 * Get organization details including settings and seat information
 */
export async function GET(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  try {
    // ... (rest of the GET logic)
  } catch (error) {
    // ... (error handling)
  }
}
```

*   **`export async function GET(...)`**: Defines an asynchronous function named `GET`. In Next.js, an exported `GET` function in an API route file automatically handles `GET` requests for that route.
*   **`request: NextRequest`**: The first argument is the incoming HTTP request object, provided by Next.js.
*   **`{ params }: { params: Promise<{ id: string }> }`**: The second argument is an object containing route parameters. Since this route is `api/organizations/[id]`, `params` will contain an `id` property corresponding to the organization's ID from the URL. The `Promise` indicates that Next.js might resolve `params` asynchronously.
*   **`try { ... } catch (error) { ... }`**: This block encloses the entire API logic. It's a standard practice to catch any unexpected errors during execution and return a generic 500 error response, preventing the server from crashing and providing a better user experience.

#### **Inside the `try` block:**

```typescript
    const session = await getSession()

    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
```

*   **`const session = await getSession()`**: Retrieves the current user's session data. This typically includes information like the logged-in user's ID, name, email, etc.
*   **`if (!session?.user?.id)`**: Checks if a session exists and if there's a user ID associated with it. If not, the user is not authenticated.
*   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`**: If the user is not authenticated, a 401 Unauthorized status is returned, preventing access to organization data.

```typescript
    const { id: organizationId } = await params
    const url = new URL(request.url)
    const includeSeats = url.searchParams.get('include') === 'seats'
```

*   **`const { id: organizationId } = await params`**: Extracts the `id` from the `params` object (which is a `Promise`) and renames it to `organizationId` for clarity. This is the ID of the organization being requested.
*   **`const url = new URL(request.url)`**: Creates a URL object from the incoming request's URL. This provides an easy way to parse URL components.
*   **`const includeSeats = url.searchParams.get('include') === 'seats'`**: Checks if the query parameter `include` is set to `'seats'` (e.g., `/api/organizations/org123?include=seats`). This flag determines whether additional seat-related information should be included in the response.

```typescript
    // Verify user has access to this organization
    const memberEntry = await db
      .select()
      .from(member)
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, session.user.id)))
      .limit(1)

    if (memberEntry.length === 0) {
      return NextResponse.json(
        { error: 'Forbidden - Not a member of this organization' },
        { status: 403 }
      )
    }
```

*   **`db.select().from(member)...`**: This is a Drizzle ORM query to check if the authenticated user is a member of the requested organization.
    *   **`.select()`**: Selects all columns from the table.
    *   **`.from(member)`**: Specifies that we are querying the `member` table.
    *   **`.where(and(eq(member.organizationId, organizationId), eq(member.userId, session.user.id)))`**: This is the core condition:
        *   `and(...)`: Combines two conditions.
        *   `eq(member.organizationId, organizationId)`: Checks if the `organizationId` column in the `member` table matches the `organizationId` from the URL.
        *   `eq(member.userId, session.user.id)`: Checks if the `userId` column in the `member` table matches the ID of the currently logged-in user.
    *   **`.limit(1)`**: We only need to find one matching entry to confirm membership, so limiting the result to 1 is efficient.
*   **`if (memberEntry.length === 0)`**: If no matching `memberEntry` is found, it means the authenticated user is not a member of this organization.
*   **`return NextResponse.json({ error: 'Forbidden - Not a member of this organization' }, { status: 403 })`**: Returns a 403 Forbidden status, indicating that the user does not have permission to access this organization's data.

```typescript
    // Get organization data
    const organizationEntry = await db
      .select()
      .from(organization)
      .where(eq(organization.id, organizationId))
      .limit(1)

    if (organizationEntry.length === 0) {
      return NextResponse.json({ error: 'Organization not found' }, { status: 404 })
    }
```

*   **`db.select().from(organization)...`**: This query retrieves the actual organization data.
    *   **`.from(organization)`**: Specifies the `organization` table.
    *   **`.where(eq(organization.id, organizationId))`**: Filters the organizations by the `organizationId` from the URL.
    *   **`.limit(1)`**: Again, we expect only one organization with a given ID.
*   **`if (organizationEntry.length === 0)`**: If no organization with the given ID is found (which shouldn't happen if the user is a member, but is a good defensive check).
*   **`return NextResponse.json({ error: 'Organization not found' }, { status: 404 })`**: Returns a 404 Not Found status.

```typescript
    const userRole = memberEntry[0].role
    const hasAdminAccess = ['owner', 'admin'].includes(userRole)
```

*   **`const userRole = memberEntry[0].role`**: Extracts the role of the user within this organization from the previously fetched `memberEntry`.
*   **`const hasAdminAccess = ['owner', 'admin'].includes(userRole)`**: Determines if the user's role is either `'owner'` or `'admin'`, indicating they have administrative privileges. This will be used for conditional data fetching later.

```typescript
    const response: any = {
      success: true,
      data: {
        id: organizationEntry[0].id,
        name: organizationEntry[0].name,
        slug: organizationEntry[0].slug,
        logo: organizationEntry[0].logo,
        metadata: organizationEntry[0].metadata,
        createdAt: organizationEntry[0].createdAt,
        updatedAt: organizationEntry[0].updatedAt,
      },
      userRole,
      hasAdminAccess,
    }
```

*   **`const response: any = { ... }`**: Constructs the base response object.
    *   **`success: true`**: A common pattern for API responses, indicating the request was processed successfully.
    *   **`data: { ... }`**: Contains the core organization details, extracted directly from the `organizationEntry`.
    *   **`userRole`**: Includes the requesting user's role within this organization.
    *   **`hasAdminAccess`**: Includes a boolean flag indicating if the user has admin rights.

```typescript
    // Include seat information if requested
    if (includeSeats) {
      const seatInfo = await getOrganizationSeatInfo(organizationId)
      if (seatInfo) {
        response.data.seats = seatInfo
      }

      // Include analytics for admins
      if (hasAdminAccess) {
        const analytics = await getOrganizationSeatAnalytics(organizationId)
        if (analytics) {
          response.data.seatAnalytics = analytics
        }
      }
    }
```

*   **`if (includeSeats)`**: This block executes only if the `?include=seats` query parameter was present in the request URL.
    *   **`const seatInfo = await getOrganizationSeatInfo(organizationId)`**: Calls a helper function to fetch general seat information (e.g., total seats, occupied seats).
    *   **`if (seatInfo) { response.data.seats = seatInfo }`**: If seat information is found, it's added to the `data` object under the `seats` key.
    *   **`if (hasAdminAccess)`**: Inside the `includeSeats` block, this nested condition further checks if the user has admin access.
        *   **`const analytics = await getOrganizationSeatAnalytics(organizationId)`**: Calls a helper function to fetch more detailed seat analytics (e.g., historical usage, breakdown).
        *   **`if (analytics) { response.data.seatAnalytics = analytics }`**: If analytics are found, they are added to the `data` object under the `seatAnalytics` key. This ensures sensitive analytics data is only exposed to authorized users.

```typescript
    return NextResponse.json(response)
```

*   **`return NextResponse.json(response)`**: Sends the final constructed JSON response back to the client with a default 200 OK status.

#### **Inside the `catch` block:**

```typescript
  } catch (error) {
    logger.error('Failed to get organization', {
      organizationId: (await params).id,
      error,
    })

    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
```

*   **`logger.error('Failed to get organization', { ... })`**: If any error occurs within the `try` block, it's caught here. The error is logged with a descriptive message, including the `organizationId` (to help identify the problematic request) and the `error` object itself.
*   **`return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`**: A generic 500 Internal Server Error is returned to the client. This prevents leaking sensitive error details while indicating that something went wrong on the server.

---

### **`PUT /api/organizations/[id]` Endpoint**

This function handles `PUT` requests to update an organization's settings or seat count.

```typescript
/**
 * PUT /api/organizations/[id]
 * Update organization settings or seat count
 */
export async function PUT(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  try {
    // ... (rest of the PUT logic)
  } catch (error) {
    // ... (error handling)
  }
}
```

*   **`export async function PUT(...)`**: Defines an asynchronous function named `PUT`. In Next.js, an exported `PUT` function handles `PUT` requests for this route.
*   The `request` and `params` arguments are the same as for the `GET` function.
*   The `try { ... } catch (error) { ... }` block is also for robust error handling.

#### **Inside the `try` block:**

```typescript
    const session = await getSession()

    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
```

*   **Authentication Check**: This is identical to the `GET` endpoint, ensuring only authenticated users can attempt to update an organization.

```typescript
    const { id: organizationId } = await params
    const body = await request.json()
    const { name, slug, logo, seats } = body
```

*   **`const { id: organizationId } = await params`**: Extracts the `organizationId` from the URL parameters.
*   **`const body = await request.json()`**: Parses the JSON body of the incoming `PUT` request. This body is expected to contain the fields to be updated.
*   **`const { name, slug, logo, seats } = body`**: Destructures the `body` object to extract specific fields: `name`, `slug`, `logo` (for organization settings) and `seats` (for seat count updates).

```typescript
    // Verify user has admin access
    const memberEntry = await db
      .select()
      .from(member)
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, session.user.id)))
      .limit(1)

    if (memberEntry.length === 0) {
      return NextResponse.json(
        { error: 'Forbidden - Not a member of this organization' },
        { status: 403 }
      )
    }

    if (!['owner', 'admin'].includes(memberEntry[0].role)) {
      return NextResponse.json({ error: 'Forbidden - Admin access required' }, { status: 403 })
    }
```

*   **Authorization Check (Admin Access)**:
    *   The first part of this check is identical to the `GET` request, verifying that the user is a member of the organization. If not, a 403 Forbidden is returned.
    *   **`if (!['owner', 'admin'].includes(memberEntry[0].role))`**: This is an additional check for `PUT` requests. It ensures that the user's role within the organization is either `'owner'` or `'admin'`. Regular members cannot update organization settings or seat counts. If the user does not have an admin role, another 403 Forbidden is returned.

#### **Handling Seat Count Update:**

```typescript
    if (seats !== undefined) {
      if (typeof seats !== 'number' || seats < 1) {
        return NextResponse.json({ error: 'Invalid seat count' }, { status: 400 })
      }

      const result = await updateOrganizationSeats(organizationId, seats, session.user.id)

      if (!result.success) {
        return NextResponse.json({ error: result.error }, { status: 400 })
      }

      logger.info('Organization seat count updated', {
        organizationId,
        newSeatCount: seats,
        updatedBy: session.user.id,
      })

      return NextResponse.json({
        success: true,
        message: 'Seat count updated successfully',
        data: {
          seats: seats,
          updatedBy: session.user.id,
          updatedAt: new Date().toISOString(),
        },
      })
    }
```

*   **`if (seats !== undefined)`**: This block executes if the `seats` field was provided in the request body, indicating an intent to update the seat count.
    *   **`if (typeof seats !== 'number' || seats < 1)`**: Basic validation: ensures `seats` is a number and is at least 1. If not, a 400 Bad Request is returned.
    *   **`const result = await updateOrganizationSeats(organizationId, seats, session.user.id)`**: Calls a specialized helper function to handle the complex logic of updating organization seats. This function likely includes checks for billing, plan limits, and ensures data consistency. It also takes the `userId` to log who initiated the change.
    *   **`if (!result.success)`**: The `updateOrganizationSeats` function is expected to return an object with a `success` property. If `success` is `false`, it means the update failed (e.g., due to billing limits), and its `error` message is returned to the client with a 400 status.
    *   **`logger.info('Organization seat count updated', { ... })`**: On successful seat count update, an informational log is recorded, including the organization ID, new seat count, and the user who made the change.
    *   **`return NextResponse.json({ success: true, message: 'Seat count updated successfully', data: { ... } })`**: Returns a success response, confirming the update and providing the new `seats` value, who updated it, and the `updatedAt` timestamp.

#### **Handling Settings Update:**

```typescript
    if (name !== undefined || slug !== undefined || logo !== undefined) {
      // Validate required fields
      if (name !== undefined && (!name || typeof name !== 'string' || name.trim().length === 0)) {
        return NextResponse.json({ error: 'Organization name is required' }, { status: 400 })
      }

      // ... (rest of settings update logic)
    }
```

*   **`if (name !== undefined || slug !== undefined || logo !== undefined)`**: This block executes if any of the `name`, `slug`, or `logo` fields were provided in the request body, indicating an intent to update organization settings. It will *not* execute if `seats` was handled above.
    *   **Name Validation**:
        *   **`if (name !== undefined && (!name || typeof name !== 'string' || name.trim().length === 0))`**: Checks if `name` was provided, and if so, validates that it's a non-empty string after trimming whitespace. If invalid, returns a 400 Bad Request.

```typescript
      if (slug !== undefined && (!slug || typeof slug !== 'string' || slug.trim().length === 0)) {
        return NextResponse.json({ error: 'Organization slug is required' }, { status: 400 })
      }
```

*   **Slug Validation (Presence and Type)**: Similar to name, validates that `slug` is a non-empty string if provided.

```typescript
      // Validate slug format
      if (slug !== undefined) {
        const slugRegex = /^[a-z0-9-_]+$/
        if (!slugRegex.test(slug)) {
          return NextResponse.json(
            {
              error: 'Slug can only contain lowercase letters, numbers, hyphens, and underscores',
            },
            { status: 400 }
          )
        }
```

*   **Slug Validation (Format)**:
    *   **`const slugRegex = /^[a-z0-9-_]+$/`**: Defines a regular expression to ensure the slug only contains lowercase letters, numbers, hyphens (`-`), and underscores (`_`).
    *   **`if (!slugRegex.test(slug))`**: If the provided `slug` does not match the regex, a 400 Bad Request error is returned with a descriptive message about the allowed characters.

```typescript
        // Check if slug is already taken by another organization
        const existingSlug = await db
          .select()
          .from(organization)
          .where(and(eq(organization.slug, slug), ne(organization.id, organizationId)))
          .limit(1)

        if (existingSlug.length > 0) {
          return NextResponse.json({ error: 'This slug is already taken' }, { status: 400 })
        }
      }
```

*   **Slug Uniqueness Check**:
    *   **`db.select().from(organization)...`**: Queries the `organization` table.
    *   **`.where(and(eq(organization.slug, slug), ne(organization.id, organizationId)))`**: Checks for an existing organization with the *same slug* (`eq(organization.slug, slug)`) but a *different ID* (`ne(organization.id, organizationId)`). This `ne` (not equal) condition is crucial to allow an organization to update its own slug without it being flagged as taken by itself.
    *   **`if (existingSlug.length > 0)`**: If another organization with the same slug is found, a 400 Bad Request error is returned.

```typescript
      // Build update object with only provided fields
      const updateData: any = { updatedAt: new Date() }
      if (name !== undefined) updateData.name = name.trim()
      if (slug !== undefined) updateData.slug = slug.trim()
      if (logo !== undefined) updateData.logo = logo || null
```

*   **`const updateData: any = { updatedAt: new Date() }`**: Initializes an object `updateData` that will hold the fields to be updated. It always includes `updatedAt` to record the time of the change.
*   **`if (name !== undefined) updateData.name = name.trim()`**: If `name` was provided, its trimmed value is added to `updateData`.
*   **`if (slug !== undefined) updateData.slug = slug.trim()`**: If `slug` was provided, its trimmed value is added to `updateData`.
*   **`if (logo !== undefined) updateData.logo = logo || null`**: If `logo` was provided, its value is added. If `logo` is an empty string or falsy, it will be set to `null` in the database, effectively clearing the logo.

```typescript
      // Update organization
      const updatedOrg = await db
        .update(organization)
        .set(updateData)
        .where(eq(organization.id, organizationId))
        .returning()

      if (updatedOrg.length === 0) {
        return NextResponse.json({ error: 'Organization not found' }, { status: 404 })
      }
```

*   **`db.update(organization).set(updateData)...`**: This is the Drizzle ORM query to perform the update.
    *   **`.update(organization)`**: Specifies the `organization` table for the update.
    *   **`.set(updateData)`**: Applies the changes defined in the `updateData` object.
    *   **`.where(eq(organization.id, organizationId))`**: Ensures that only the organization matching the `organizationId` from the URL is updated.
    *   **`.returning()`**: This is important! It tells Drizzle to return the *updated* rows. Without it, `updatedOrg` would just contain metadata about the update, not the actual updated record.
*   **`if (updatedOrg.length === 0)`**: If no organization was found and updated (e.g., the ID was valid but no record exists, which should ideally be caught earlier, but serves as a final safeguard), returns a 404 Not Found.

```typescript
      logger.info('Organization settings updated', {
        organizationId,
        updatedBy: session.user.id,
        changes: { name, slug, logo },
      })

      return NextResponse.json({
        success: true,
        message: 'Organization updated successfully',
        data: {
          id: updatedOrg[0].id,
          name: updatedOrg[0].name,
          slug: updatedOrg[0].slug,
          logo: updatedOrg[0].logo,
          updatedAt: updatedOrg[0].updatedAt,
        },
      })
    }
```

*   **`logger.info('Organization settings updated', { ... })`**: Logs the successful update of organization settings, including the organization ID, the user who updated it, and the specific fields that were changed.
*   **`return NextResponse.json({ success: true, message: 'Organization updated successfully', data: { ... } })`**: Returns a success response to the client, confirming the update and providing the new organization details (id, name, slug, logo, updatedAt).

```typescript
    return NextResponse.json({ error: 'No valid fields provided for update' }, { status: 400 })
```

*   **`return NextResponse.json({ error: 'No valid fields provided for update' }, { status: 400 })`**: If neither the `seats` block nor the settings (`name`, `slug`, `logo`) block was executed (meaning the request body was empty or contained fields not supported for update by this endpoint), a 400 Bad Request error is returned.

#### **Inside the `catch` block:**

```typescript
  } catch (error) {
    logger.error('Failed to update organization', {
      organizationId: (await params).id,
      error,
    })

    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
```

*   **`logger.error('Failed to update organization', { ... })`**: Similar to the `GET` endpoint, any unexpected errors during the `PUT` operation are logged.
*   **`return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`**: A generic 500 Internal Server Error is returned to the client.

---

### **`DELETE` Method Comment**

```typescript
// DELETE method removed - organization deletion not implemented
// If deletion is needed in the future, it should be implemented with proper
// cleanup of subscriptions, members, workspaces, and billing data
```

*   This comment explicitly states that a `DELETE` endpoint for organizations is *not* implemented in this file.
*   It also provides a crucial warning: implementing organization deletion is complex. It would require a cascading deletion or careful cleanup of all related data, such as user memberships, associated workspaces, billing subscriptions, and any other linked entities, to maintain data integrity and avoid orphaned records. This is a good practice note for future development.