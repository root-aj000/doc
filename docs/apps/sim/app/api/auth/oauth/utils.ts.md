This TypeScript file acts as a central utility for managing user identification, fetching user credentials, and, most importantly, handling the lifecycle of OAuth access tokens, including automatic refreshing when they expire. It's designed to be robust, secure, and easy to integrate into different parts of an application that relies on external services via OAuth.

---

### File Purpose and What It Does

At its core, this file provides a set of asynchronous functions to:

1.  **Identify Users:** Determine the current user's ID, whether the request originates from a client-side session or a server-side workflow.
2.  **Retrieve Credentials:** Fetch specific OAuth credentials (like an access token, refresh token, and expiry) associated with a user and an external service.
3.  **Manage OAuth Tokens:** This is the primary function. It ensures that when an external service requires an OAuth access token, the application always provides a valid, non-expired one. If a token is expired or missing, it attempts to automatically refresh it using a stored refresh token, saving the updated token information back into the database. This prevents service disruptions due to expired tokens.

In essence, it centralizes the logic for secure user identification and seamless OAuth token management, making it easier for other parts of the application to interact with external APIs without worrying about token expiry.

---

### Detailed Code Explanation

Let's break down each part of the file.

#### Imports

```typescript
import { db } from '@sim/db'
import { account, workflow } from '@sim/db/schema'
import { and, desc, eq } from 'drizzle-orm'
import { getSession } from '@/lib/auth'
import { createLogger } from '@/lib/logs/console/logger'
import { refreshOAuthToken } from '@/lib/oauth/oauth'
```

*   `import { db } from '@sim/db'`: This imports the database client instance, likely configured for a specific ORM (Object-Relational Mapper) like Drizzle ORM. `db` is what we use to interact with the database.
*   `import { account, workflow } from '@sim/db/schema'`: These imports bring in the schema definitions for the `account` and `workflow` tables. These definitions allow the ORM to understand the structure of your database tables and interact with them in a type-safe manner.
    *   `account`: Likely stores user credentials for external services (e.g., Google, Stripe, etc.), including OAuth tokens.
    *   `workflow`: Likely stores information about server-side processes or automated tasks, potentially linked to a `userId`.
*   `import { and, desc, eq } from 'drizzle-orm'`: These are utility functions from the Drizzle ORM library, used for building database queries:
    *   `and`: Combines multiple conditions with a logical AND.
    *   `desc`: Specifies descending order for sorting results.
    *   `eq`: Checks for equality (e.g., `column = value`).
*   `import { getSession } from '@/lib/auth'`: This imports a function responsible for retrieving the current user's session data. This is typically used in web applications to identify authenticated users from client-side requests.
*   `import { createLogger } from '@/lib/logs/console/logger'`: This imports a function to create a logger instance, which will be used throughout the file to record events, warnings, and errors for debugging and monitoring purposes.
*   `import { refreshOAuthToken } from '@/lib/oauth/oauth'`: This imports an external utility function that performs the actual OAuth token refresh operation by making a request to the OAuth provider's API. This file delegates the "how to refresh" to this utility.

#### Logger Setup

```typescript
const logger = createLogger('OAuthUtilsAPI')
```

*   `const logger = createLogger('OAuthUtilsAPI')`: An instance of the logger is created, specifically named `'OAuthUtilsAPI'`. This helps in tracing logs back to their source within the application.

---

### `getUserId` Function

```typescript
/**
 * Get the user ID based on either a session or a workflow ID
 */
export async function getUserId(
  requestId: string,
  workflowId?: string
): Promise<string | undefined> {
  // If workflowId is provided, this is a server-side request
  if (workflowId) {
    // Get the workflow to verify the user ID
    const workflows = await db
      .select({ userId: workflow.userId })
      .from(workflow)
      .where(eq(workflow.id, workflowId))
      .limit(1)

    if (!workflows.length) {
      logger.warn(`[${requestId}] Workflow not found`)
      return undefined
    }

    return workflows[0].userId
  }
  // This is a client-side request, use the session
  const session = await getSession()

  // Check if the user is authenticated
  if (!session?.user?.id) {
    logger.warn(`[${requestId}] Unauthenticated request rejected`)
    return undefined
  }

  return session.user.id
}
```

