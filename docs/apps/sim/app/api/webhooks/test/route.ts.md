This TypeScript file defines a Next.js API route designed to help users test the configuration of various webhook providers within the application. It acts as a diagnostic tool, allowing a user to send a GET request with a webhook ID, and the API will attempt to verify or simulate an event for that specific webhook based on its configured provider (e.g., WhatsApp, Telegram, GitHub, Stripe, etc.).

---

### **Detailed Explanation**

#### 1. Purpose of this file and what it does

**Purpose:**
The primary purpose of this file is to provide a server-side endpoint (`/api/webhooks/test`) that allows for on-demand testing of configured webhooks. When a user configures a webhook (e.g., for WhatsApp, Telegram, GitHub), they often need to ensure that the webhook URL and its associated settings (like tokens, secrets) are correctly set up and that the application can receive and process events. This API route facilitates that verification process.

**What it does:**
1.  **Receives a Request:** It accepts a GET request containing a `webhookId` as a query parameter.
2.  **Fetches Webhook Data:** It queries the database to retrieve the full configuration details for the specified `webhookId`.
3.  **Identifies Provider:** Based on the `provider` field in the webhook's configuration, it determines which external service the webhook is for (e.g., 'whatsapp', 'telegram', 'github').
4.  **Performs Provider-Specific Tests:**
    *   For some providers (like WhatsApp, Telegram), it actively makes an outbound HTTP request to a *simulated* version of the webhook URL (or to the provider's API itself) to check for a valid response.
    *   For other providers (like GitHub, Stripe, generic), it primarily validates the presence of required configuration fields and then provides clear instructions or `curl` commands for how the user can manually test the webhook.
5.  **Logs and Responds:** It logs the testing process and its outcomes, then returns a JSON response indicating whether the test was successful, along with relevant diagnostic information or setup instructions.
6.  **Error Handling:** It includes robust error handling to catch issues during database access or external API calls.

#### 2. Simplified Complex Logic

The most complex part of this file is the `switch (provider)` statement. This structure handles the different testing methodologies required for each webhook provider. Instead of one universal test, the code branches out:

*   **WhatsApp:** Simulates WhatsApp's "verification" handshake. When WhatsApp sets up a webhook, it sends a GET request to verify the URL. This code mimics that by constructing a verification URL for the application's *own* webhook trigger endpoint and expecting a specific challenge string back.
*   **Telegram:** Simulates an incoming message from Telegram by sending a POST request with a sample payload to the application's webhook trigger endpoint. It also attempts to query Telegram's `getWebhookInfo` API to retrieve the current status of the webhook directly from Telegram.
*   **GitHub, Stripe, Slack, Airtable, Microsoft Teams, Generic:** For these, a direct "ping" test from the server isn't always the most effective or standard way to test. Instead, the code:
    *   Validates essential configuration fields (e.g., `signingSecret` for Slack).
    *   Constructs a `curl` command that the user can copy-paste and run themselves, showing exactly how the external service would call their webhook with a sample payload.
    *   Provides clear instructions on where to configure the webhook in the respective service's dashboard.

Essentially, the `switch` statement makes sure that each provider's unique requirements for setup and testing are addressed appropriately, either through an automated check or by guiding the user.

#### 3. Line-by-line Explanation

```typescript
import { db } from '@sim/db'
```
*   **`import { db } from '@sim/db'`**: Imports the Drizzle ORM database instance from a local `@sim/db` module. This `db` object is used to interact with the database.

```typescript
import { webhook } from '@sim/db/schema'
```
*   **`import { webhook } from '@sim/db/schema'`**: Imports the `webhook` table schema definition. This schema object is used with Drizzle ORM to specify which table to query and how its columns are structured.

```typescript
import { eq } from 'drizzle-orm'
```
*   **`import { eq } from 'drizzle-orm'`**: Imports the `eq` function from the `drizzle-orm` library. `eq` is a Drizzle-specific operator used to define an equality condition in a database query's `WHERE` clause (e.g., `WHERE id = 'some_id'`).

```typescript
import { type NextRequest, NextResponse } from 'next/server'
```
*   **`import { type NextRequest, NextResponse } from 'next/server'`**: Imports types and classes from Next.js server-side utilities.
    *   `NextRequest`: A type representing the incoming HTTP request object in a Next.js API route. It extends the standard Web `Request` API.
    *   `NextResponse`: A class used to create and send HTTP responses from a Next.js API route. It extends the standard Web `Response` API.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a custom utility function `createLogger` from the application's logging library. This function is used to instantiate a logger for specific modules or contexts, often adding metadata like the module name to log messages.

```typescript
import { getBaseUrl } from '@/lib/urls/utils'
```
*   **`import { getBaseUrl } from '@/lib/urls/utils'`**: Imports a custom utility function `getBaseUrl`. This function likely retrieves the base URL of the running application, which is crucial for constructing absolute URLs, especially when dealing with webhooks that need to know their public-facing address.

```typescript
import { generateRequestId } from '@/lib/utils'
```
*   **`import { generateRequestId } from '@/lib/utils'`**: Imports a custom utility function `generateRequestId`. This function generates a unique identifier for each incoming request, which is extremely useful for tracing requests through logs, especially in distributed systems.

---

```typescript
const logger = createLogger('WebhookTestAPI')
```
*   **`const logger = createLogger('WebhookTestAPI')`**: Initializes a logger instance specifically for this API route, labeling its output with 'WebhookTestAPI'. This helps in filtering and understanding log messages originating from this file.

---

```typescript
export const dynamic = 'force-dynamic'
```
*   **`export const dynamic = 'force-dynamic'`**: A Next.js-specific configuration option. Setting it to `'force-dynamic'` ensures that this API route is always executed on every request, bypassing any caching mechanisms (like static rendering or edge caching). This is important for API routes that need to perform real-time database queries or external API calls and should not serve stale data.

---

```typescript
export async function GET(request: NextRequest) {
```
*   **`export async function GET(request: NextRequest)`**: Defines the main asynchronous handler for HTTP GET requests to this API route. In Next.js, functions named after HTTP methods (GET, POST, PUT, DELETE) in an `api` directory file automatically become the handlers for those methods. `request: NextRequest` indicates that the function receives a `NextRequest` object as its argument.

---

```typescript
  const requestId = generateRequestId()
```
*   **`const requestId = generateRequestId()`**: Calls the `generateRequestId` utility to create a unique identifier for the current request. This `requestId` will be used in log messages to correlate all actions related to this specific incoming request.

---

```typescript
  try {
```
*   **`try {`**: Starts a `try` block, which is used for error handling. Any errors that occur within this block will be caught by the subsequent `catch` block.

---

```typescript
    const { searchParams } = new URL(request.url)
```
*   **`const { searchParams } = new URL(request.url)`**:
    *   `new URL(request.url)`: Creates a `URL` object from the incoming request's URL string. This object provides convenient methods for parsing URL components.
    *   `{ searchParams } = ...`: Destructures the `searchParams` property from the `URL` object. `searchParams` is a `URLSearchParams` object that makes it easy to work with query parameters (e.g., `?id=123`).

```typescript
    const webhookId = searchParams.get('id')
```
*   **`const webhookId = searchParams.get('id')`**: Retrieves the value of the `id` query parameter from the `searchParams`. This `id` is expected to be the identifier of the webhook to be tested.

```typescript
    if (!webhookId) {
```
*   **`if (!webhookId) {`**: Checks if the `webhookId` was provided in the query parameters. If it's `null` or an empty string, it means the ID is missing.

```typescript
      logger.warn(`[${requestId}] Missing webhook ID in test request`)
```
*   **`logger.warn(...)`**: Logs a warning message indicating that the `webhookId` is missing, including the `requestId` for tracing.

```typescript
      return NextResponse.json({ success: false, error: 'Webhook ID is required' }, { status: 400 })
```
*   **`return NextResponse.json(...)`**: Sends an HTTP response back to the client.
    *   `{ success: false, error: 'Webhook ID is required' }`: The JSON body of the response, indicating failure and the reason.
    *   `{ status: 400 }`: Sets the HTTP status code to 400 (Bad Request), signifying that the client sent an invalid request.

---

```typescript
    logger.debug(`[${requestId}] Testing webhook with ID: ${webhookId}`)
```
*   **`logger.debug(...)`**: Logs a debug message confirming that the testing process has started for the given `webhookId`.

---

```typescript
    const webhooks = await db.select().from(webhook).where(eq(webhook.id, webhookId)).limit(1)
```
*   **`const webhooks = await db.select().from(webhook).where(eq(webhook.id, webhookId)).limit(1)`**: Performs a database query using Drizzle ORM.
    *   `db.select()`: Starts a select query.
    *   `.from(webhook)`: Specifies that the query should be against the `webhook` table.
    *   `.where(eq(webhook.id, webhookId))`: Filters the results to find a webhook where the `id` column matches the `webhookId` obtained from the request.
    *   `.limit(1)`: Restricts the query to return at most one record, as `webhookId` is expected to be unique.
    *   `await`: Waits for the database query to complete.

```typescript
    if (webhooks.length === 0) {
```
*   **`if (webhooks.length === 0) {`**: Checks if the database query returned any results. If `webhooks` is an empty array, no webhook with the given ID was found.

```typescript
      logger.warn(`[${requestId}] Webhook not found: ${webhookId}`)
```
*   **`logger.warn(...)`**: Logs a warning message indicating that the webhook was not found in the database.

```typescript
      return NextResponse.json({ success: false, error: 'Webhook not found' }, { status: 404 })
```
*   **`return NextResponse.json(...)`**: Sends a JSON response with `success: false` and an error message. The HTTP status code is set to 404 (Not Found), indicating that the requested resource (the webhook) could not be found.

---

```typescript
    const foundWebhook = webhooks[0]
```
*   **`const foundWebhook = webhooks[0]`**: Since `limit(1)` was used, if a webhook was found, it will be the first (and only) element in the `webhooks` array. This line extracts that webhook object into the `foundWebhook` variable.

```typescript
    const provider = foundWebhook.provider || 'generic'
```
*   **`const provider = foundWebhook.provider || 'generic'`**: Retrieves the `provider` field from the `foundWebhook` object. If `foundWebhook.provider` is `null` or `undefined` (or any falsy value), it defaults to `'generic'`. This `provider` string determines which specific testing logic will be executed.

```typescript
    const providerConfig = (foundWebhook.providerConfig as Record<string, any>) || {}
```
*   **`const providerConfig = (foundWebhook.providerConfig as Record<string, any>) || {}`**: Retrieves the `providerConfig` field from the `foundWebhook`.
    *   `as Record<string, any>`: This is a TypeScript type assertion, telling the compiler that `providerConfig` is expected to be an object where keys are strings and values can be any type. This is common for JSONB fields in databases that store flexible configurations.
    *   `|| {}`: If `foundWebhook.providerConfig` is `null` or `undefined`, it defaults to an empty object `{}` to prevent errors when trying to access properties of `providerConfig` later.

```typescript
    const webhookUrl = `${getBaseUrl()}/api/webhooks/trigger/${foundWebhook.path}`
```
*   **`const webhookUrl = \`${getBaseUrl()}/api/webhooks/trigger/${foundWebhook.path}\``**: Constructs the full, absolute URL that external services (like WhatsApp, Telegram) would call to trigger this specific webhook.
    *   `` `${getBaseUrl()}` ``: Calls the `getBaseUrl()` utility to get the base URL of the application (e.g., `https://example.com`).
    *   `/api/webhooks/trigger/${foundWebhook.path}`: Appends the path to the webhook trigger endpoint. This suggests there's another API route (e.g., in `pages/api/webhooks/trigger/[path].ts`) that actually *receives* and processes incoming webhook events for different `foundWebhook.path` values.

```typescript
    logger.info(`[${requestId}] Testing webhook for provider: ${provider}`, {
      webhookId,
      path: foundWebhook.path,
      isActive: foundWebhook.isActive,
    })
```
*   **`logger.info(...)`**: Logs an informational message indicating which provider is being tested, along with relevant details about the webhook like its ID, path, and `isActive` status. This provides context for the upcoming provider-specific tests.

---

```typescript
    switch (provider) {
```
*   **`switch (provider) {`**: This `switch` statement is the core logic that branches the execution based on the `provider` type (e.g., 'whatsapp', 'telegram', 'github'). Each `case` handles the specific testing requirements for that provider.

---

#### **Case: 'whatsapp'**

```typescript
      case 'whatsapp': {
        const verificationToken = providerConfig.verificationToken
```
*   **`case 'whatsapp': {`**: This block handles testing for WhatsApp webhooks.
*   **`const verificationToken = providerConfig.verificationToken`**: Extracts the `verificationToken` from the `providerConfig` object. WhatsApp requires a token to verify the authenticity of webhook URLs during setup.

```typescript
        if (!verificationToken) {
          logger.warn(`[${requestId}] WhatsApp webhook missing verification token: ${webhookId}`)
          return NextResponse.json(
            { success: false, error: 'Webhook has no verification token' },
            { status: 400 }
          )
        }
```
*   **`if (!verificationToken)`**: Checks if the required `verificationToken` is missing.
*   **`logger.warn(...)`**: Logs a warning if the token is missing.
*   **`return NextResponse.json(...)`**: Returns a 400 (Bad Request) response indicating that the webhook configuration is incomplete.

```typescript
        const challenge = `test_${Date.now()}`
```
*   **`const challenge = \`test_${Date.now()}\``**: Generates a unique `challenge` string. When WhatsApp verifies a webhook, it sends a `hub.challenge` parameter, and the webhook must respond with the exact same string to confirm its authenticity. This line simulates that challenge.

```typescript
        const whatsappUrl = `${webhookUrl}?hub.mode=subscribe&hub.verify_token=${verificationToken}&hub.challenge=${challenge}`
```
*   **`const whatsappUrl = \`...\``**: Constructs the URL that this test will call. This URL is formatted *exactly* how WhatsApp would ping the application's webhook trigger endpoint (`webhookUrl`) for verification.
    *   `hub.mode=subscribe`: Indicates the intent to subscribe to webhook events.
    *   `hub.verify_token=${verificationToken}`: Passes the configured verification token.
    *   `hub.challenge=${challenge}`: Passes the generated challenge string.

```typescript
        logger.debug(`[${requestId}] Testing WhatsApp webhook verification`, {
          webhookId,
          challenge,
        })
```
*   **`logger.debug(...)`**: Logs debug information about the WhatsApp verification test, including the `webhookId` and the `challenge` string.

```typescript
        const response = await fetch(whatsappUrl, {
          headers: {
            'User-Agent': 'facebookplatform/1.0',
          },
        })
```
*   **`const response = await fetch(whatsappUrl, { ... })`**: Makes an HTTP GET request to the `whatsappUrl` that was just constructed.
    *   `await fetch(...)`: Waits for the response from the simulated WhatsApp verification request. This request is actually hitting the application's *own* `/api/webhooks/trigger/[path]` endpoint.
    *   `headers: { 'User-Agent': 'facebookplatform/1.0' }`: Sets the `User-Agent` header to mimic how WhatsApp (owned by Facebook) would identify itself, which might be important for the webhook trigger endpoint to recognize and process the request correctly.

```typescript
        const status = response.status
        const contentType = response.headers.get('content-type')
        const responseText = await response.text()
```
*   **`const status = response.status`**: Gets the HTTP status code from the `fetch` response.
*   **`const contentType = response.headers.get('content-type')`**: Gets the `Content-Type` header from the response.
*   **`const responseText = await response.text()`**: Reads the entire response body as plain text. This is where the application's webhook trigger endpoint is expected to echo back the `challenge` string.

```typescript
        const success = status === 200 && responseText === challenge
```
*   **`const success = status === 200 && responseText === challenge`**: Determines if the WhatsApp verification test was successful. For WhatsApp, a successful verification means:
    *   The HTTP status code is 200 OK.
    *   The response body *exactly* matches the `challenge` string sent in the request.

```typescript
        if (success) {
          logger.info(`[${requestId}] WhatsApp webhook verification successful: ${webhookId}`)
        } else {
          logger.warn(`[${requestId}] WhatsApp webhook verification failed: ${webhookId}`, {
            status,
            contentType,
            responseTextLength: responseText.length,
          })
        }
```
*   **`if (success) { ... } else { ... }`**: Logs the outcome of the WhatsApp verification test as either `info` (successful) or `warn` (failed), including diagnostic details if it failed.

```typescript
        return NextResponse.json({
          success,
          webhook: {
            id: foundWebhook.id,
            url: webhookUrl,
            verificationToken,
            isActive: foundWebhook.isActive,
          },
          test: {
            status,
            contentType,
            responseText,
            expectedStatus: 200,
            expectedContentType: 'text/plain',
            expectedResponse: challenge,
          },
          message: success
            ? 'Webhook configuration is valid. You can now use this URL in WhatsApp.'
            : 'Webhook verification failed. Please check your configuration.',
          diagnostics: {
            statusMatch: status === 200 ? '✅ Status code is 200' : '❌ Status code should be 200',
            contentTypeMatch:
              contentType === 'text/plain'
                ? '✅ Content-Type is text/plain'
                : '❌ Content-Type should be text/plain',
            bodyMatch:
              responseText === challenge
                ? '✅ Response body matches challenge'
                : '❌ Response body should exactly match the challenge string',
          },
        })
      }
```
*   **`return NextResponse.json(...)`**: Returns a comprehensive JSON response to the client with:
    *   `success`: Boolean indicating the test result.
    *   `webhook`: Basic details about the webhook being tested.
    *   `test`: Raw details from the `fetch` response (status, content type, response text) and what was expected.
    *   `message`: A user-friendly message based on success or failure.
    *   `diagnostics`: Detailed breakdown of what passed/failed (status code, content type, response body match) with emoji indicators, making it easy for the user to troubleshoot.

---

#### **Case: 'telegram'**

```typescript
      case 'telegram': {
        const botToken = providerConfig.botToken
```
*   **`case 'telegram': {`**: This block handles testing for Telegram webhooks.
*   **`const botToken = providerConfig.botToken`**: Extracts the `botToken` from the `providerConfig`. Telegram webhooks typically require a bot token for configuration and interaction.

```typescript
        if (!botToken) {
          logger.warn(`[${requestId}] Telegram webhook missing configuration: ${webhookId}`)
          return NextResponse.json(
            { success: false, error: 'Webhook has incomplete configuration' },
            { status: 400 }
          )
        }
```
*   **`if (!botToken)`**: Checks if the `botToken` is missing.
*   **`logger.warn(...)`**: Logs a warning if the token is missing.
*   **`return NextResponse.json(...)`**: Returns a 400 (Bad Request) response indicating incomplete configuration.

```typescript
        const testMessage = {
          update_id: 12345,
          message: {
            message_id: 67890,
            from: {
              id: 123456789,
              first_name: 'Test',
              username: 'testbot',
            },
            chat: {
              id: 123456789,
              first_name: 'Test',
              username: 'testbot',
              type: 'private',
            },
            date: Math.floor(Date.now() / 1000),
            text: 'This is a test message',
          },
        }
```
*   **`const testMessage = { ... }`**: Defines a sample JSON payload that mimics an incoming Telegram `update` object (specifically, a `message` update). This will be sent to the webhook URL to simulate a real event.

```typescript
        logger.debug(`[${requestId}] Testing Telegram webhook connection`, {
          webhookId,
          url: webhookUrl,
        })
```
*   **`logger.debug(...)`**: Logs debug information about the Telegram webhook test, including the `webhookId` and the `webhookUrl` it will hit.

```typescript
        const response = await fetch(webhookUrl, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'User-Agent': 'TelegramBot/1.0',
          },
          body: JSON.stringify(testMessage),
        })
```
*   **`const response = await fetch(webhookUrl, { ... })`**: Makes an HTTP POST request to the application's `webhookUrl` (the webhook trigger endpoint).
    *   `method: 'POST'`: Specifies that it's a POST request.
    *   `headers: { ... }`: Sets headers:
        *   `'Content-Type': 'application/json'`: Indicates that the request body is JSON.
        *   `'User-Agent': 'TelegramBot/1.0'`: Mimics Telegram's typical `User-Agent` header.
    *   `body: JSON.stringify(testMessage)`: Sends the `testMessage` object as a JSON string in the request body.

```typescript
        const status = response.status
        let responseText = ''
        try {
          responseText = await response.text()
        } catch (_e) {}
```
*   **`const status = response.status`**: Gets the HTTP status code from the response.
*   **`let responseText = '' ... try { ... } catch (_e) {}`**: Attempts to read the response body as text. A `try...catch` block is used here because not all responses (especially error responses) might have a readable body, or parsing it might fail, and we want to prevent that from crashing the entire test.

```typescript
        const success = status >= 200 && status < 300
```
*   **`const success = status >= 200 && status < 300`**: Determines if the Telegram test was successful. For most HTTP interactions, any status code in the 2xx range (200-299) indicates success.

```typescript
        if (success) {
          logger.info(`[${requestId}] Telegram webhook test successful: ${webhookId}`)
        } else {
          logger.warn(`[${requestId}] Telegram webhook test failed: ${webhookId}`, {
            status,
            responseText,
          })
        }
```
*   **`if (success) { ... } else { ... }`**: Logs the outcome of the Telegram test.

```typescript
        let webhookInfo = null
        try {
          const webhookInfoUrl = `https://api.telegram.org/bot${botToken}/getWebhookInfo`
          const infoResponse = await fetch(webhookInfoUrl, {
            headers: {
              'User-Agent': 'TelegramBot/1.0',
            },
          })
          if (infoResponse.ok) {
            const infoJson = await infoResponse.json()
            if (infoJson.ok) {
              webhookInfo = infoJson.result
            }
          }
        } catch (e) {
          logger.warn(`[${requestId}] Failed to get Telegram webhook info`, e)
        }
