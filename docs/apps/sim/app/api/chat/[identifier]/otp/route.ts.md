This TypeScript file defines the API endpoints for managing One-Time Password (OTP) based authentication for chat deployments within a Next.js application. It specifically handles two main operations: sending an OTP to a user's email and verifying a user-provided OTP.

---

### **Detailed Explanation**

#### **Purpose of this file and what it does**

This file (`app/api/chat/[identifier]/otp/route.ts`) acts as the backend for handling OTP-based access to specific chat deployments. Imagine you have a chat service where some chats are protected, requiring users to enter a code sent to their email to gain access. This file implements that mechanism.

It exposes two HTTP endpoints:

1.  **`POST /api/chat/[identifier]/otp`**: This endpoint is used by a user (or their browser) to *request* an OTP. The user provides their email address, and if their email is authorized for the specified chat, a unique 6-digit code (OTP) is generated, stored temporarily, and sent to their email.
2.  **`PUT /api/chat/[identifier]/otp`**: This endpoint is used to *verify* an OTP. After receiving the OTP via email, the user submits their email and the received code. The system then checks if the provided OTP matches the stored one. If it matches, the user is considered authenticated for that chat, and an authentication cookie is set in their browser.

The file uses a combination of a database (Drizzle ORM) to look up chat details, Redis (with an in-memory fallback) to store temporary OTPs, Zod for robust input validation, and an email service to send the OTPs.

---

#### **Line-by-Line Explanation**

```typescript
import { db } from '@sim/db'
import { chat } from '@sim/db/schema'
import { eq } from 'drizzle-orm'
import type { NextRequest } from 'next/server'
import { z } from 'zod'
import { renderOTPEmail } from '@/components/emails/render-email'
import { sendEmail } from '@/lib/email/mailer'
import { createLogger } from '@/lib/logs/console/logger'
import { getRedisClient, markMessageAsProcessed, releaseLock } from '@/lib/redis'
import { generateRequestId } from '@/lib/utils'
import { addCorsHeaders, setChatAuthCookie } from '@/app/api/chat/utils'
import { createErrorResponse, createSuccessResponse } from '@/app/api/workflows/utils'
```
*   **`import { db } from '@sim/db'`**: Imports the database client instance, likely configured with Drizzle ORM, for interacting with the database.
*   **`import { chat } from '@sim/db/schema'`**: Imports the Drizzle schema definition for the `chat` table, allowing typed database queries related to chat deployments.
*   **`import { eq } from 'drizzle-orm'`**: Imports the `eq` (equals) function from Drizzle ORM, used for constructing equality conditions in database queries (e.g., `WHERE column = value`).
*   **`import type { NextRequest } from 'next/server'`**: Imports the `NextRequest` type from Next.js, which represents an incoming HTTP request in Next.js API routes. This is used for type safety when defining the API handlers.
*   **`import { z } from 'zod'`**: Imports the `z` object from the Zod library, a TypeScript-first schema declaration and validation library. It's used here to define and validate the structure and types of incoming request bodies.
*   **`import { renderOTPEmail } from '@/components/emails/render-email'`**: Imports a function responsible for generating the HTML content of the OTP email. This likely uses a templating engine to create a nicely formatted email.
*   **`import { sendEmail } from '@/lib/email/mailer'`**: Imports a utility function for sending emails. This function likely wraps an underlying email service (e.g., Nodemailer, SendGrid, Mailgun).
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a factory function to create a logger instance, used for logging messages (debug, info, warn, error) to the console or other logging destinations.
*   **`import { getRedisClient, markMessageAsProcessed, releaseLock } from '@/lib/redis'`**: Imports Redis-related utilities:
    *   `getRedisClient()`: Retrieves an instance of the Redis client if Redis is configured and available.
    *   `markMessageAsProcessed()`: A generic function primarily used to set a key in Redis (or in-memory cache) with a value and expiry, often signaling that a process has been handled. It's repurposed here for OTP storage.
    *   `releaseLock()`: A generic function used to delete a key, often used to release a lock in a distributed system. Here, it's used to delete OTP entries.