**Purpose:** This function is a flexible way to determine the ID of the user associated with an incoming request. It can handle two scenarios: server-side requests (identified by a `workflowId`) and client-side requests (identified by a user session).

**Parameters:**
*   `requestId: string`: A unique identifier for the current request, used for correlating logs.
*   `workflowId?: string`: An optional ID for a server-side workflow. If provided, it signifies a server-triggered action.

**Simplified Logic:**
If a `workflowId` is given, it looks up the workflow in the database to find the user associated with it. If no `workflowId` is given, it assumes a client-side request and tries to get the user's ID from the current session. In either case, if a user ID cannot be determined, it logs a warning and returns `undefined`.

**Line-by-Line Explanation:**

*   `export async function getUserId(...)`: Defines an asynchronous function `getUserId` that can be imported and used elsewhere. It returns a `Promise` that resolves to either a `string` (the user ID) or `undefined`.
*   `if (workflowId) { ... }`: This checks if a `workflowId` was provided.
    *   If `workflowId` is present, it indicates a server-side operation or an automated process.
    *   `const workflows = await db .select({ userId: workflow.userId }) .from(workflow) .where(eq(workflow.id, workflowId)) .limit(1)`: This is a Drizzle ORM query:
        *   `db.select({ userId: workflow.userId })`: Selects only the `userId` column from the `workflow` table.
        *   `.from(workflow)`: Specifies that we are querying the `workflow` table.
        *   `.where(eq(workflow.id, workflowId))`: Filters the results to find the workflow where the `id` column matches the provided `workflowId`.
        *   `.limit(1)`: Ensures that the query returns at most one result, as `id` should be unique.
        *   `await`: Waits for the database query to complete.
    *   `if (!workflows.length) { ... }`: Checks if any workflow was found. If `workflows` is an empty array (`!workflows.length` is true), it means no workflow matched the `workflowId`.
        *   `logger.warn(`[${requestId}] Workflow not found`)`: Logs a warning, including the `requestId` for context.
        *   `return undefined`: Returns `undefined` because the user ID couldn't be determined via the workflow.
    *   `return workflows[0].userId`: If a workflow was found, it returns the `userId` from the first (and only) result.
*   `// This is a client-side request, use the session`: This comment indicates the alternative path if `workflowId` was not provided.
*   `const session = await getSession()`: Calls the `getSession` function to retrieve the current user's session object. This typically contains authentication information.
*   `if (!session?.user?.id) { ... }`: This checks if the session exists, has a `user` object, and if that `user` object has an `id`. The `?` (optional chaining) safely accesses nested properties that might be `null` or `undefined`.
    *   If any of these conditions are false, it means the request is unauthenticated.
    *   `logger.warn(`[${requestId}] Unauthenticated request rejected`)`: Logs a warning for unauthenticated access.
    *   `return undefined`: Returns `undefined` as no authenticated user ID could be found.
*   `return session.user.id`: If an authenticated session with a user ID is found, it returns that user ID.

---

### `getCredential` Function

```typescript
/**
 * Get a credential by ID and verify it belongs to the user
 */
export async function getCredential(requestId: string, credentialId: string, userId: string) {
  const credentials = await db
    .select()
    .from(account)
    .where(and(eq(account.id, credentialId), eq(account.userId, userId)))
    .limit(1)

  if (!credentials.length) {
    logger.warn(`[${requestId}] Credential not found`)
    return undefined
  }

  return credentials[0]
}
```

**Purpose:** This function securely retrieves a specific OAuth credential (e.g., for Google, Stripe, etc.) from the `account` table, ensuring that the credential actually belongs to the specified `userId`. This prevents one user from accessing another user's credentials.

**Parameters:**
*   `requestId: string`: A unique identifier for the request, used for logging.
*   `credentialId: string`: The specific ID of the credential record to retrieve.
*   `userId: string`: The ID of the user who is expected to own this credential.