```
*   **`let webhookInfo = null ... try { ... } catch (e) { ... }`**: This block attempts to fetch *actual* webhook status information directly from Telegram's Bot API.
    *   `const webhookInfoUrl = \`https://api.telegram.org/bot${botToken}/getWebhookInfo\``: Constructs the URL to call Telegram's `getWebhookInfo` endpoint, using the provided `botToken`.
    *   `const infoResponse = await fetch(...)`: Makes a GET request to Telegram's API.
    *   `infoResponse.ok`: Checks if the HTTP status code is in the 200-299 range.
    *   `await infoResponse.json()`: Parses the JSON response from Telegram.
    *   `infoJson.ok`: Telegram's API responses include an `ok` field to indicate if the API call itself was successful.
    *   `webhookInfo = infoJson.result`: If successful, stores the detailed webhook information.
    *   The `try...catch` here specifically handles errors that might occur when trying to fetch or parse Telegram's webhook info, logging them without stopping the primary test result.

```typescript
        const curlCommand = [
          `curl -X POST "${webhookUrl}"`,
          `-H "Content-Type: application/json"`,
          `-H "User-Agent: TelegramBot/1.0"`,
          `-d '${JSON.stringify(testMessage, null, 2)}'`,
        ].join(' \\\n')
```
*   **`const curlCommand = [ ... ].join(' \\\n')`**: Constructs a `curl` command string. This provides the user with a ready-to-use command to manually test the webhook from their terminal, showing exactly what payload and headers would be sent. `JSON.stringify(testMessage, null, 2)` formats the JSON payload with indentation for readability. ` \\\n` is used to join lines in a way that allows the command to be copied and pasted multi-line in a shell.