*   **`import { generateRequestId } from '@/lib/utils'`**: Imports a utility function to generate a unique request ID. This ID is useful for tracing requests through logs.
*   **`import { addCorsHeaders, setChatAuthCookie } from '@/app/api/chat/utils'`**: Imports utility functions specific to chat APIs:
    *   `addCorsHeaders()`: Adds Cross-Origin Resource Sharing (CORS) headers to a `Response` object, allowing requests from different origins.
    *   `setChatAuthCookie()`: Sets an authentication cookie in the response, signaling successful authentication for the chat.
*   **`import { createErrorResponse, createSuccessResponse } from '@/app/api/workflows/utils'`**: Imports utility functions for creating standardized API responses:
    *   `createErrorResponse()`: Generates a JSON response for errors.
    *   `createSuccessResponse()`: Generates a JSON response for successful operations.

```typescript
const logger = createLogger('ChatOtpAPI')
```
*   **`const logger = createLogger('ChatOtpAPI')`**: Initializes a logger instance specifically named 'ChatOtpAPI' for this file. This helps in identifying log messages originating from this part of the application.

```typescript
function generateOTP() {
  return Math.floor(100000 + Math.random() * 900000).toString()
}
```
*   **`function generateOTP()`**: This simple function generates a 6-digit one-time password.
    *   `Math.random()`: Produces a floating-point number between 0 (inclusive) and 1 (exclusive).
    *   `Math.random() * 900000`: Scales this number to be between 0 and 900,000.
    *   `100000 + ...`: Adds 100,000 to ensure the number is always between 100,000 and 999,999 (inclusive), making it a 6-digit number.
    *   `Math.floor(...)`: Rounds the number down to the nearest whole integer.
    *   `.toString()`: Converts the number to a string, which is the desired format for an OTP.

#### **OTP Storage Utility Functions (Redis & Fallback)**

This section defines functions to store, retrieve, and delete OTPs. It's designed to prioritize Redis for performance and scalability, but includes a clever *in-memory fallback* mechanism using `markMessageAsProcessed` and a direct `inMemoryCache` access for environments where Redis might not be available (e.g., local development or specific deployment configurations).

```typescript
// OTP storage utility functions using Redis
// We use 15 minutes (900 seconds) expiry for OTPs
const OTP_EXPIRY = 15 * 60
```
*   **`const OTP_EXPIRY = 15 * 60`**: Defines a constant for the OTP expiry time in seconds. Here, it's set to 900 seconds, which is 15 minutes. This ensures OTPs are temporary and cannot be used indefinitely.

```typescript
// Store OTP in Redis
async function storeOTP(email: string, chatId: string, otp: string): Promise<void> {
  const key = `otp:${email}:${chatId}`
  const redis = getRedisClient()

  if (redis) {
    // Use Redis if available
    await redis.set(key, otp, 'EX', OTP_EXPIRY)
  } else {
    // Use the existing function as fallback to mark that an OTP exists
    await markMessageAsProcessed(key, OTP_EXPIRY)

    // For the fallback case, we need to handle storing the OTP value separately
    // since markMessageAsProcessed only stores "1"
    const valueKey = `${key}:value`
    try {
      // Access the in-memory cache directly - hacky but works for fallback
      const inMemoryCache = (global as any).inMemoryCache
      if (inMemoryCache) {
        const fullKey = `processed:${valueKey}`
        const expiry = OTP_EXPIRY ? Date.now() + OTP_EXPIRY * 1000 : null
        inMemoryCache.set(fullKey, { value: otp, expiry })
      }
    } catch (error) {
      logger.error('Error storing OTP in fallback cache:', error)
    }
  }
}
```
*   **`async function storeOTP(email: string, chatId: string, otp: string): Promise<void>`**: An asynchronous function to store an OTP.
    *   `email: string`, `chatId: string`, `otp: string`: Parameters representing the user's email, the ID of the chat, and the generated OTP.
    *   `Promise<void>`: Indicates the function does not return a value, but its operations are asynchronous.