**Simplified Logic:**
It queries the `account` table for a credential matching both the provided `credentialId` AND the `userId`. If found, it returns the credential; otherwise, it logs a warning and returns `undefined`.

**Line-by-Line Explanation:**

*   `export async function getCredential(...)`: Defines an asynchronous function `getCredential`. It returns a `Promise` that resolves to either the credential object (from the `account` table) or `undefined`.
*   `const credentials = await db .select() .from(account) .where(and(eq(account.id, credentialId), eq(account.userId, userId))) .limit(1)`: This is a Drizzle ORM query:
    *   `db.select()`: Selects all columns (`*`) from the table.
    *   `.from(account)`: Specifies that we are querying the `account` table, where credentials are stored.
    *   `.where(and(eq(account.id, credentialId), eq(account.userId, userId)))`: This is the crucial security part. It filters the results based on two conditions:
        *   `eq(account.id, credentialId)`: The credential's ID must match the `credentialId` provided.
        *   `eq(account.userId, userId)`: AND the credential's `userId` must match the `userId` provided. This prevents users from retrieving credentials they don't own.
        *   `and(...)`: Combines these two conditions, meaning both must be true for a credential to be returned.
    *   `.limit(1)`: Ensures that at most one credential is returned, as `id` should be unique.
    *   `await`: Waits for the database query to complete.
*   `if (!credentials.length) { ... }`: Checks if any credential was found.
    *   `logger.warn(`[${requestId}] Credential not found`)`: Logs a warning if no matching credential is found.
    *   `return undefined`: Returns `undefined`.
*   `return credentials[0]`: If a credential is found, it returns the first (and only) result from the array.

---

### `getOAuthToken` Function

```typescript
export async function getOAuthToken(userId: string, providerId: string): Promise<string | null> {
  const connections = await db
    .select({
      id: account.id,
      accessToken: account.accessToken,
      refreshToken: account.refreshToken,
      accessTokenExpiresAt: account.accessTokenExpiresAt,
    })
    .from(account)
    .where(and(eq(account.userId, userId), eq(account.providerId, providerId)))
    // Always use the most recently updated credential for this provider
    .orderBy(desc(account.updatedAt))
    .limit(1)

  if (connections.length === 0) {
    logger.warn(`No OAuth token found for user ${userId}, provider ${providerId}`)
    return null
  }

  const credential = connections[0]

  // Determine whether we should refresh: missing token OR expired token
  const now = new Date()
  const tokenExpiry = credential.accessTokenExpiresAt
  const shouldAttemptRefresh =
    !!credential.refreshToken && (!credential.accessToken || (tokenExpiry && tokenExpiry < now))

  if (shouldAttemptRefresh) {
    logger.info(
      `Access token expired for user ${userId}, provider ${providerId}. Attempting to refresh.`
    )

    try {
      // Use the existing refreshOAuthToken function
      const refreshResult = await refreshOAuthToken(providerId, credential.refreshToken!)

      if (!refreshResult) {
        logger.error(`Failed to refresh token for user ${userId}, provider ${providerId}`, {
          providerId,
          userId,
          hasRefreshToken: !!credential.refreshToken,
        })
        return null
      }

      const { accessToken, expiresIn, refreshToken: newRefreshToken } = refreshResult

      // Update the database with new tokens
      const updateData: any = {
        accessToken,
        accessTokenExpiresAt: new Date(Date.now() + expiresIn * 1000), // Convert seconds to milliseconds
        updatedAt: new Date(),
      }

      // If we received a new refresh token (some providers like Airtable rotate them), save it
      if (newRefreshToken && newRefreshToken !== credential.refreshToken) {
        logger.info(`Updating refresh token for user ${userId}, provider ${providerId}`)
        updateData.refreshToken = newRefreshToken
      }

      // Update the token in the database with the actual expiration time from the provider
      await db.update(account).set(updateData).where(eq(account.id, credential.id))

      logger.info(`Successfully refreshed token for user ${userId}, provider ${providerId}`)
      return accessToken
    } catch (error) {
      logger.error(`Error refreshing token for user ${userId}, provider ${providerId}`, {
        error: error instanceof Error ? error.message : String(error),
        stack: error instanceof Error ? error.stack : undefined,
        providerId,
        userId,
      })
      return null
    }
  }

  if (!credential.accessToken) {
    logger.warn(
      `Access token is null and no refresh attempted or available for user ${userId}, provider ${providerId}`
    )
    return null
  }

  logger.info(`Found valid OAuth token for user ${userId}, provider ${providerId}`)
  return credential.accessToken
}
```