```typescript
        return NextResponse.json({
          success,
          webhook: {
            id: foundWebhook.id,
            url: webhookUrl,
            botToken: `${botToken.substring(0, 5)}...${botToken.substring(botToken.length - 5)}`, // Show partial token for security
            isActive: foundWebhook.isActive,
          },
          test: {
            status,
            responseText,
            webhookInfo,
          },
          message: success
            ? 'Telegram webhook appears to be working. Any message sent to your bot will trigger the workflow.'
            : 'Telegram webhook test failed. Please check server logs for more details.',
          curlCommand,
          info: 'To fix issues with Telegram webhooks getting 403 Forbidden responses, ensure the webhook request includes a User-Agent header.',
        })
      }
```
*   **`return NextResponse.json(...)`**: Returns a comprehensive JSON response to the client with:
    *   `success`: Boolean indicating test result.
    *   `webhook`: Basic webhook details, including a partially masked `botToken` for security.
    *   `test`: Raw test results (status, response text) and the `webhookInfo` fetched from Telegram.
    *   `message`: User-friendly message.
    *   `curlCommand`: The generated curl command for manual testing.
    *   `info`: A helpful tip for common Telegram webhook issues.

---

#### **Case: 'github'**

```typescript
      case 'github': {
        const contentType = providerConfig.contentType || 'application/json'
```
*   **`case 'github': {`**: Handles testing for GitHub webhooks.
*   **`const contentType = providerConfig.contentType || 'application/json'`**: Extracts the `contentType` from `providerConfig`, defaulting to `'application/json'` if not specified. GitHub webhooks usually send JSON payloads.