*   **`const key = `otp:${email}:${chatId}``**: Constructs a unique key for storing the OTP. This key combines "otp", the user's email, and the chat ID, ensuring OTPs are specific to a user and a chat.
*   **`const redis = getRedisClient()`**: Attempts to get a Redis client instance.
*   **`if (redis)`**: Checks if a Redis client was successfully obtained.
    *   **`await redis.set(key, otp, 'EX', OTP_EXPIRY)`**: If Redis is available, it stores the `otp` string in Redis under the `key`. `'EX'` is a Redis command argument that sets an expiry for the key in seconds, using the `OTP_EXPIRY` constant.
*   **`else { ... }`**: If Redis is not available, the fallback mechanism is used.
    *   **`await markMessageAsProcessed(key, OTP_EXPIRY)`**: This function is typically used to mark that a message has been processed, storing a generic "1" under the given key with an expiry. Here, it's repurposed to simply indicate that *an* OTP exists for this `key`.
    *   **`const valueKey = `${key}:value``**: Since `markMessageAsProcessed` only stores a generic "1", a separate key (`valueKey`) is created to store the *actual* OTP string in the fallback.
    *   **`try { ... } catch (error) { ... }`**: A `try-catch` block to handle potential errors during fallback cache access.
    *   **`const inMemoryCache = (global as any).inMemoryCache`**: This is the "hacky" part. It attempts to access a global in-memory cache object. `(global as any)` is used to bypass TypeScript's type checking for the global scope, assuming `inMemoryCache` is attached there. This is common in development or simple environments where a full Redis instance isn't guaranteed.
    *   **`if (inMemoryCache)`**: Checks if the in-memory cache exists.
    *   **`const fullKey = `processed:${valueKey}``**: Creates a specific key for the in-memory cache, prefixed with `processed:` to align with `markMessageAsProcessed`'s internal workings.
    *   **`const expiry = OTP_EXPIRY ? Date.now() + OTP_EXPIRY * 1000 : null`**: Calculates the absolute expiry timestamp for the in-memory cache entry (current time + `OTP_EXPIRY` in milliseconds).
    *   **`inMemoryCache.set(fullKey, { value: otp, expiry })`**: Stores an object containing the `otp` and its `expiry` in the in-memory cache.
    *   **`logger.error(...)`**: Logs any errors encountered during the fallback storage.

```typescript
// Get OTP from Redis
async function getOTP(email: string, chatId: string): Promise<string | null> {
  const key = `otp:${email}:${chatId}`
  const redis = getRedisClient()

  if (redis) {
    // Use Redis if available
    return await redis.get(key)
  }
  // Use the existing function as fallback - check if it exists
  const exists = await new Promise((resolve) => {
    try {
      // Check the in-memory cache directly - hacky but works for fallback
      const inMemoryCache = (global as any).inMemoryCache
      const fullKey = `processed:${key}`
      const cacheEntry = inMemoryCache?.get(fullKey)
      resolve(!!cacheEntry)
    } catch {
      resolve(false)
    }
  })

  if (!exists) return null

  // Try to get the value key
  const valueKey = `${key}:value`
  try {
    const inMemoryCache = (global as any).inMemoryCache
    const fullKey = `processed:${valueKey}`
    const cacheEntry = inMemoryCache?.get(fullKey)
    return cacheEntry?.value || null
  } catch {
    return null
  }
}
```
*   **`async function getOTP(email: string, chatId: string): Promise<string | null>`**: An asynchronous function to retrieve an OTP.
    *   `Promise<string | null>`: Indicates it returns the OTP string if found, otherwise `null`.
*   **`const key = `otp:${email}:${chatId}``**: Reconstructs the unique key for the OTP.
*   **`const redis = getRedisClient()`**: Attempts to get a Redis client.
*   **`if (redis)`**: If Redis is available.
    *   **`return await redis.get(key)`**: Retrieves the value associated with the `key` from Redis. If the key doesn't exist or has expired, Redis returns `null`.