**Purpose:** This is a crucial function for OAuth integration. Its main goal is to provide a valid access token for a given user and OAuth provider. It intelligently checks if the stored token is still valid (not expired or missing) and, if not, automatically attempts to refresh it using the stored refresh token.

**Parameters:**
*   `userId: string`: The ID of the user whose OAuth token is needed.
*   `providerId: string`: The ID of the OAuth provider (e.g., 'google', 'stripe') for which the token is required.

**Simplified Logic:**
1.  **Fetch Credential:** It first retrieves the user's latest credential for the specified provider from the database.
2.  **Check for Refresh:** It then determines if the access token needs to be refreshed. This happens if there's no access token, or if it has expired, AND a refresh token is available.
3.  **Perform Refresh (if needed):** If a refresh is necessary, it calls an external `refreshOAuthToken` utility. If successful, it updates the database with the new access token, its expiry, and potentially a new refresh token.
4.  **Handle Failures:** If the refresh fails or if no valid token (and no means to refresh) is found, it logs an error/warning and returns `null`.
5.  **Return Token:** Otherwise, it returns the valid (either existing or newly refreshed) access token.

**Line-by-Line Explanation:**

*   `export async function getOAuthToken(...)`: Defines an asynchronous function `getOAuthToken` that returns a `Promise<string | null>`, meaning it will return the access token string or `null` if it can't be obtained.
*   `const connections = await db .select({ ... }) .from(account) .where(and(eq(account.userId, userId), eq(account.providerId, providerId))) .orderBy(desc(account.updatedAt)) .limit(1)`: This query retrieves the relevant credential.
    *   `db.select({ id: account.id, accessToken: account.accessToken, refreshToken: account.refreshToken, accessTokenExpiresAt: account.accessTokenExpiresAt })`: Selects specific columns needed for token management: the credential `id`, `accessToken`, `refreshToken`, and `accessTokenExpiresAt`.
    *   `.from(account)`: Queries the `account` table.
    *   `.where(and(eq(account.userId, userId), eq(account.providerId, providerId)))`: Filters to find credentials belonging to the `userId` for the specific `providerId`.
    *   `.orderBy(desc(account.updatedAt))`: Orders the results by `updatedAt` in descending order. This is important to ensure we get the *most recently updated* credential if a user has multiple for the same provider (e.g., in edge cases).
    *   `.limit(1)`: Gets only the top (most recent) credential.
*   `if (connections.length === 0) { ... }`: If no matching credential (connection) is found in the database.
    *   `logger.warn(...)`: Logs a warning.
    *   `return null`: Returns `null` as no token can be provided.