```typescript
        logger.info(`[${requestId}] GitHub webhook test successful: ${webhookId}`)
        return NextResponse.json({
          success: true,
          webhook: {
            id: foundWebhook.id,
            url: webhookUrl,
            contentType,
            isActive: foundWebhook.isActive,
          },
          message:
            'GitHub webhook configuration is valid. Use this URL in your GitHub repository settings.',
          setup: {
            url: webhookUrl,
            contentType,
            events: ['push', 'pull_request', 'issues', 'issue_comment'],
          },
        })
      }
```
*   **`logger.info(...)`**: Logs an informational message. For GitHub, a direct server-side test isn't typically performed in the same way as WhatsApp/Telegram. The main "test" is to ensure the configuration is complete and provide guidance.
*   **`return NextResponse.json(...)`**: Returns a JSON response with:
    *   `success: true`: Assumes validity if basic config is present (or no explicit "failure" condition is met).
    *   `webhook`: Basic details and the content type.
    *   `message`: Instructions on where to use the URL.
    *   `setup`: Recommended configuration details for GitHub, including the URL, content type, and common events to subscribe to.

---

#### **Case: 'stripe'**

```typescript
      case 'stripe': {
        logger.info(`[${requestId}] Stripe webhook test successful: ${webhookId}`)
        return NextResponse.json({
          success: true,
          webhook: {
            id: foundWebhook.id,
            url: webhookUrl,
            isActive: foundWebhook.isActive,
          },
          message: 'Stripe webhook configuration is valid. Use this URL in your Stripe dashboard.',
          setup: {
            url: webhookUrl,
            events: [
              'charge.succeeded',
              'invoice.payment_succeeded',
              'customer.subscription.created',
            ],
          },
        })
      }
```
*   **`case 'stripe': {`**: Handles testing for Stripe webhooks.
*   **`logger.info(...)`**: Logs an informational message. Similar to GitHub, direct server-side pinging is not the standard test.
*   **`return NextResponse.json(...)`**: Returns a JSON response with:
    *   `success: true`: Assumes validity.
    *   `webhook`: Basic details.
    *   `message`: Instructions on where to use the URL in the Stripe dashboard.
    *   `setup`: Recommended configuration details for Stripe, including the URL and common events to subscribe to.

