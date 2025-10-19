This TypeScript file provides a set of utility functions for handling internal authentication and authorization within a Next.js application, particularly for server-to-server communication and scheduled tasks (CRON jobs). It leverages JSON Web Tokens (JWTs) for internal API calls and a simple shared secret mechanism for CRON job authentication.

---

### **Purpose of this File**

This file acts as a central point for managing different types of authentication within your application's backend:

1.  **Generating Internal JWTs**: Creates short-lived, signed tokens that your own backend services can use to authenticate with each other, ensuring that only trusted internal components can access specific APIs.
2.  **Verifying Internal JWTs**: Validates these internal tokens to ensure they are legitimate, haven't been tampered with, and are used by the intended services.
3.  **Verifying CRON Job Authentication**: Provides a simple, effective way to secure API endpoints that are meant to be called by scheduled tasks (like CRON jobs) using a pre-shared secret, preventing unauthorized external access to these critical background processes.

In essence, it's a security layer to ensure that internal communications and automated tasks are authorized and secure.

---

### **Detailed Explanation**

Let's break down the code step by step:

```typescript
import { jwtVerify, SignJWT } from 'jose'
import { type NextRequest, NextResponse } from 'next/server'
import { env } from '@/lib/env'
import { createLogger } from '@/lib/logs/console/logger'
```

*   **`import { jwtVerify, SignJWT } from 'jose'`**: This line imports two key functions from the `jose` library (Javascript Object Signing and Encryption).
    *   `SignJWT`: Used to create and sign (generate) JSON Web Tokens.
    *   `jwtVerify`: Used to verify and decode JSON Web Tokens.
    *   `jose` is a modern, robust, and secure library for handling JWTs.
*   **`import { type NextRequest, NextResponse } from 'next/server'`**: These are Next.js-specific types and classes, essential for handling API routes in a Next.js environment.
    *   `NextRequest`: Represents an incoming HTTP request, providing access to headers, body, URL, etc.
    *   `NextResponse`: Used to construct and send an HTTP response.
*   **`import { env } from '@/lib/env'`**: This imports an `env` object from a local file (`@/lib/env`). This object likely holds environment variables (like API secrets) which are crucial for security.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a function to create a logger instance from a local logging utility.

---

```typescript
const logger = createLogger('CronAuth')
```

*   **`const logger = createLogger('CronAuth')`**: Initializes a logger specific to "CronAuth" operations. This allows you to categorize and filter log messages, making it easier to monitor security events related to CRON jobs and internal authentication.

---

```typescript
// Create a secret key for JWT signing
const getJwtSecret = () => {
  const secret = new TextEncoder().encode(env.INTERNAL_API_SECRET)
  return secret
}
```

*   **`const getJwtSecret = () => { ... }`**: This function is a helper to securely obtain the secret key used for signing and verifying JWTs.
*   **`const secret = new TextEncoder().encode(env.INTERNAL_API_SECRET)`**:
    *   `env.INTERNAL_API_SECRET`: Retrieves a secret string from your environment variables. This string should be a strong, randomly generated secret that is kept confidential.
    *   `new TextEncoder().encode(...)`: The `jose` library (and cryptographic operations in general) typically expects secrets to be in a byte array format (`Uint8Array`), not just a plain string. `TextEncoder().encode()` converts the string secret into its UTF-8 byte representation, which is required for cryptographic functions.
*   **`return secret`**: Returns the byte array representation of the secret.

---

```typescript
/**
 * Generate an internal JWT token for server-side API calls
 * Token expires in 5 minutes to keep it short-lived
 */
export async function generateInternalToken(): Promise<string> {
  const secret = getJwtSecret()

  const token = await new SignJWT({ type: 'internal' })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('5m')
    .setIssuer('sim-internal')
    .setAudience('sim-api')
    .sign(secret)

  return token
}
```

*   **`export async function generateInternalToken(): Promise<string>`**: This asynchronous function is responsible for creating a new internal JWT. It will return the signed token as a string.
*   **`const secret = getJwtSecret()`**: Calls the helper function to get the secret key for signing.
*   **`const token = await new SignJWT({ type: 'internal' }) ... .sign(secret)`**: This is the core logic for creating the JWT.
    *   **`new SignJWT({ type: 'internal' })`**: Initializes a new JWT with a payload. The payload is a JavaScript object `{ type: 'internal' }`, indicating that this token is specifically for internal use. You could add more data here if needed (e.g., `userId`, `permissions`).
    *   **`.setProtectedHeader({ alg: 'HS256' })`**: Sets the header of the JWT. `alg: 'HS256'` specifies the algorithm used for signing the token, which is HMAC SHA-256. This is a common and secure symmetric algorithm.
    *   **`.setIssuedAt()`**: Sets the `iat` (issued at) claim, which records the timestamp when the token was generated.
    *   **`.setExpirationTime('5m')`**: Sets the `exp` (expiration time) claim. The token will be valid for only 5 minutes. Short-lived tokens are a good security practice, as they reduce the window of opportunity for an attacker if a token is compromised.
    *   **`.setIssuer('sim-internal')`**: Sets the `iss` (issuer) claim. This identifies the entity that issued the JWT (e.g., "sim-internal" service).
    *   **`.setAudience('sim-api')`**: Sets the `aud` (audience) claim. This identifies the recipient(s) that the JWT is intended for (e.g., "sim-api" service).
    *   **`.sign(secret)`**: This is the final step where the token is cryptographically signed using the `secret` key. This signature ensures the token's authenticity and integrity (i.e., it hasn't been tampered with).
