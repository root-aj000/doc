This TypeScript file defines a Next.js API route that serves unified billing information. It's designed to be flexible, capable of providing billing details for either a specific user or an entire organization, based on the request's query parameters.

---

### **Purpose of This File**

The primary purpose of `app/api/billing/route.ts` is to act as a single, central endpoint for retrieving billing-related data. Instead of having separate API endpoints for user billing and organization billing, this file unifies them.

Here's a breakdown of its key responsibilities:

1.  **Authentication & Authorization**: It ensures that only authenticated users can access billing data and, for organization-specific data, verifies that the requesting user is a member of that organization.
2.  **Context-Driven Data Retrieval**: It dynamically switches between fetching user-centric or organization-centric billing data based on the `context` query parameter in the request.
3.  **Data Consolidation**: It combines various pieces of billing information, potentially from different sources (e.g., a simplified summary for users, detailed metrics for organizations), and also attaches a `billingBlocked` status for the requesting user.
4.  **Data Formatting**: It transforms raw data (like database date objects) into a consistent and easily consumable format (e.g., ISO strings) for the client-side.
5.  **Error Handling**: It provides robust error handling, returning appropriate HTTP status codes and messages for issues like unauthorized access, invalid requests, or internal server problems.
6.  **Logging**: It logs critical errors to aid in debugging and monitoring.

In essence, it's a versatile API gateway for all billing-related UI components to consume.

---

### **Simplified Complex Logic**

The core logic of this file revolves around a "context" system. Imagine you have two different dashboards: one for an individual user to see their personal billing, and another for an organization admin to see their company's billing. This API route serves both by checking a `context` parameter in the URL.

1.  **You must be logged in**: No billing data is shown to guests.
2.  **Tell me what you want**: The API expects a `context` parameter.
    *   If `context=user`: It will fetch billing data primarily related to the logged-in individual. It might still include organization details if the user is part of one, but the focus is on the individual's view. It also checks if *that user's* billing is blocked.
    *   If `context=organization`: You *also* need to provide an `id` parameter (the organization's ID).
        *   **Permission Check**: Before anything else, it verifies if the logged-in user is actually a member of that specific organization. If not, access is denied.
        *   **Organization Data**: It then fetches detailed billing for the organization.
        *   **Your Blocked Status**: Even for organization billing, it still includes whether the *logged-in user's* personal billing is blocked. This is useful if the UI needs to show a warning to that specific user.
3.  **Error Handling**: If anything goes wrong (not logged in, wrong parameters, not authorized, organization not found), it immediately stops and sends back an error message with a clear status code.
4.  **Data Shaping**: The raw data retrieved from the database or helper functions is often "shaped" or transformed into a more user-friendly format (e.g., dates are converted to text strings) before being sent back.

In summary, this API is like a smart cashier: it first checks your ID, then asks if you want your personal bill or the company's bill (and needs the company's name if you choose that), then fetches the correct bill, double-checks if you're allowed to see the company's bill, and finally presents it in a clear, easy-to-read format.

---

### **Line-by-Line Explanation**

Let's walk through the code step by step:

```typescript
import { db } from '@sim/db'
```
*   **`import { db } from '@sim/db'`**: Imports the `db` object, which is the Drizzle ORM database client instance. This allows the API route to interact with the database.

```typescript
import { member, userStats } from '@sim/db/schema'
```
*   **`import { member, userStats } from '@sim/db/schema'`**: Imports `member` and `userStats` which are Drizzle schema objects (or "tables"). `member` likely represents users' memberships in organizations, and `userStats` contains user-specific statistics, including billing-related flags.

```typescript
import { and, eq } from 'drizzle-orm'
```
*   **`import { and, eq } from 'drizzle-orm'`**: Imports `and` and `eq` from `drizzle-orm`. These are Drizzle utility functions used to construct database query conditions:
    *   `eq`: Checks for equality (e.g., `column === value`).
    *   `and`: Combines multiple conditions with a logical AND.

```typescript
import { type NextRequest, NextResponse } from 'next/server'
```
*   **`import { type NextRequest, NextResponse } from 'next/server'`**: Imports `NextRequest` (for type checking the incoming request object) and `NextResponse` (for constructing responses) from Next.js, specific to API routes.

```typescript
import { getSession } from '@/lib/auth'
```
*   **`import { getSession } from '@/lib/auth'`**: Imports a helper function `getSession` from the application's authentication library. This function is responsible for retrieving the current user's session data, typically including their ID and other user information.

```typescript
import { getSimplifiedBillingSummary } from '@/lib/billing/core/billing'
```
*   **`import { getSimplifiedBillingSummary } from '@/lib/billing/core/billing'`**: Imports a function `getSimplifiedBillingSummary` which likely fetches a high-level, simplified billing overview for a given user.