---

#### **Case: 'generic'**

```typescript
      case 'generic': {
        const token = providerConfig.token
        const secretHeaderName = providerConfig.secretHeaderName
        const requireAuth = providerConfig.requireAuth
        const allowedIps = providerConfig.allowedIps
```
*   **`case 'generic': {`**: Handles testing for a generic webhook, which might have custom authentication or IP restrictions.
*   **`const token = providerConfig.token`**: Extracts a generic authentication token.
*   **`const secretHeaderName = providerConfig.secretHeaderName`**: Extracts a custom header name for the secret, if one is configured.
*   **`const requireAuth = providerConfig.requireAuth`**: Boolean indicating if authentication is required.
*   **`const allowedIps = providerConfig.allowedIps`**: An array of allowed IP addresses for incoming requests.

```typescript
        let curlCommand = `curl -X POST "${webhookUrl}" -H "Content-Type: application/json"`
```
*   **`let curlCommand = \`...\``**: Initializes a `curl` command string for a basic POST request with JSON content type.

```typescript
        if (requireAuth && token) {
          if (secretHeaderName) {
            curlCommand += ` -H "${secretHeaderName}: ${token}"`
          } else {
            curlCommand += ` -H "Authorization: Bearer ${token}"`
          }
        }
```
*   **`if (requireAuth && token) { ... }`**: Conditionally adds authentication headers to the `curlCommand` if authentication is required and a token is provided.
    *   If `secretHeaderName` is set, it uses that custom header.
    *   Otherwise, it defaults to a standard `Authorization: Bearer <token>` header.