*   `const credential = connections[0]`: Stores the retrieved credential object for easier access.
*   `const now = new Date()`: Gets the current date and time.
*   `const tokenExpiry = credential.accessTokenExpiresAt`: Retrieves the stored expiration time of the access token.
*   `const shouldAttemptRefresh = !!credential.refreshToken && (!credential.accessToken || (tokenExpiry && tokenExpiry < now))`: This is the core logic for deciding whether to refresh:
    *   `!!credential.refreshToken`: Checks if a `refreshToken` actually exists (it's essential for refreshing).
    *   `!credential.accessToken`: Checks if the `accessToken` itself is missing.
    *   `(tokenExpiry && tokenExpiry < now)`: Checks if an `accessTokenExpiresAt` is present AND if it's in the past (i.e., the token has expired).
    *   The `||` means refresh if the token is missing OR it's expired. The outer `&&` means we only consider refreshing if a `refreshToken` is available.
*   `if (shouldAttemptRefresh) { ... }`: If the conditions for refreshing are met:
    *   `logger.info(...)`: Logs that a refresh attempt is being made.
    *   `try { ... } catch (error) { ... }`: A `try-catch` block is used to handle potential errors during the refresh process.
        *   `const refreshResult = await refreshOAuthToken(providerId, credential.refreshToken!)`: Calls the external `refreshOAuthToken` function. `credential.refreshToken!` uses a non-null assertion because `shouldAttemptRefresh` already verified its existence.
        *   `if (!refreshResult) { ... }`: If `refreshOAuthToken` returns `null` or `undefined` (indicating a failure).
            *   `logger.error(...)`: Logs an error.
            *   `return null`: Returns `null` due to refresh failure.
        *   `const { accessToken, expiresIn, refreshToken: newRefreshToken } = refreshResult`: Destructures the successful refresh result, getting the new `accessToken`, how long it's `expiresIn` (in seconds), and potentially a `newRefreshToken`.
        *   `const updateData: any = { ... }`: Prepares an object with the data to update in the database:
            *   `accessToken`: The newly obtained access token.
            *   `accessTokenExpiresAt`: Calculates the new expiry time by adding `expiresIn` seconds to the current time (`Date.now() + expiresIn * 1000`).
            *   `updatedAt`: Sets the `updatedAt` timestamp to the current time.
        *   `if (newRefreshToken && newRefreshToken !== credential.refreshToken) { ... }`: Some OAuth providers rotate refresh tokens. This checks if a `newRefreshToken` was provided and if it's different from the old one.
            *   If so, `updateData.refreshToken = newRefreshToken` adds the new refresh token to the update payload.
        *   `await db.update(account).set(updateData).where(eq(account.id, credential.id))`: Updates the `account` record in the database with the new token information.
        *   `logger.info(...)`: Logs success.
        *   `return accessToken`: Returns the newly refreshed access token.
        *   `catch (error) { ... }`: If an error occurs during the refresh process.
            *   `logger.error(...)`: Logs the error with details.
            *   `return null`: Returns `null` because the refresh failed.
*   `if (!credential.accessToken) { ... }`: This `if` block is reached if `shouldAttemptRefresh` was false, but there's still no `accessToken`. This can happen if there's no refresh token available, or if the initial check found no `accessToken` but `tokenExpiry` was also `null` (meaning no expiry was ever set) or the token was technically expired but no refresh token was present.
    *   `logger.warn(...)`: Logs a warning.
    *   `return null`: Returns `null` as no valid token is available.
*   `logger.info(...)`: If `shouldAttemptRefresh` was false and `credential.accessToken` exists (meaning it's a valid, non-expired token), this logs that a valid token was found.
*   `return credential.accessToken`: Returns the existing, valid access token.

---

### `refreshAccessTokenIfNeeded` Function

```typescript
/**
 * Refreshes an OAuth token if needed based on credential information
 * @param credentialId The ID of the credential to check and potentially refresh
 * @param userId The user ID who owns the credential (for security verification)
 * @param requestId Request ID for log correlation
 * @returns The valid access token or null if refresh fails
 */
export async function refreshAccessTokenIfNeeded(
  credentialId: string,
  userId: string,
  requestId: string
): Promise<string | null> {
  // Get the credential directly using the getCredential helper
  const credential = await getCredential(requestId, credentialId, userId)

  if (!credential) {
    return null
  }

  // Decide if we should refresh: token missing OR expired
  const expiresAt = credential.accessTokenExpiresAt
  const now = new Date()
  const shouldRefresh =
    !!credential.refreshToken && (!credential.accessToken || (expiresAt && expiresAt <= now))

  const accessToken = credential.accessToken

  if (shouldRefresh) {
    logger.info(`[${requestId}] Token expired, attempting to refresh for credential`)
    try {
      const refreshedToken = await refreshOAuthToken(
        credential.providerId,
        credential.refreshToken!
      )

      if (!refreshedToken) {
        logger.error(`[${requestId}] Failed to refresh token for credential: ${credentialId}`, {
          credentialId,
          providerId: credential.providerId,
          userId: credential.userId,
          hasRefreshToken: !!credential.refreshToken,
        })
        return null
      }

      // Prepare update data
      const updateData: any = {
        accessToken: refreshedToken.accessToken,
        accessTokenExpiresAt: new Date(Date.now() + refreshedToken.expiresIn * 1000),
        updatedAt: new Date(),
      }

      // If we received a new refresh token, update it
      if (refreshedToken.refreshToken && refreshedToken.refreshToken !== credential.refreshToken) {
        logger.info(`[${requestId}] Updating refresh token for credential`)
        updateData.refreshToken = refreshedToken.refreshToken
      }

      // Update the token in the database
      await db.update(account).set(updateData).where(eq(account.id, credentialId))

      logger.info(`[${requestId}] Successfully refreshed access token for credential`)
      return refreshedToken.accessToken
    } catch (error) {
      logger.error(`[${requestId}] Error refreshing token for credential`, {
        error: error instanceof Error ? error.message : String(error),
        stack: error instanceof Error ? error.stack : undefined,
        providerId: credential.providerId,
        credentialId,
        userId: credential.userId,
      })
      return null
    }
  } else if (!accessToken) {
    // We have no access token and either no refresh token or not eligible to refresh
    logger.error(`[${requestId}] Missing access token for credential`)
    return null
  }

  logger.info(`[${requestId}] Access token is valid for credential`)
  return accessToken
}
```

**Purpose:** This function is very similar to `getOAuthToken`, but it takes a `credentialId` directly instead of a `providerId`. It's designed to refresh a specific, known credential if it's expired or missing.

**Parameters:**
*   `credentialId: string`: The ID of the specific credential record to operate on.
*   `userId: string`: The ID of the user owning the credential (for security verification).
*   `requestId: string`: Request ID for log correlation.

**Simplified Logic:**
1.  It first uses `getCredential` to securely fetch the credential details, ensuring it belongs to the `userId`.
2.  Then, it applies the same logic as `getOAuthToken`: check if a refresh is needed (expired or missing access token, with a refresh token present).
3.  If needed, it attempts to refresh the token, updates the database, and returns the new token.
4.  If not needed, it returns the existing valid token.
5.  Handles errors by logging and returning `null`.

**Line-by-Line Explanation:**

*   `export async function refreshAccessTokenIfNeeded(...)`: Defines the asynchronous function.
*   `const credential = await getCredential(requestId, credentialId, userId)`: Calls the helper `getCredential` function to fetch the credential, ensuring it belongs to the `userId`.
*   `if (!credential) { return null }`: If `getCredential` couldn't find or verify the credential, this function immediately returns `null`.
*   `const expiresAt = credential.accessTokenExpiresAt; const now = new Date()`: Gets the expiration time of the token and the current time.
*   `const shouldRefresh = !!credential.refreshToken && (!credential.accessToken || (expiresAt && expiresAt <= now))`: Identical logic to `getOAuthToken` for determining if a refresh is necessary.
*   `const accessToken = credential.accessToken`: Stores the current `accessToken` for later use.
*   `if (shouldRefresh) { ... }`: If a refresh is needed, this block executes.
    *   `logger.info(...)`: Logs the refresh attempt.
    *   `try { ... } catch (error) { ... }`: Error handling for the refresh process.
        *   `const refreshedToken = await refreshOAuthToken(credential.providerId, credential.refreshToken!)`: Calls the external refresh utility.
        *   `if (!refreshedToken) { ... }`: Handles failed refresh. Logs an error and returns `null`.
        *   `const updateData: any = { ... }`: Prepares data for updating the database with new `accessToken`, `accessTokenExpiresAt`, and `updatedAt`.
        *   `if (refreshedToken.refreshToken && refreshedToken.refreshToken !== credential.refreshToken) { ... }`: Checks and updates for a rotated refresh token.
        *   `await db.update(account).set(updateData).where(eq(account.id, credentialId))`: Updates the database.
        *   `logger.info(...)`: Logs success.
        *   `return refreshedToken.accessToken`: Returns the newly refreshed token.
        *   `catch (error) { ... }`: Logs and handles errors during refresh.
*   `else if (!accessToken) { ... }`: This `else if` is hit if `shouldRefresh` was false (meaning no refresh was attempted), but the `accessToken` is still missing. This indicates a problem.
    *   `logger.error(...)`: Logs an error about a missing access token without a refresh path.
    *   `return null`: Returns `null`.
*   `logger.info(...)`: If `shouldRefresh` was false and `accessToken` was present, it means the existing token is valid.
*   `return accessToken`: Returns the existing, valid access token.

---

### `refreshTokenIfNeeded` Function (Enhanced Version)

```typescript
/**
 * Enhanced version that returns additional information about the refresh operation
 */
export async function refreshTokenIfNeeded(
  requestId: string,
  credential: any, // This is expected to be a full credential object already fetched
  credentialId: string
): Promise<{ accessToken: string; refreshed: boolean }> {
  // Decide if we should refresh: token missing OR expired
  const expiresAt = credential.accessTokenExpiresAt
  const now = new Date()
  const shouldRefresh =
    !!credential.refreshToken && (!credential.accessToken || (expiresAt && expiresAt <= now))

  // If token appears valid and present, return it directly
  if (!shouldRefresh) {
    logger.info(`[${requestId}] Access token is valid`)
    return { accessToken: credential.accessToken, refreshed: false }
  }

  try {
    const refreshResult = await refreshOAuthToken(credential.providerId, credential.refreshToken!)

    if (!refreshResult) {
      logger.error(`[${requestId}] Failed to refresh token for credential`)
      throw new Error('Failed to refresh token')
    }

    const { accessToken: refreshedToken, expiresIn, refreshToken: newRefreshToken } = refreshResult

    // Prepare update data
    const updateData: any = {
      accessToken: refreshedToken,
      accessTokenExpiresAt: new Date(Date.now() + expiresIn * 1000), // Use provider's expiry
      updatedAt: new Date(),
    }

    // If we received a new refresh token, update it
    if (newRefreshToken && newRefreshToken !== credential.refreshToken) {
      logger.info(`[${requestId}] Updating refresh token`)
      updateData.refreshToken = newRefreshToken
    }

    await db.update(account).set(updateData).where(eq(account.id, credentialId))

    logger.info(`[${requestId}] Successfully refreshed access token`)
    return { accessToken: refreshedToken, refreshed: true }
  } catch (error) {
    logger.warn(
      `[${requestId}] Refresh attempt failed, checking if another concurrent request succeeded`
    )

    const freshCredential = await getCredential(requestId, credentialId, credential.userId)
    if (freshCredential?.accessToken) {
      const freshExpiresAt = freshCredential.accessTokenExpiresAt
      const stillValid = !freshExpiresAt || freshExpiresAt > new Date()

      if (stillValid) {
        logger.info(`[${requestId}] Found valid token from concurrent refresh, using it`)
        return { accessToken: freshCredential.accessToken, refreshed: true }
      }
    }

    logger.error(`[${requestId}] Refresh failed and no valid token found in DB`, error)
    throw error
  }
}
```

**Purpose:** This is an "enhanced" version of the token refresh logic. Unlike the previous functions, it expects a `credential` object to be passed in (meaning it's already fetched). Its key difference is its return type (providing a `refreshed` flag) and its more sophisticated error handling, specifically designed to mitigate issues with *concurrent refresh attempts*.

**Parameters:**
*   `requestId: string`: Request ID for log correlation.
*   `credential: any`: The full credential object (expected to be already retrieved from the database). Using `any` here implies the caller is responsible for providing a properly shaped object, or there's an implicit type from Drizzle that's not imported directly here.
*   `credentialId: string`: The ID of the credential, used for database updates.

**Simplified Logic:**
1.  **Check for Refresh:** It first checks if the provided `credential`'s access token is expired or missing, and if a refresh token is present.
2.  **Return Existing:** If no refresh is needed, it immediately returns the existing access token along with `refreshed: false`.
3.  **Perform Refresh:** If a refresh is needed, it attempts to call `refreshOAuthToken`.
4.  **Update Database:** If successful, it updates the database and returns the new token with `refreshed: true`.
5.  **Concurrent Refresh Handling:** If the refresh *fails* (e.g., due to a network error or the refresh token being invalid), it performs an extra step: it re-fetches the credential from the database. This is to check if another *concurrent* request might have successfully refreshed the token in the meantime. If a fresh, valid token is found in the DB, it uses that.
6.  **Throw Error:** If the initial refresh failed and no concurrent refresh was successful, it throws an error, indicating a definitive failure to obtain a valid token.

**Line-by-Line Explanation:**

*   `export async function refreshTokenIfNeeded(...)`: Defines the asynchronous function. It returns a `Promise<{ accessToken: string; refreshed: boolean }>`.
*   `const expiresAt = credential.accessTokenExpiresAt; const now = new Date()`: Gets expiry time and current time.
*   `const shouldRefresh = !!credential.refreshToken && (!credential.accessToken || (expiresAt && expiresAt <= now))`: Identical refresh logic as previous functions.
*   `if (!shouldRefresh) { ... }`: If no refresh is required:
    *   `logger.info(...)`: Logs that the token is valid.
    *   `return { accessToken: credential.accessToken, refreshed: false }`: Returns the existing token and `refreshed: false`.
*   `try { ... } catch (error) { ... }`: Standard `try-catch` block for refresh attempt.
    *   `const refreshResult = await refreshOAuthToken(credential.providerId, credential.refreshToken!)`: Calls the external refresh utility.
    *   `if (!refreshResult) { ... }`: If `refreshOAuthToken` fails:
        *   `logger.error(...)`: Logs the error.
        *   `throw new Error('Failed to refresh token')`: Instead of returning `null`, this function *throws an error*, indicating a hard failure that needs to be handled by the caller.
    *   `const { accessToken: refreshedToken, expiresIn, refreshToken: newRefreshToken } = refreshResult`: Destructures successful refresh result.
    *   `const updateData: any = { ... }`: Prepares update data.
    *   `if (newRefreshToken && newRefreshToken !== credential.refreshToken) { ... }`: Checks and updates for rotated refresh token.
    *   `await db.update(account).set(updateData).where(eq(account.id, credentialId))`: Updates the database.
    *   `logger.info(...)`: Logs success.
    *   `return { accessToken: refreshedToken, refreshed: true }`: Returns the new token and `refreshed: true`.
*   `catch (error) { ... }`: This `catch` block handles errors from the `try` block, including the `Error` thrown internally if `refreshResult` is `null`.
    *   `logger.warn(...)`: Logs that the refresh attempt failed, and it's now checking for concurrent successes. This is the unique part of this function.
    *   `const freshCredential = await getCredential(requestId, credentialId, credential.userId)`: It immediately tries to re-fetch the credential from the database. The idea is that if this function failed to refresh, maybe another instance (e.g., another serverless function, another thread) tried and succeeded *while this one was attempting its refresh*.
    *   `if (freshCredential?.accessToken) { ... }`: If the re-fetched credential exists and has an `accessToken`.
        *   `const freshExpiresAt = freshCredential.accessTokenExpiresAt; const stillValid = !freshExpiresAt || freshExpiresAt > new Date()`: Checks if this newly fetched token is actually valid (not expired).
        *   `if (stillValid) { ... }`: If the re-fetched token is valid:
            *   `logger.info(...)`: Logs that a valid token was found from a concurrent refresh.
            *   `return { accessToken: freshCredential.accessToken, refreshed: true }`: Returns this concurrently refreshed token. The `refreshed: true` indicates that *a* refresh (even if not this specific call's attempt) did happen and resulted in a new valid token.
    *   `logger.error(...)`: If the initial refresh failed AND the re-fetch from the database also didn't yield a valid token, then it's a definitive failure.
    *   `throw error`: Re-throws the original error, propagating the failure up the call stack.

This comprehensive explanation covers the purpose, simplified logic, and a deep line-by-line breakdown for each part of the provided TypeScript code, fulfilling all the requirements.