```typescript
import { getOrganizationBillingData } from '@/lib/billing/core/organization'
```
*   **`import { getOrganizationBillingData } from '@/lib/billing/core/organization'`**: Imports a function `getOrganizationBillingData` which retrieves detailed billing information specific to an organization.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a utility function `createLogger` to create a new logger instance for logging messages (e.g., errors, warnings) to the console or other configured outputs.

```typescript
const logger = createLogger('UnifiedBillingAPI')
```
*   **`const logger = createLogger('UnifiedBillingAPI')`**: Initializes a logger instance specifically named 'UnifiedBillingAPI'. This helps in identifying logs originating from this particular API route.

```typescript
/**
 * Unified Billing Endpoint
 */
export async function GET(request: NextRequest) {
```
*   **`export async function GET(request: NextRequest)`**: This defines the main API route handler. In Next.js, exporting an `async` function named `GET` (or `POST`, `PUT`, `DELETE`) makes it the handler for HTTP GET requests to this route (`/api/billing` in this case). It receives a `NextRequest` object containing details about the incoming request.

```typescript
  const session = await getSession()
```
*   **`const session = await getSession()`**: Calls the `getSession` helper to retrieve the authentication session data for the current request. This will tell us if a user is logged in and provide their ID.

```typescript
  try {
```
*   **`try {`**: Starts a `try` block. This is standard practice for error handling, allowing potential errors within this block to be caught and handled gracefully by the subsequent `catch` block.

```typescript
    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
```
*   **`if (!session?.user?.id) { ... }`**: This is the first authentication check. If `session` is null or undefined, or `session.user` is missing, or `session.user.id` is not present, it means the user is not authenticated.
*   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`**: If unauthorized, it returns a JSON response with an `error` message and an HTTP `401 Unauthorized` status code.

```typescript
    const { searchParams } = new URL(request.url)
```
*   **`const { searchParams } = new URL(request.url)`**: Creates a `URL` object from the incoming request's URL. This provides easy access to URL components, particularly `searchParams` which contains query parameters (e.g., `?context=user&id=123`).

```typescript
    const context = searchParams.get('context') || 'user'
```
*   **`const context = searchParams.get('context') || 'user'`**: Retrieves the value of the `context` query parameter. If `context` is not provided in the URL, it defaults to `'user'`. This parameter dictates whether the API should fetch user-specific or organization-specific billing data.

```typescript
    const contextId = searchParams.get('id')
```
*   **`const contextId = searchParams.get('id')`**: Retrieves the value of the `id` query parameter. This parameter is expected to be an organization ID when `context` is `'organization'`.

```typescript
    // Validate context parameter
    if (!['user', 'organization'].includes(context)) {
      return NextResponse.json(
        { error: 'Invalid context. Must be "user" or "organization"' },
        { status: 400 }
      )
    }
```
*   **`if (!['user', 'organization'].includes(context)) { ... }`**: Validates the `context` parameter. If it's anything other than `'user'` or `'organization'`, it's considered an invalid request.
*   **`return NextResponse.json({ error: '...' }, { status: 400 })`**: Returns a JSON error response with an HTTP `400 Bad Request` status code, indicating an invalid parameter.

```typescript
    // For organization context, require contextId
    if (context === 'organization' && !contextId) {
      return NextResponse.json(
        { error: 'Organization ID is required when context=organization' },
        { status: 400 }
      )
    }
```
*   **`if (context === 'organization' && !contextId) { ... }`**: If the `context` is `'organization'`, this line checks if the `contextId` (the organization's ID) was provided. If `contextId` is missing, it's an invalid request.
*   **`return NextResponse.json({ error: '...' }, { status: 400 })`**: Returns a JSON error response with an HTTP `400 Bad Request` status code, explicitly stating that an organization ID is required.

```typescript
    let billingData