```typescript
        curlCommand += ` -d '{"event":"test_event","timestamp":"${new Date().toISOString()}"}'`
```
*   **`curlCommand += \`...\``**: Appends a sample JSON payload to the `curlCommand`. This payload is a simple `test_event` with a timestamp.

```typescript
        logger.info(`[${requestId}] General webhook test successful: ${webhookId}`)
        return NextResponse.json({
          success: true,
          webhook: {
            id: foundWebhook.id,
            url: webhookUrl,
            isActive: foundWebhook.isActive,
          },
          message:
            'General webhook configuration is valid. Use the URL and authentication details as needed.',
          details: {
            requireAuth: requireAuth || false,
            hasToken: !!token,
            hasCustomHeader: !!secretHeaderName,
            customHeaderName: secretHeaderName,
            hasIpRestrictions: Array.isArray(allowedIps) && allowedIps.length > 0,
          },
          test: {
            curlCommand,
            headers: requireAuth
              ? secretHeaderName
                ? { [secretHeaderName]: token }
                : { Authorization: `Bearer ${token}` }
              : {},
            samplePayload: {
              event: 'test_event',
              timestamp: new Date().toISOString(),
            },
          },
        })
      }
```
*   **`logger.info(...)`**: Logs an informational message.
*   **`return NextResponse.json(...)`**: Returns a JSON response with:
    *   `success: true`: Assumes validity.
    *   `webhook`: Basic details.
    *   `message`: General guidance.
    *   `details`: Information about the configured generic webhook features (auth, custom headers, IP restrictions).
    *   `test`: The generated `curlCommand`, the expected headers (if authentication is used), and the sample payload.

---

#### **Case: 'slack'**

```typescript
      case 'slack': {
        const signingSecret = providerConfig.signingSecret
```
*   **`case 'slack': {`**: Handles testing for Slack webhooks.
*   **`const signingSecret = providerConfig.signingSecret`**: Extracts the `signingSecret` from `providerConfig`. Slack webhooks use a signing secret to verify the authenticity of incoming requests.

```typescript
        if (!signingSecret) {
          logger.warn(`[${requestId}] Slack webhook missing signing secret: ${webhookId}`)
          return NextResponse.json(
            { success: false, error: 'Webhook has no signing secret configured' },
            { status: 400 }
          )
        }
```
*   **`if (!signingSecret)`**: Checks if the `signingSecret` is missing.
*   **`logger.warn(...)`**: Logs a warning if missing.
*   **`return NextResponse.json(...)`**: Returns a 400 (Bad Request) response.