*   **`else { ... }`**: If Redis is not available, uses the fallback.
    *   **`const exists = await new Promise((resolve) => { ... })`**: This block checks if an OTP *exists* in the in-memory fallback cache for the given `key`.
        *   It uses a `new Promise` to wrap the synchronous access to `inMemoryCache`, potentially to mimic the async nature of Redis or for error handling.
        *   **`const inMemoryCache = (global as any).inMemoryCache`**: Accesses the global in-memory cache.
        *   **`const fullKey = `processed:${key}``**: Forms the key for `markMessageAsProcessed`'s entry.
        *   **`const cacheEntry = inMemoryCache?.get(fullKey)`**: Checks if this "existence" entry is in the cache.
        *   **`resolve(!!cacheEntry)`**: Resolves the promise with `true` if `cacheEntry` exists (and is not null/undefined), `false` otherwise.
    *   **`if (!exists) return null`**: If the "existence" entry isn't found, no OTP is present in the fallback, so return `null`.
    *   **`const valueKey = `${key}:value``**: Reconstructs the `valueKey` used to store the actual OTP string in the fallback.
    *   **`try { ... } catch { ... }`**: Another `try-catch` for robustness when accessing the in-memory cache.
    *   **`const inMemoryCache = (global as any).inMemoryCache`**: Accesses the global in-memory cache again.
    *   **`const fullKey = `processed:${valueKey}``**: Forms the key for the actual OTP value.
    *   **`const cacheEntry = inMemoryCache?.get(fullKey)`**: Retrieves the stored object from the cache.
    *   **`return cacheEntry?.value || null`**: Returns the `value` property from the cached object, or `null` if not found or an error occurs.

```typescript
// Delete OTP from Redis
async function deleteOTP(email: string, chatId: string): Promise<void> {
  const key = `otp:${email}:${chatId}`
  const redis = getRedisClient()

  if (redis) {
    // Use Redis if available
    await redis.del(key)
  } else {
    // Use the existing function as fallback
    await releaseLock(`processed:${key}`)
    await releaseLock(`processed:${key}:value`)
  }
}
```
*   **`async function deleteOTP(email: string, chatId: string): Promise<void>`**: An asynchronous function to delete an OTP.
*   **`const key = `otp:${email}:${chatId}``**: Reconstructs the unique key.
*   **`const redis = getRedisClient()`**: Attempts to get a Redis client.
*   **`if (redis)`**: If Redis is available.
    *   **`await redis.del(key)`**: Deletes the key (and its associated OTP) from Redis.
*   **`else { ... }`**: If Redis is not available, uses the fallback.
    *   **`await releaseLock(`processed:${key}`) `**: Uses `releaseLock` (which typically deletes keys) to remove the "existence" entry for the OTP from the in-memory cache. The `processed:` prefix is crucial for matching how `markMessageAsProcessed` stored it.
    *   **`await releaseLock(`processed:${key}:value`) `**: Deletes the entry containing the actual OTP value from the in-memory cache.

#### **Zod Schemas for Input Validation**

```typescript
const otpRequestSchema = z.object({
  email: z.string().email('Invalid email address'),
})

const otpVerifySchema = z.object({
  email: z.string().email('Invalid email address'),
  otp: z.string().length(6, 'OTP must be 6 digits'),
})
```
*   **`const otpRequestSchema = z.object({ ... })`**: Defines a Zod schema for the request body when requesting an OTP.
    *   `email: z.string().email('Invalid email address')`: Specifies that the `email` property must be a string and a valid email format. If not, it returns the custom error message.
*   **`const otpVerifySchema = z.object({ ... })`**: Defines a Zod schema for the request body when verifying an OTP.
    *   `email: z.string().email('Invalid email address')`: Same as above.
    *   `otp: z.string().length(6, 'OTP must be 6 digits')`: Specifies that the `otp` property must be a string and exactly 6 characters long.

#### **`POST` Endpoint: Send OTP**

This function handles HTTP POST requests to `/api/chat/[identifier]/otp`. Its job is to generate an OTP and send it to the user's email, provided they are authorized for the chat.