```
*   **`let billingData`**: Declares a variable `billingData` to store the billing information. Its type and structure will vary depending on whether `context` is `'user'` or `'organization'`.

```typescript
    if (context === 'user') {
```
*   **`if (context === 'user') {`**: This block executes if the request is for user-specific billing information.

```typescript
      // Get user billing (may include organization if they're part of one)
      billingData = await getSimplifiedBillingSummary(session.user.id, contextId || undefined)
```
*   **`billingData = await getSimplifiedBillingSummary(session.user.id, contextId || undefined)`**: Calls the `getSimplifiedBillingSummary` helper function. It passes the current user's ID (`session.user.id`) and optionally the `contextId`. The `contextId || undefined` part ensures that if `contextId` is null or empty, `undefined` is passed instead, which might be handled differently by the helper.

```typescript
      // Attach billingBlocked status for the current user
      const stats = await db
        .select({ blocked: userStats.billingBlocked })
        .from(userStats)
        .where(eq(userStats.userId, session.user.id))
        .limit(1)
```
*   **`const stats = await db ... limit(1)`**: This is a Drizzle ORM query:
    *   **`.select({ blocked: userStats.billingBlocked })`**: Selects only the `billingBlocked` column from the `userStats` table, aliasing it as `blocked` for convenience.
    *   **`.from(userStats)`**: Specifies that the query is against the `userStats` table.
    *   **`.where(eq(userStats.userId, session.user.id))`**: Filters the results to find the row where the `userId` matches the current user's ID.
    *   **`.limit(1)`**: Ensures that at most one record is returned, as `userId` should be unique in `userStats`.

```typescript
      billingData = {
        ...billingData,
        billingBlocked: stats.length > 0 ? !!stats[0].blocked : false,
      }
```
*   **`billingData = { ...billingData, billingBlocked: ... }`**: This line updates the `billingData` object.
    *   **`...billingData`**: Spreads all existing properties from the previously fetched `billingData`.
    *   **`billingBlocked: stats.length > 0 ? !!stats[0].blocked : false`**: Adds or updates a `billingBlocked` property.
        *   `stats.length > 0`: Checks if a `userStats` record was found for the user.
        *   `!!stats[0].blocked`: If a record was found, it takes the `blocked` value (which might be `0`, `1`, `true`, `false`, `null`, etc.) and converts it to a strict boolean (`true` or `false`). `!!` is a common JavaScript idiom for this conversion.
        *   `: false`: If no `userStats` record was found, `billingBlocked` defaults to `false`.

```typescript
    } else { // context === 'organization'
```
*   **`}` `else {`**: This block executes if the `context` is `'organization'`.

```typescript
      // Get user role in organization for permission checks first
      const memberRecord = await db
        .select({ role: member.role })
        .from(member)
        .where(and(eq(member.organizationId, contextId!), eq(member.userId, session.user.id)))
        .limit(1)
```
*   **`const memberRecord = await db ... limit(1)`**: This Drizzle ORM query checks if the current user is a member of the specified organization and retrieves their role:
    *   **`.select({ role: member.role })`**: Selects only the `role` column from the `member` table, aliasing it as `role`.
    *   **`.from(member)`**: Specifies the `member` table.
    *   **`.where(and(eq(member.organizationId, contextId!), eq(member.userId, session.user.id)))`**: Filters for a member record where:
        *   `organizationId` matches the `contextId` (the requested organization ID). `contextId!` asserts that `contextId` is definitely not null or undefined here due to prior validation.
        *   AND `userId` matches the current user's ID (`session.user.id`).
    *   **`.limit(1)`**: Limits the result to one record.

```typescript
      if (memberRecord.length === 0) {
        return NextResponse.json(
          { error: 'Access denied - not a member of this organization' },
          { status: 403 }
        )
      }
```
*   **`if (memberRecord.length === 0) { ... }`**: This is an authorization check. If no `memberRecord` is found for the current user in the requested organization, it means the user is not a member.
*   **`return NextResponse.json({ error: '...' }, { status: 403 })`**: Returns a JSON error response with an HTTP `403 Forbidden` status code, indicating access is denied.

```typescript
      // Get organization-specific billing
      const rawBillingData = await getOrganizationBillingData(contextId!)
```
*   **`const rawBillingData = await getOrganizationBillingData(contextId!)`**: Calls the `getOrganizationBillingData` helper function, passing the required `contextId` (organization ID) to fetch detailed organization billing information.

```typescript
      if (!rawBillingData) {
        return NextResponse.json(
          { error: 'Organization not found or access denied' },
          { status: 404 }
        )
      }
```
*   **`if (!rawBillingData) { ... }`**: Checks if `getOrganizationBillingData` returned any data. If not (e.g., organization ID didn't match an existing organization, or internal access issues within the helper), it implies the organization wasn't found or access failed.
*   **`return NextResponse.json({ error: '...' }, { status: 404 })`**: Returns a JSON error response with an HTTP `404 Not Found` status code.

```typescript
      // Transform data to match component expectations
      billingData = {
        organizationId: rawBillingData.organizationId,
        organizationName: rawBillingData.organizationName,
        subscriptionPlan: rawBillingData.subscriptionPlan,
        subscriptionStatus: rawBillingData.subscriptionStatus,
        totalSeats: rawBillingData.totalSeats,
        usedSeats: rawBillingData.usedSeats,
        seatsCount: rawBillingData.seatsCount,
        totalCurrentUsage: rawBillingData.totalCurrentUsage,
        totalUsageLimit: rawBillingData.totalUsageLimit,
        minimumBillingAmount: rawBillingData.minimumBillingAmount,
        averageUsagePerMember: rawBillingData.averageUsagePerMember,
        billingPeriodStart: rawBillingData.billingPeriodStart?.toISOString() || null,
        billingPeriodEnd: rawBillingData.billingPeriodEnd?.toISOString() || null,
        members: rawBillingData.members.map((member) => ({
          ...member,
          joinedAt: member.joinedAt.toISOString(),
          lastActive: member.lastActive?.toISOString() || null,
        })),
      }
```
*   **`billingData = { ... }`**: This block transforms the `rawBillingData` into a structured object (`billingData`) that is more suitable for client-side consumption.
    *   Many properties are directly copied from `rawBillingData`.
    *   **`billingPeriodStart: rawBillingData.billingPeriodStart?.toISOString() || null`**: Converts `billingPeriodStart` (which is likely a `Date` object) into an ISO 8601 string. The `?.` (optional chaining) handles cases where `billingPeriodStart` might be `null` or `undefined`, defaulting to `null` in such cases.
    *   **`billingPeriodEnd: rawBillingData.billingPeriodEnd?.toISOString() || null`**: Similar date conversion for `billingPeriodEnd`.
    *   **`members: rawBillingData.members.map((member) => ({ ... }))`**: Iterates over the `members` array within `rawBillingData`. For each `member` object, it creates a new object:
        *   `...member`: Spreads all original properties of the `member`.
        *   `joinedAt: member.joinedAt.toISOString()`: Converts `joinedAt` (a `Date`) to an ISO string.
        *   `lastActive: member.lastActive?.toISOString() || null`: Converts `lastActive` (an optional `Date`) to an ISO string, or `null` if it's not present.

```typescript
      const userRole = memberRecord[0].role
```
*   **`const userRole = memberRecord[0].role`**: Extracts the `role` of the current user within the organization from the `memberRecord` fetched earlier.

```typescript
      // Include the requesting user's blocked flag as well so UI can reflect it
      const stats = await db
        .select({ blocked: userStats.billingBlocked })
        .from(userStats)
        .where(eq(userStats.userId, session.user.id))
        .limit(1)
```
*   **`const stats = await db ... limit(1)`**: This is another Drizzle query, identical to the one in the `user` context. It fetches the `billingBlocked` status specifically for the *current logged-in user*, even when retrieving organization billing. This allows the UI to show a personalized warning or status to the user accessing the organization's billing.

```typescript
      // Merge blocked flag into data for convenience
      billingData = {
        ...billingData,
        billingBlocked: stats.length > 0 ? !!stats[0].blocked : false,
      }
```
*   **`billingData = { ...billingData, billingBlocked: ... }`**: Similar to the `user` context, this merges the current user's `billingBlocked` status into the `billingData` object (which currently holds organization-specific data).

```typescript
      return NextResponse.json({
        success: true,
        context,
        data: billingData,
        userRole,
        billingBlocked: billingData.billingBlocked,
      })
    }
```
*   **`return NextResponse.json({ success: true, ... })`**: This is the final successful response for the `'organization'` context. It returns:
    *   `success: true`: A flag indicating success.
    *   `context`: The requested context (`'organization'`).
    *   `data`: The `billingData` object (containing organization details and the user's `billingBlocked` status).
    *   `userRole`: The role of the requesting user in the organization.
    *   `billingBlocked`: The requesting user's `billingBlocked` status, also explicitly included at the top level for convenience.

```typescript
    return NextResponse.json({
      success: true,
      context,
      data: billingData,
    })
  } catch (error) {
```
*   **`return NextResponse.json({ success: true, context, data: billingData, })`**: This `return` statement is for the `'user'` context. If the `if (context === 'user')` block successfully completes, it will return this JSON response with `success`, `context`, and the user-specific `billingData`.
*   **`}` `catch (error) {`**: Catches any errors that occurred within the `try` block.

```typescript
    logger.error('Failed to get billing data', {
      userId: session?.user?.id,
      error,
    })
```
*   **`logger.error('Failed to get billing data', { userId: session?.user?.id, error, })`**: Logs the caught error using the `logger` instance. It includes a descriptive message, the `userId` (if available), and the `error` object itself, which is crucial for debugging.

```typescript
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```
*   **`return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`**: If an unexpected error occurs, it returns a generic JSON error message with an HTTP `500 Internal Server Error` status code. This prevents exposing sensitive internal error details to the client.