```typescript
        logger.info(`[${requestId}] Slack webhook test successful: ${webhookId}`)
        return NextResponse.json({
          success: true,
          webhook: {
            id: foundWebhook.id,
            url: webhookUrl,
            isActive: foundWebhook.isActive,
          },
          message:
            'Slack webhook configuration is valid. Use this URL in your Slack Event Subscriptions settings.',
          setup: {
            url: webhookUrl,
            events: ['message.channels', 'reaction_added', 'app_mention'],
            signingSecretConfigured: true,
          },
          test: {
            curlCommand: [
              `curl -X POST "${webhookUrl}"`,
              `-H "Content-Type: application/json"`,
              `-H "X-Slack-Request-Timestamp: $(date +%s)"`,
              `-H "X-Slack-Signature: v0=$(date +%s)"`, // Simplified signature for example, actual calculation is complex
              `-d '{"type":"event_callback","event":{"type":"message","channel":"C0123456789","user":"U0123456789","text":"Hello from Slack!","ts":"1234567890.123456"},"team_id":"T0123456789"}'`,
            ].join(' \\\n'),
            samplePayload: {
              type: 'event_callback',
              token: 'XXYYZZ',
              team_id: 'T123ABC',
              event: {
                type: 'message',
                user: 'U123ABC',
                text: 'Hello from Slack!',
                ts: '1234567890.1234',
              },
              event_id: 'Ev123ABC',
            },
          },
        })
      }
```
*   **`logger.info(...)`**: Logs an informational message.
*   **`return NextResponse.json(...)`**: Returns a JSON response with:
    *   `success: true`: Assumes validity if the signing secret is present.
    *   `webhook`: Basic details.
    *   `message`: Instructions for Slack configuration.
    *   `setup`: Recommended setup details, including URL, common events, and confirmation that the signing secret is configured.
    *   `test`: A `curlCommand` demonstrating how Slack would send a request (including `X-Slack-Request-Timestamp` and a placeholder `X-Slack-Signature`) and a `samplePayload`.

---

#### **Case: 'airtable'**

```typescript
      case 'airtable': {
        const baseId = providerConfig.baseId
        const tableId = providerConfig.tableId
        const webhookSecret = providerConfig.webhookSecret
```
*   **`case 'airtable': {`**: Handles testing for Airtable webhooks.
*   **`const baseId = providerConfig.baseId`**: Extracts the Airtable Base ID.
*   **`const tableId = providerConfig.tableId`**: Extracts the Airtable Table ID.
*   **`const webhookSecret = providerConfig.webhookSecret`**: Extracts the optional webhook secret for Airtable.

```typescript
        if (!baseId || !tableId) {
          logger.warn(`[${requestId}] Airtable webhook missing Base ID or Table ID: ${webhookId}`)
          return NextResponse.json(
            {
              success: false,
              error: 'Webhook configuration is incomplete (missing Base ID or Table ID)',
            },
            { status: 400 }
          )
        }
```
*   **`if (!baseId || !tableId)`**: Checks if the required `baseId` or `tableId` are missing.
*   **`logger.warn(...)`**: Logs a warning if missing.
*   **`return NextResponse.json(...)`**: Returns a 400 (Bad Request) response.

```typescript
        const samplePayload = {
          webhook: {
            id: 'whiYOUR_WEBHOOK_ID',
          },
          base: {
            id: baseId,
          },
          payloadFormat: 'v0',
          actionMetadata: {
            source: 'tableOrViewChange',
            sourceMetadata: {},
          },
          payloads: [
            {
              timestamp: new Date().toISOString(),
              baseTransactionNumber: Date.now(),
              changedTablesById: {
                [tableId]: {
                  changedRecordsById: {
                    recSAMPLEID1: {
                      current: { cellValuesByFieldId: { fldSAMPLEID: 'New Value' } },
                      previous: { cellValuesByFieldId: { fldSAMPLEID: 'Old Value' } },
                    },
                  },
                  changedFieldsById: {},
                  changedViewsById: {},
                },
              },
            },
          ],
        }
```
*   **`const samplePayload = { ... }`**: Defines a detailed sample JSON payload that mimics an Airtable `tableOrViewChange` event, including the configured `baseId` and `tableId`. This payload shows what an actual event from Airtable would look like.

```typescript
        let curlCommand = `curl -X POST "${webhookUrl}" -H "Content-Type: application/json"`
        curlCommand += ` -d '${JSON.stringify(samplePayload, null, 2)}'`
```
*   **`let curlCommand = ...`**: Constructs a `curl` command including the `webhookUrl`, content type, and the `samplePayload` stringified for readability.

```typescript
        logger.info(`[${requestId}] Airtable webhook test successful: ${webhookId}`)
        return NextResponse.json({
          success: true,
          webhook: {
            id: foundWebhook.id,
            url: webhookUrl,
            baseId: baseId,
            tableId: tableId,
            secretConfigured: !!webhookSecret,
            isActive: foundWebhook.isActive,
          },
          message:
            'Airtable webhook configuration appears valid. Use the sample curl command to manually send a test payload to your webhook URL.',
          test: {
            curlCommand: curlCommand,
            samplePayload: samplePayload,
          },
        })
      }
```
*   **`logger.info(...)`**: Logs an informational message.
*   **`return NextResponse.json(...)`**: Returns a JSON response with:
    *   `success: true`: Assumes validity.
    *   `webhook`: Basic details, including `baseId`, `tableId`, and if a `secret` is configured.
    *   `message`: Instructions to use the `curlCommand` for manual testing.
    *   `test`: The `curlCommand` and the `samplePayload`.