```typescript
export async function POST(
  request: NextRequest,
  { params }: { params: Promise<{ identifier: string }> }
) {
  const { identifier } = await params
  const requestId = generateRequestId()

  try {
    logger.debug(`[${requestId}] Processing OTP request for identifier: ${identifier}`)

    // Parse request body
    let body
    try {
      body = await request.json()
      const { email } = otpRequestSchema.parse(body)

      // Find the chat deployment
      const deploymentResult = await db
        .select({
          id: chat.id,
          authType: chat.authType,
          allowedEmails: chat.allowedEmails,
          title: chat.title,
        })
        .from(chat)
        .where(eq(chat.identifier, identifier))
        .limit(1)

      if (deploymentResult.length === 0) {
        logger.warn(`[${requestId}] Chat not found for identifier: ${identifier}`)
        return addCorsHeaders(createErrorResponse('Chat not found', 404), request)
      }

      const deployment = deploymentResult[0]

      // Verify this is an email-protected chat
      if (deployment.authType !== 'email') {
        return addCorsHeaders(
          createErrorResponse('This chat does not use email authentication', 400),
          request
        )
      }

      const allowedEmails: string[] = Array.isArray(deployment.allowedEmails)
        ? deployment.allowedEmails
        : []

      const isEmailAllowed =
        allowedEmails.includes(email) ||
        allowedEmails.some((allowed: string) => {
          if (allowed.startsWith('@')) {
            const domain = email.split('@')[1]
            return domain && allowed === `@${domain}`
          }
          return false
        })

      if (!isEmailAllowed) {
        return addCorsHeaders(
          createErrorResponse('Email not authorized for this chat', 403),
          request
        )
      }

      const otp = generateOTP()

      await storeOTP(email, deployment.id, otp)

      const emailHtml = await renderOTPEmail(
        otp,
        email,
        'email-verification',
        deployment.title || 'Chat'
      )

      const emailResult = await sendEmail({
        to: email,
        subject: `Verification code for ${deployment.title || 'Chat'}`,
        html: emailHtml,
      })

      if (!emailResult.success) {
        logger.error(`[${requestId}] Failed to send OTP email:`, emailResult.message)
        return addCorsHeaders(
          createErrorResponse('Failed to send verification email', 500),
          request
        )
      }

      // Add a small delay to ensure Redis has fully processed the operation
      // This helps with eventual consistency in distributed systems
      await new Promise((resolve) => setTimeout(resolve, 500))

      logger.info(`[${requestId}] OTP sent to ${email} for chat ${deployment.id}`)
      return addCorsHeaders(createSuccessResponse({ message: 'Verification code sent' }), request)
    } catch (error: any) {
      if (error instanceof z.ZodError) {
        return addCorsHeaders(
          createErrorResponse(error.errors[0]?.message || 'Invalid request', 400),
          request
        )
      }
      throw error
    }
  } catch (error: any) {
    logger.error(`[${requestId}] Error processing OTP request:`, error)
    return addCorsHeaders(
      createErrorResponse(error.message || 'Failed to process request', 500),
      request
    )
  }
}
```
*   **`export async function POST(...)`**: Defines the Next.js API route handler for POST requests.
    *   `request: NextRequest`: The incoming HTTP request object.
    *   `{ params }: { params: Promise<{ identifier: string }> }`: Next.js 13+ automatically parses dynamic segments from the URL (`[identifier]`) into the `params` object. It's destructured here, and the type `Promise<{ identifier: string }>` reflects how Next.js might pass these parameters.