*   **`return token`**: Returns the complete, signed JWT string.

---

```typescript
/**
 * Verify an internal JWT token
 * Returns true if valid, false otherwise
 */
export async function verifyInternalToken(token: string): Promise<boolean> {
  try {
    const secret = getJwtSecret()

    const { payload } = await jwtVerify(token, secret, {
      issuer: 'sim-internal',
      audience: 'sim-api',
    })

    // Check that it's an internal token
    return payload.type === 'internal'
  } catch (error) {
    // Token verification failed
    return false
  }
}
```

*   **`export async function verifyInternalToken(token: string): Promise<boolean>`**: This asynchronous function takes a JWT string and attempts to verify it. It returns `true` if the token is valid, and `false` otherwise.
*   **`try { ... } catch (error) { ... }`**: This block uses a `try...catch` statement to handle potential errors during token verification. If anything goes wrong (e.g., invalid signature, expired token), the `catch` block will execute.
*   **`const secret = getJwtSecret()`**: Retrieves the same secret key used for signing, which is necessary for verification.
*   **`const { payload } = await jwtVerify(token, secret, { ... })`**:
    *   `jwtVerify(token, secret, { ... })`: This is the core verification function. It takes the token string, the secret key, and an options object.
    *   **`issuer: 'sim-internal'`**: Verifies that the token's `iss` claim matches 'sim-internal'. If it doesn't, verification fails.
    *   **`audience: 'sim-api'`**: Verifies that the token's `aud` claim matches 'sim-api'. If it doesn't, verification fails.
    *   If verification is successful (signature is valid, not expired, issuer/audience match), it returns an object containing the `payload` (the data we originally put in, like `{ type: 'internal' }`).
    *   `const { payload } = ...`: Destructures the `payload` from the returned object.
*   **`return payload.type === 'internal'`**: After successful cryptographic verification, this line adds an extra layer of check by ensuring the `type` property within the token's payload is indeed 'internal'. This prevents tokens with valid signatures but unexpected content from being accepted.
*   **`catch (error) { return false }`**: If any error occurs during `jwtVerify` (e.g., token is expired, tampered with, invalid issuer/audience), the `catch` block is triggered, and the function returns `false`, indicating the token is invalid.

---

```typescript
/**
 * Verify CRON authentication for scheduled API endpoints
 * Returns null if authorized, or a NextResponse with error if unauthorized
 */
export function verifyCronAuth(request: NextRequest, context?: string): NextResponse | null {
  const authHeader = request.headers.get('authorization')
  const expectedAuth = `Bearer ${env.CRON_SECRET}`
  if (authHeader !== expectedAuth) {
    const contextInfo = context ? ` for ${context}` : ''
    logger.warn(`Unauthorized CRON access attempt${contextInfo}`, {
      providedAuth: authHeader,
      ip: request.headers.get('x-forwarded-for') ?? request.headers.get('x-real-ip') ?? 'unknown',
      userAgent: request.headers.get('user-agent') ?? 'unknown',
      context,
    })

    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  return null
}
```

*   **`export function verifyCronAuth(request: NextRequest, context?: string): NextResponse | null`**: This synchronous function is designed to authenticate requests coming from CRON jobs. It takes a `NextRequest` object and an optional `context` string (for logging purposes). It returns either a `NextResponse` with an error if unauthorized, or `null` if the request is authorized.
*   **`const authHeader = request.headers.get('authorization')`**: Retrieves the value of the `Authorization` header from the incoming request. This is where CRON jobs are expected to send their secret.
*   **`const expectedAuth = \`Bearer ${env.CRON_SECRET}\``**: Constructs the expected authorization string. It uses a "Bearer" scheme followed by a secret retrieved from environment variables (`env.CRON_SECRET`). This `CRON_SECRET` is a pre-shared secret that both the CRON job scheduler and your API endpoint know.
*   **`if (authHeader !== expectedAuth) { ... }`**: This is the core authentication check. If the provided `Authorization` header does not exactly match the `expectedAuth` string, the request is deemed unauthorized.
    *   **`const contextInfo = context ? \` for ${context}\` : ''`**: Creates an optional string for logging, adding more context if provided (e.g., "for user data sync").
    *   **`logger.warn(...)`**: Logs a warning message when an unauthorized access attempt is detected. This is crucial for security monitoring. It includes:
        *   The actual `providedAuth` header (could be `null` or wrong).
        *   The `ip` address of the request (attempting to get it from `x-forwarded-for` or `x-real-ip` headers, which are common when behind proxies or load balancers).
        *   The `userAgent` string, if available.
        *   The `context` string.
    *   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`**: If unauthorized, it returns a `NextResponse` with a JSON body `{ error: 'Unauthorized' }` and an HTTP status code of `401 Unauthorized`. This response tells the client (the CRON job scheduler in this case) that their request was denied due to lack of valid authentication.
*   **`return null`**: If the `authHeader` matches `expectedAuth`, the `if` block is skipped, and the function returns `null`. This indicates that the request is authorized, and the calling API route can proceed with its logic.

---