---

#### **Case: 'microsoftteams'**

```typescript
      case 'microsoftteams': {
        const hmacSecret = providerConfig.hmacSecret
```
*   **`case 'microsoftteams': {`**: Handles testing for Microsoft Teams webhooks.
*   **`const hmacSecret = providerConfig.hmacSecret`**: Extracts the `hmacSecret` from `providerConfig`. Microsoft Teams outgoing webhooks use an HMAC secret for signature verification.

```typescript
        if (!hmacSecret) {
          logger.warn(`[${requestId}] Microsoft Teams webhook missing HMAC secret: ${webhookId}`)
          return NextResponse.json(
            { success: false, error: 'Microsoft Teams webhook requires HMAC secret' },
            { status: 400 }
          )
        }
```
*   **`if (!hmacSecret)`**: Checks if the `hmacSecret` is missing.
*   **`logger.warn(...)`**: Logs a warning if missing.
*   **`return NextResponse.json(...)`**: Returns a 400 (Bad Request) response.

```typescript
        logger.info(`[${requestId}] Microsoft Teams webhook test successful: ${webhookId}`)
        return NextResponse.json({
          success: true,
          webhook: {
            id: foundWebhook.id,
            url: webhookUrl,
            isActive: foundWebhook.isActive,
          },
          message: 'Microsoft Teams outgoing webhook configuration is valid.',
          setup: {
            url: webhookUrl,
            hmacSecretConfigured: !!hmacSecret,
            instructions: [
              'Create an outgoing webhook in Microsoft Teams',
              'Set the callback URL to the webhook URL above',
              'Copy the HMAC security token to the configuration',
              'Users can trigger the webhook by @mentioning it in Teams',
            ],
          },
          test: {
            curlCommand: `curl -X POST "${webhookUrl}" \\
  -H "Content-Type: application/json" \\
  -H "Authorization: HMAC <signature>" \\
  -d '{"type":"message","text":"Hello from Microsoft Teams!","from":{"id":"test","name":"Test User"}}'`,
            samplePayload: {
              type: 'message',
              id: '1234567890',
              timestamp: new Date().toISOString(),
              text: 'Hello Sim Bot!',
              from: {
                id: '29:1234567890abcdef',
                name: 'Test User',
              },
              conversation: {
                id: '19:meeting_abcdef@thread.v2',
              },
            },
          },
        })
      }
```
*   **`logger.info(...)`**: Logs an informational message.
*   **`return NextResponse.json(...)`**: Returns a JSON response with:
    *   `success: true`: Assumes validity if `hmacSecret` is present.
    *   `webhook`: Basic details.
    *   `message`: Confirmation of valid configuration.
    *   `setup`: Instructions for configuring the webhook in Microsoft Teams, including how to get and use the HMAC secret.
    *   `test`: A `curlCommand` demonstrating a sample request (including the `Authorization: HMAC <signature>` header) and a `samplePayload`.

---

#### **Default Case**

```typescript
      default: {
        logger.info(`[${requestId}] Generic webhook test successful: ${webhookId}`)
        return NextResponse.json({
          success: true,
          webhook: {
            id: foundWebhook.id,
            url: webhookUrl,
            provider: foundWebhook.provider,
            isActive: foundWebhook.isActive,
          },
          message:
            'Webhook configuration is valid. You can use this URL to receive webhook events.',
        })
      }
```
*   **`default: {`**: This block is executed if the `provider` type does not match any of the explicit `case` statements. It acts as a fallback for unknown or unspecific providers.
*   **`logger.info(...)`**: Logs an informational message for a generic webhook test.
*   **`return NextResponse.json(...)`**: Returns a JSON response with:
    *   `success: true`: Assumes validity, as no specific validation logic is applied here.
    *   `webhook`: Basic details including the `provider` name.
    *   `message`: A general message indicating the URL can be used.

---

#### **Error Handling (Catch Block)**

```typescript
  } catch (error: any) {
```
*   **`} catch (error: any) {`**: This block catches any unhandled exceptions that occurred within the `try` block. `error: any` is used to catch any type of error, but in a production environment, you might want more specific error types.

```typescript
    logger.error(`[${requestId}] Error testing webhook`, error)
```
*   **`logger.error(...)`**: Logs the error message, including the `requestId` for tracing and the actual `error` object, which provides details like stack traces.

```typescript
    return NextResponse.json(
      {
        success: false,
        error: 'Test failed',
        message: error.message,
      },
      { status: 500 }
    )
  }
}
```
*   **`return NextResponse.json(...)`**: Sends a JSON response to the client indicating a failure.
    *   `success: false`: Marks the operation as unsuccessful.
    *   `error: 'Test failed'`: A generic error message.
    *   `message: error.message`: Includes the specific error message from the caught exception (e.g., "Network error," "Database connection failed").
    *   `{ status: 500 }`: Sets the HTTP status code to 500 (Internal Server Error), indicating a problem on the server side.

---

This file provides a powerful and flexible way to diagnose webhook configurations, making it easier for users to set up and troubleshoot integrations with various external services.