*   **`const { identifier } = await params`**: Extracts the `identifier` (unique ID for the chat) from the URL parameters.
*   **`const requestId = generateRequestId()`**: Generates a unique ID for this specific request for logging purposes.
*   **`try { ... } catch (error: any) { ... }`**: An outer `try-catch` block for general error handling that wraps the entire request processing. If any unhandled error occurs, it logs it and returns a generic 500 error.
    *   **`logger.debug(...)`**: Logs the start of the request processing.
    *   **`let body; try { body = await request.json(); const { email } = otpRequestSchema.parse(body) } catch (error: any) { ... }`**: An inner `try-catch` block specifically for parsing the request body and validating it with Zod.
        *   `body = await request.json()`: Attempts to parse the incoming request body as JSON.
        *   `const { email } = otpRequestSchema.parse(body)`: Validates the parsed `body` against `otpRequestSchema`. If validation fails, a `ZodError` is thrown.
        *   **`if (error instanceof z.ZodError)`**: Catches validation errors.
        *   **`return addCorsHeaders(createErrorResponse(error.errors[0]?.message || 'Invalid request', 400), request)`**: Returns a 400 Bad Request error with a specific validation message (or a generic one). `addCorsHeaders` is called to ensure the response is accessible from different origins.
        *   **`throw error`**: If the error is not a `ZodError`, it's re-thrown to be caught by the outer `try-catch`.
    *   **Database Lookup:**
        *   **`const deploymentResult = await db.select(...).from(chat).where(eq(chat.identifier, identifier)).limit(1)`**: Queries the database using Drizzle ORM to find the chat deployment matching the `identifier`. It selects `id`, `authType`, `allowedEmails`, and `title`.
        *   **`if (deploymentResult.length === 0)`**: If no chat deployment is found.
        *   **`logger.warn(...)`**: Logs a warning.
        *   **`return addCorsHeaders(createErrorResponse('Chat not found', 404), request)`**: Returns a 404 Not Found error.
        *   **`const deployment = deploymentResult[0]`**: Extracts the first (and only) result.
    *   **Authentication Type Check:**
        *   **`if (deployment.authType !== 'email')`**: Checks if the chat is configured for email authentication.
        *   **`return addCorsHeaders(createErrorResponse('This chat does not use email authentication', 400), request)`**: Returns a 400 Bad Request if it's not an email-protected chat.
    *   **Email Authorization Check:**
        *   **`const allowedEmails: string[] = Array.isArray(deployment.allowedEmails) ? deployment.allowedEmails : []`**: Ensures `allowedEmails` is an array of strings, handling cases where it might be null or undefined.
        *   **`const isEmailAllowed = allowedEmails.includes(email) || allowedEmails.some(...)`**: Checks if the requested `email` is authorized.
            *   `allowedEmails.includes(email)`: Checks for an exact email match.
            *   `allowedEmails.some(...)`: Checks for domain-based authorization (e.g., if `@example.com` is in `allowedEmails`, then any email from `example.com` is allowed).
        *   **`if (!isEmailAllowed)`**: If the email is not authorized.
        *   **`return addCorsHeaders(createErrorResponse('Email not authorized for this chat', 403), request)`**: Returns a 403 Forbidden error.
    *   **OTP Generation and Storage:**
        *   **`const otp = generateOTP()`**: Generates the 6-digit OTP.
        *   **`await storeOTP(email, deployment.id, otp)`**: Stores the generated OTP using the Redis/fallback mechanism.
    *   **Email Sending:**
        *   **`const emailHtml = await renderOTPEmail(otp, email, 'email-verification', deployment.title || 'Chat')`**: Renders the HTML content for the email, injecting the OTP and chat title.
        *   **`const emailResult = await sendEmail({ ... })`**: Sends the email using the configured mailer.
        *   **`if (!emailResult.success)`**: If sending the email fails.
        *   **`logger.error(...)`**: Logs the email sending failure.
        *   **`return addCorsHeaders(createErrorResponse('Failed to send verification email', 500), request)`**: Returns a 500 Internal Server Error.
    *   **Delay for Consistency:**
        *   **`await new Promise((resolve) => setTimeout(resolve, 500))`**: Introduces a 500ms delay. This is a common practice in distributed systems that use eventually consistent caches (like Redis) to give the cache time to fully process the `storeOTP` operation before a subsequent `getOTP` call (though not strictly necessary for *this* flow, it can help prevent race conditions in more complex distributed scenarios where the OTP might be immediately checked by another service).
    *   **Success Response:**
        *   **`logger.info(...)`**: Logs that the OTP was successfully sent.
        *   **`return addCorsHeaders(createSuccessResponse({ message: 'Verification code sent' }), request)`**: Returns a 200 OK success response, indicating the email has been sent.

#### **`PUT` Endpoint: Verify OTP**

This function handles HTTP PUT requests to `/api/chat/[identifier]/otp`. Its purpose is to validate the OTP provided by the user against the stored OTP and, if successful, authenticate the user.

```typescript
export async function PUT(
  request: NextRequest,
  { params }: { params: Promise<{ identifier: string }> }
) {
  const { identifier } = await params
  const requestId = generateRequestId()

  try {
    logger.debug(`[${requestId}] Verifying OTP for identifier: ${identifier}`)

    // Parse request body
    let body
    try {
      body = await request.json()
      const { email, otp } = otpVerifySchema.parse(body)

      // Find the chat deployment
      const deploymentResult = await db
        .select({
          id: chat.id,
          authType: chat.authType,
        })
        .from(chat)
        .where(eq(chat.identifier, identifier))
        .limit(1)

      if (deploymentResult.length === 0) {
        logger.warn(`[${requestId}] Chat not found for identifier: ${identifier}`)
        return addCorsHeaders(createErrorResponse('Chat not found', 404), request)
      }

      const deployment = deploymentResult[0]

      // Check if OTP exists and is valid
      const storedOTP = await getOTP(email, deployment.id)
      if (!storedOTP) {
        return addCorsHeaders(
          createErrorResponse('No verification code found, request a new one', 400),
          request
        )
      }

      // Check if OTP matches
      if (storedOTP !== otp) {
        return addCorsHeaders(createErrorResponse('Invalid verification code', 400), request)
      }

      // OTP is valid, clean up
      await deleteOTP(email, deployment.id)

      // Create success response with auth cookie
      const response = addCorsHeaders(createSuccessResponse({ authenticated: true }), request)

      // Set authentication cookie
      setChatAuthCookie(response, deployment.id, deployment.authType)

      return response
    } catch (error: any) {
      if (error instanceof z.ZodError) {
        return addCorsHeaders(
          createErrorResponse(error.errors[0]?.message || 'Invalid request', 400),
          request
        )
      }
      throw error
    }
  } catch (error: any) {
    logger.error(`[${requestId}] Error verifying OTP:`, error)
    return addCorsHeaders(
      createErrorResponse(error.message || 'Failed to process request', 500),
      request
    )
  }
}
```
*   **`export async function PUT(...)`**: Defines the Next.js API route handler for PUT requests. Parameters are identical to the `POST` function.
*   **`const { identifier } = await params`**: Extracts the chat `identifier`.
*   **`const requestId = generateRequestId()`**: Generates a request ID.
*   **`try { ... } catch (error: any) { ... }`**: Outer `try-catch` for general error handling, similar to the `POST` endpoint.
    *   **`logger.debug(...)`**: Logs the start of OTP verification.
    *   **`let body; try { body = await request.json(); const { email, otp } = otpVerifySchema.parse(body) } catch (error: any) { ... }`**: Inner `try-catch` for parsing and validating the request body using `otpVerifySchema`, which expects both `email` and `otp`. Error handling is the same as in `POST`.
    *   **Database Lookup:**
        *   **`const deploymentResult = await db.select(...).from(chat).where(eq(chat.identifier, identifier)).limit(1)`**: Queries the database to find the chat deployment. This time, it only needs `id` and `authType`.
        *   **`if (deploymentResult.length === 0)`**: Handles chat not found, returning a 404 error.
        *   **`const deployment = deploymentResult[0]`**: Extracts the chat deployment details.
    *   **OTP Existence Check:**
        *   **`const storedOTP = await getOTP(email, deployment.id)`**: Retrieves the stored OTP using the `getOTP` function.
        *   **`if (!storedOTP)`**: If no OTP is found (it either never existed or has expired).
        *   **`return addCorsHeaders(createErrorResponse('No verification code found, request a new one', 400), request)`**: Returns a 400 Bad Request error.
    *   **OTP Match Check:**
        *   **`if (storedOTP !== otp)`**: Compares the provided `otp` with the `storedOTP`.
        *   **`return addCorsHeaders(createErrorResponse('Invalid verification code', 400), request)`**: Returns a 400 Bad Request error if they don't match.
    *   **Cleanup and Authentication:**
        *   **`await deleteOTP(email, deployment.id)`**: If the OTP is valid, it's deleted from storage to prevent reuse.
        *   **`const response = addCorsHeaders(createSuccessResponse({ authenticated: true }), request)`**: Creates a successful response indicating authentication, and applies CORS headers.
        *   **`setChatAuthCookie(response, deployment.id, deployment.authType)`**: Critically, this function sets an authentication cookie in the browser via the response headers. This cookie will be used by the frontend to confirm the user's access to the chat.
        *   **`return response`**: Returns the final response with the authentication cookie.