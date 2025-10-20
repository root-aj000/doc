This TypeScript file is a crucial part of a billing system, specifically designed to handle **manual enterprise subscriptions** initiated through Stripe webhooks. Its primary function is to process a `Stripe.Event` (typically `subscription.created` or `subscription.updated`) for an enterprise plan, validate its metadata, record or update the subscription details in the application's database, update the associated organization's usage limits, and send a confirmation email to the user.

In essence, it acts as a bridge between Stripe's subscription management and the application's internal data model for enterprise customers.

---

### **Detailed Code Explanation**

Let's break down the code section by section.

#### **1. Imports and Setup**

```typescript
import { db } from '@sim/db'
import { organization, subscription, user } from '@sim/db/schema'
import { eq } from 'drizzle-orm'
import type Stripe from 'stripe'
import {
  getEmailSubject,
  renderEnterpriseSubscriptionEmail,
} from '@/components/emails/render-email'
import { sendEmail } from '@/lib/email/mailer'
import { getFromEmailAddress } from '@/lib/email/utils'
import { createLogger } from '@/lib/logs/console/logger'
import type { EnterpriseSubscriptionMetadata } from '../types'

const logger = createLogger('BillingEnterprise')
```

*   **`import { db } from '@sim/db'`**: Imports the Drizzle ORM database client. `db` is the instance used to interact with your PostgreSQL (or other) database.
*   **`import { organization, subscription, user } from '@sim/db/schema'`**: Imports table schemas (definitions of your database tables) for `organization`, `subscription`, and `user` from your Drizzle ORM schema file. These allow Drizzle to understand the structure of your data and build queries.
*   **`import { eq } from 'drizzle-orm'`**: Imports the `eq` (equals) operator from Drizzle ORM. This is used in `WHERE` clauses to compare a column's value with a specific value.
*   **`import type Stripe from 'stripe'`**: Imports the `Stripe` type definitions. The `type` keyword ensures this is only used for type checking at compile time and doesn't add any runtime code, preventing your server from needing the full Stripe library if it's not performing API calls itself.
*   **`import { getEmailSubject, renderEnterpriseSubscriptionEmail, } from '@/components/emails/render-email'`**: Imports two functions related to email content generation:
    *   `getEmailSubject`: A utility to get a standardized subject line for emails.
    *   `renderEnterpriseSubscriptionEmail`: A function that generates the HTML content for the enterprise subscription confirmation email.
*   **`import { sendEmail } from '@/lib/email/mailer'`**: Imports the `sendEmail` function, which is responsible for actually sending emails (likely via a third-party service like Resend, SendGrid, etc.).
*   **`import { getFromEmailAddress } from '@/lib/email/utils'`**: Imports a utility function to retrieve the sender's email address.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a function to create a logger instance for structured logging throughout the application.
*   **`import type { EnterpriseSubscriptionMetadata } from '../types'`**: Imports a TypeScript type definition for `EnterpriseSubscriptionMetadata`. This type defines the expected structure of metadata specifically for enterprise subscriptions, helping ensure type safety when accessing these properties.
*   **`const logger = createLogger('BillingEnterprise')`**: Initializes a logger instance with the name `'BillingEnterprise'`. This logger will be used to output informational messages, warnings, and errors during the execution of this file's logic, making debugging and monitoring easier.

---

#### **2. Metadata Type Guard Function**

```typescript
function isEnterpriseMetadata(value: unknown): value is EnterpriseSubscriptionMetadata {
  return (
    !!value &&
    typeof value === 'object' &&
    'plan' in value &&
    'referenceId' in value &&
    'monthlyPrice' in value &&
    'seats' in value &&
    typeof value.plan === 'string' &&
    value.plan.toLowerCase() === 'enterprise' &&
    typeof value.referenceId === 'string' &&
    typeof value.monthlyPrice === 'string' &&
    typeof value.seats === 'string'
  )
}
```

This function is a **Type Guard**. Its purpose is to safely check if an `unknown` value (like `Stripe.Metadata`) conforms to the `EnterpriseSubscriptionMetadata` type. If it returns `true`, TypeScript can then safely treat `value` as `EnterpriseSubscriptionMetadata` within that scope, providing better type checking and autocompletion.

*   **`function isEnterpriseMetadata(value: unknown): value is EnterpriseSubscriptionMetadata`**: Defines a function named `isEnterpriseMetadata` that takes any `value` (type `unknown`). The return type `value is EnterpriseSubscriptionMetadata` is the special Type Guard syntax.
*   **`!!value`**: Checks if `value` is truthy (i.e., not `null`, `undefined`, `0`, `false`, `''`). This ensures we're dealing with *something*.
*   **`typeof value === 'object'`**: Ensures `value` is an object.
*   **`'plan' in value && 'referenceId' in value && ...`**: Checks if specific properties (`plan`, `referenceId`, `monthlyPrice`, `seats`) exist directly on the `value` object. This doesn't check their types yet, just their presence.
*   **`typeof value.plan === 'string' && value.plan.toLowerCase() === 'enterprise'`**: Checks if the `plan` property is a string and, when converted to lowercase, is exactly `'enterprise'`. This is a crucial check to confirm it's an enterprise subscription.
*   **`typeof value.referenceId === 'string' && ...`**: Checks if `referenceId`, `monthlyPrice`, and `seats` are all strings. Stripe metadata values are always strings, so this is expected, even if they represent numbers.

---

#### **3. `handleManualEnterpriseSubscription` Function**

This is the main asynchronous function that processes the Stripe event.

```typescript
export async function handleManualEnterpriseSubscription(event: Stripe.Event) {
  const stripeSubscription = event.data.object as Stripe.Subscription

  const metaPlan = (stripeSubscription.metadata?.plan as string | undefined)?.toLowerCase() || ''

  if (metaPlan !== 'enterprise') {
    logger.info('[subscription.created] Skipping non-enterprise subscription', {
      subscriptionId: stripeSubscription.id,
      plan: metaPlan || 'unknown',
    })
    return
  }
```

*   **`export async function handleManualEnterpriseSubscription(event: Stripe.Event)`**: Defines an asynchronous function `handleManualEnterpriseSubscription` that takes a `Stripe.Event` object as input. `export` means it can be used by other files.
*   **`const stripeSubscription = event.data.object as Stripe.Subscription`**: Extracts the main object from the Stripe event's `data` payload. For `subscription.created` or `subscription.updated` events, `event.data.object` contains a `Stripe.Subscription` object, so it's cast to that type.
*   **`const metaPlan = (stripeSubscription.metadata?.plan as string | undefined)?.toLowerCase() || ''`**: Safely retrieves the `plan` from the `stripeSubscription.metadata`.
    *   `stripeSubscription.metadata?.plan`: Accesses `metadata.plan` safely using optional chaining (`?.`).
    *   `as string | undefined`: Casts it to `string` or `undefined`.
    *   `?.toLowerCase()`: Converts the plan name to lowercase if it exists.
    *   `|| ''`: Provides an empty string as a fallback if `metaPlan` is `undefined` or `null`.
*   **`if (metaPlan !== 'enterprise') { ... }`**: This is an early exit condition. If the plan in the metadata is not explicitly `'enterprise'`, the function logs an informational message and stops execution, as it's not intended to handle other plan types.

---

##### **Customer ID Validation**

```typescript
  const stripeCustomerId = stripeSubscription.customer as string

  if (!stripeCustomerId) {
    logger.error('[subscription.created] Missing Stripe customer ID', {
      subscriptionId: stripeSubscription.id,
    })
    throw new Error('Missing Stripe customer ID on subscription')
  }
```

*   **`const stripeCustomerId = stripeSubscription.customer as string`**: Extracts the `customer` ID from the Stripe subscription object. This is the ID of the customer in Stripe associated with this subscription. It's cast to `string` as it's expected to be a string.
*   **`if (!stripeCustomerId) { ... }`**: Checks if `stripeCustomerId` is missing (null, undefined, or empty string).
    *   **`logger.error(...)`**: Logs an error if the customer ID is missing, which is a critical piece of information.
    *   **`throw new Error(...)`**: Throws an error, stopping the function's execution. A subscription without a customer ID cannot be processed correctly.

---

##### **Metadata Extraction and Validation**

```typescript
  const metadata = stripeSubscription.metadata || {}

  const referenceId =
    typeof metadata.referenceId === 'string' && metadata.referenceId.length > 0
      ? metadata.referenceId
      : null

  if (!referenceId) {
    logger.error('[subscription.created] Unable to resolve referenceId', {
      subscriptionId: stripeSubscription.id,
      stripeCustomerId,
    })
    throw new Error('Unable to resolve referenceId for subscription')
  }

  if (!isEnterpriseMetadata(metadata)) {
    logger.error('[subscription.created] Invalid enterprise metadata shape', {
      subscriptionId: stripeSubscription.id,
      metadata,
    })
    throw new Error('Invalid enterprise metadata for subscription')
  }
  const enterpriseMetadata = metadata
  const metadataJson: Record<string, unknown> = { ...enterpriseMetadata }
```

*   **`const metadata = stripeSubscription.metadata || {}`**: Gets the `metadata` object from the Stripe subscription. If `metadata` is `null` or `undefined`, it defaults to an empty object `{}` to prevent errors when trying to access properties.
*   **`const referenceId = ...`**: Extracts and validates the `referenceId` from the metadata.
    *   `typeof metadata.referenceId === 'string' && metadata.referenceId.length > 0`: Checks if `referenceId` exists, is a string, and is not empty.
    *   `? metadata.referenceId : null`: If valid, assigns the value; otherwise, assigns `null`. The `referenceId` for enterprise plans is expected to be the internal `organization.id`.
*   **`if (!referenceId) { ... }`**: If `referenceId` is `null` (i.e., missing or invalid), logs an error and throws an exception. This is critical as `referenceId` links the Stripe subscription to an organization in the application.
*   **`if (!isEnterpriseMetadata(metadata)) { ... }`**: Calls the type guard function `isEnterpriseMetadata` to ensure the metadata object conforms to the expected `EnterpriseSubscriptionMetadata` structure.
    *   If it doesn't conform, an error is logged, and an exception is thrown.
*   **`const enterpriseMetadata = metadata`**: Once `isEnterpriseMetadata` returns true, TypeScript knows `metadata` is of type `EnterpriseSubscriptionMetadata`. This line reassigns it to a new constant name, primarily for clarity, so we explicitly treat it as the validated enterprise metadata.
*   **`const metadataJson: Record<string, unknown> = { ...enterpriseMetadata }`**: Creates a shallow copy of the `enterpriseMetadata` into `metadataJson`. This is often done to ensure the object is a plain JavaScript object, suitable for storage (e.g., in a JSONB column in the database).

---

##### **Seats and Monthly Price Validation**

```typescript
  // Extract and parse seats and monthly price from metadata (they come as strings from Stripe)
  const seats = Number.parseInt(enterpriseMetadata.seats, 10)
  const monthlyPrice = Number.parseFloat(enterpriseMetadata.monthlyPrice)

  if (!seats || seats <= 0 || Number.isNaN(seats)) {
    logger.error('[subscription.created] Invalid or missing seats in enterprise metadata', {
      subscriptionId: stripeSubscription.id,
      seatsRaw: enterpriseMetadata.seats,
      seatsParsed: seats,
    })
    throw new Error('Enterprise subscription must include valid seats in metadata')
  }

  if (!monthlyPrice || monthlyPrice <= 0 || Number.isNaN(monthlyPrice)) {
    logger.error('[subscription.created] Invalid or missing monthlyPrice in enterprise metadata', {
      subscriptionId: stripeSubscription.id,
      monthlyPriceRaw: enterpriseMetadata.monthlyPrice,
      monthlyPriceParsed: monthlyPrice,
    })
    throw new Error('Enterprise subscription must include valid monthlyPrice in metadata')
  }
```

*   **`const seats = Number.parseInt(enterpriseMetadata.seats, 10)`**: Converts the `seats` string from metadata into an integer. `10` specifies base 10 for parsing.
*   **`const monthlyPrice = Number.parseFloat(enterpriseMetadata.monthlyPrice)`**: Converts the `monthlyPrice` string from metadata into a floating-point number.
*   **`if (!seats || seats <= 0 || Number.isNaN(seats)) { ... }`**: Validates the parsed `seats` value.
    *   `!seats`: Catches `0` (which is falsy) or `NaN`.
    *   `seats <= 0`: Explicitly checks for non-positive values.
    *   `Number.isNaN(seats)`: Checks if the parsing failed and resulted in `NaN`.
    *   If invalid, logs an error (including raw and parsed values for debugging) and throws an exception.
*   **`if (!monthlyPrice || monthlyPrice <= 0 || Number.isNaN(monthlyPrice)) { ... }`**: Performs similar validation for `monthlyPrice`.
    *   If invalid, logs an error and throws an exception.

---

##### **Subscription Data Preparation**

```typescript
  // Get the first subscription item which contains the period information
  const referenceItem = stripeSubscription.items?.data?.[0]

  const subscriptionRow = {
    id: crypto.randomUUID(),
    plan: 'enterprise',
    referenceId,
    stripeCustomerId,
    stripeSubscriptionId: stripeSubscription.id,
    status: stripeSubscription.status || null,
    periodStart: referenceItem?.current_period_start
      ? new Date(referenceItem.current_period_start * 1000)
      : null,
    periodEnd: referenceItem?.current_period_end
      ? new Date(referenceItem.current_period_end * 1000)
      : null,
    cancelAtPeriodEnd: stripeSubscription.cancel_at_period_end ?? null,
    seats,
    trialStart: stripeSubscription.trial_start
      ? new Date(stripeSubscription.trial_start * 1000)
      : null,
    trialEnd: stripeSubscription.trial_end ? new Date(stripeSubscription.trial_end * 1000) : null,
    metadata: metadataJson,
  }
```

*   **`const referenceItem = stripeSubscription.items?.data?.[0]`**: Stripe subscriptions can have multiple "items" (e.g., for different pricing tiers or add-ons). This line attempts to get the first item in the `items.data` array, which typically holds the primary pricing information and period details. Optional chaining (`?.`) prevents errors if `items` or `data` are missing.
*   **`const subscriptionRow = { ... }`**: Creates an object `subscriptionRow` that maps the Stripe subscription data to the application's `subscription` table schema.
    *   **`id: crypto.randomUUID()`**: Generates a unique ID for the new subscription record in the application's database using the Web Crypto API.
    *   **`plan: 'enterprise'`**: Sets the plan explicitly to 'enterprise'.
    *   **`referenceId`**: Uses the validated `referenceId` (organization ID).
    *   **`stripeCustomerId`**: Uses the validated Stripe customer ID.
    *   **`stripeSubscriptionId: stripeSubscription.id`**: Stores the unique ID of the subscription from Stripe.
    *   **`status: stripeSubscription.status || null`**: Records the current status of the Stripe subscription, defaulting to `null` if missing.
    *   **`periodStart: ...`** and **`periodEnd: ...`**: Extracts the current billing period start and end timestamps from `referenceItem`. Stripe timestamps are in seconds, so they are multiplied by `1000` to convert to milliseconds for `new Date()`. If `referenceItem` or the timestamp is missing, it defaults to `null`.
    *   **`cancelAtPeriodEnd: stripeSubscription.cancel_at_period_end ?? null`**: Indicates if the subscription is set to cancel at the end of the current period. Uses nullish coalescing `??` for a cleaner default.
    *   **`seats`**: Uses the parsed `seats` value.
    *   **`trialStart: ...`** and **`trialEnd: ...`**: Similar to `periodStart`/`periodEnd`, extracts trial period timestamps if present, converting them to `Date` objects.
    *   **`metadata: metadataJson`**: Stores the validated and copied metadata object.

---

##### **Database Upsert (Subscription)**

```typescript
  const existing = await db
    .select({ id: subscription.id })
    .from(subscription)
    .where(eq(subscription.stripeSubscriptionId, stripeSubscription.id))
    .limit(1)

  if (existing.length > 0) {
    await db
      .update(subscription)
      .set({
        plan: subscriptionRow.plan,
        referenceId: subscriptionRow.referenceId,
        stripeCustomerId: subscriptionRow.stripeCustomerId,
        status: subscriptionRow.status,
        periodStart: subscriptionRow.periodStart,
        periodEnd: subscriptionRow.periodEnd,
        cancelAtPeriodEnd: subscriptionRow.cancelAtPeriodEnd,
        seats: subscriptionRow.seats,
        trialStart: subscriptionRow.trialStart,
        trialEnd: subscriptionRow.trialEnd,
        metadata: subscriptionRow.metadata,
      })
      .where(eq(subscription.stripeSubscriptionId, stripeSubscription.id))
  } else {
    await db.insert(subscription).values(subscriptionRow)
  }
```

This block performs an "upsert" operation: it checks if a subscription with the given Stripe ID already exists in the database. If it does, it updates the existing record; otherwise, it inserts a new one.

*   **`const existing = await db .select({ id: subscription.id }) .from(subscription) .where(eq(subscription.stripeSubscriptionId, stripeSubscription.id)) .limit(1)`**:
    *   This Drizzle ORM query attempts to find an existing subscription record in the `subscription` table.
    *   `select({ id: subscription.id })`: Selects only the `id` column, as we only need to know if a record exists.
    *   `from(subscription)`: Specifies the `subscription` table.
    *   `where(eq(subscription.stripeSubscriptionId, stripeSubscription.id))`: Filters for records where the `stripeSubscriptionId` matches the ID from the current Stripe event.
    *   `limit(1)`: Optimizes the query by telling the database to stop after finding the first match.
*   **`if (existing.length > 0) { ... }`**: If the `existing` array contains any results (meaning a record was found):
    *   **`await db.update(subscription).set({ ... }).where(...)`**: An `UPDATE` query is executed.
        *   `update(subscription)`: Targets the `subscription` table.
        *   `set({ ... })`: Updates all the fields with the values from `subscriptionRow`.
        *   `where(eq(subscription.stripeSubscriptionId, stripeSubscription.id))`: Ensures only the matching record is updated.
*   **`else { await db.insert(subscription).values(subscriptionRow) }`**: If `existing.length` is `0` (no matching record found):
    *   **`await db.insert(subscription).values(subscriptionRow)`**: An `INSERT` query is executed, adding a new record to the `subscription` table using the data prepared in `subscriptionRow`.

---

##### **Organization Usage Limit Update**

```typescript
  // Update the organization's usage limit to match the monthly price
  // The referenceId for enterprise plans is the organization ID
  try {
    await db
      .update(organization)
      .set({
        orgUsageLimit: monthlyPrice.toFixed(2),
        updatedAt: new Date(),
      })
      .where(eq(organization.id, referenceId))

    logger.info('[subscription.created] Updated organization usage limit', {
      organizationId: referenceId,
      usageLimit: monthlyPrice,
    })
  } catch (error) {
    logger.error('[subscription.created] Failed to update organization usage limit', {
      organizationId: referenceId,
      usageLimit: monthlyPrice,
      error,
    })
    // Don't throw - the subscription was created successfully, just log the error
  }
```

This block updates the associated organization's usage limit based on the monthly price of the enterprise subscription.

*   **`try { ... } catch (error) { ... }`**: This operation is wrapped in a `try...catch` block. This is important because updating the organization is secondary to creating the subscription. If this update fails, the subscription should still be considered successful, but the error should be logged.
*   **`await db.update(organization).set({ ... }).where(eq(organization.id, referenceId))`**:
    *   `update(organization)`: Targets the `organization` table.
    *   `set({ orgUsageLimit: monthlyPrice.toFixed(2), updatedAt: new Date() })`: Sets the `orgUsageLimit` to the `monthlyPrice`, formatted to two decimal places, and updates the `updatedAt` timestamp.
    *   `where(eq(organization.id, referenceId))`: Filters for the organization whose `id` matches the `referenceId` (which we established earlier is the organization ID).
*   **`logger.info(...)`**: Logs a success message if the update is successful.
*   **`catch (error) { ... }`**: If an error occurs during the organization update:
    *   **`logger.error(...)`**: Logs the error with relevant context.
    *   **`// Don't throw - the subscription was created successfully, just log the error`**: This comment explicitly states the design decision: a failure to update the organization should not prevent the overall process from completing (as the subscription itself was handled).

---

##### **Post-Upsert Logging**

```typescript
  logger.info('[subscription.created] Upserted enterprise subscription', {
    subscriptionId: subscriptionRow.id,
    referenceId: subscriptionRow.referenceId,
    plan: subscriptionRow.plan,
    status: subscriptionRow.status,
    monthlyPrice,
    seats,
    note: 'Seats from metadata, Stripe quantity set to 1',
  })
```

A final informational log message confirming that the enterprise subscription has been successfully upserted into the database, including key details for quick review.
*   `note: 'Seats from metadata, Stripe quantity set to 1'`: This is an important detail. It indicates that the "seats" quantity is managed via custom metadata on the Stripe subscription, not via Stripe's built-in `quantity` field for subscription items (which might be set to `1` regardless of the actual seat count).

---

##### **Email Notification**

```typescript
  try {
    const userDetails = await db
      .select({
        id: user.id,
        name: user.name,
        email: user.email,
      })
      .from(user)
      .where(eq(user.stripeCustomerId, stripeCustomerId))
      .limit(1)

    const orgDetails = await db
      .select({
        id: organization.id,
        name: organization.name,
      })
      .from(organization)
      .where(eq(organization.id, referenceId))
      .limit(1)

    if (userDetails.length > 0 && orgDetails.length > 0) {
      const user = userDetails[0]
      const org = orgDetails[0]

      const html = await renderEnterpriseSubscriptionEmail(user.name || user.email, user.email)

      const emailResult = await sendEmail({
        to: user.email,
        subject: getEmailSubject('enterprise-subscription'),
        html,
        from: getFromEmailAddress(),
        emailType: 'transactional',
      })

      if (emailResult.success) {
        logger.info('[subscription.created] Enterprise subscription email sent successfully', {
          userId: user.id,
          email: user.email,
          organizationId: org.id,
          subscriptionId: subscriptionRow.id,
        })
      } else {
        logger.warn('[subscription.created] Failed to send enterprise subscription email', {
          userId: user.id,
          email: user.email,
          error: emailResult.message,
        })
      }
    } else {
      logger.warn(
        '[subscription.created] Could not find user or organization for email notification',
        {
          userFound: userDetails.length > 0,
          orgFound: orgDetails.length > 0,
          stripeCustomerId,
          referenceId,
        }
      )
    }
  } catch (emailError) {
    logger.error('[subscription.created] Error sending enterprise subscription email', {
      error: emailError,
      stripeCustomerId,
      referenceId,
      subscriptionId: subscriptionRow.id,
    })
  }
}
```

Finally, this block attempts to send a confirmation email to the user associated with the subscription.

*   **`try { ... } catch (emailError) { ... }`**: This entire email sending process is also wrapped in a `try...catch` block. Email sending is generally considered non-critical for the core subscription process, so failures here should be logged but not stop the main flow.
*   **`const userDetails = await db .select(...) .from(user) .where(eq(user.stripeCustomerId, stripeCustomerId)) .limit(1)`**: Queries the `user` table to find the user associated with the `stripeCustomerId`. It selects their `id`, `name`, and `email`.
*   **`const orgDetails = await db .select(...) .from(organization) .where(eq(organization.id, referenceId)) .limit(1)`**: Queries the `organization` table to find the organization using the `referenceId`. It selects their `id` and `name`.
*   **`if (userDetails.length > 0 && orgDetails.length > 0) { ... }`**: Proceeds only if both a user and an organization were found.
    *   **`const user = userDetails[0]`** and **`const org = orgDetails[0]`**: Extracts the first (and only, due to `limit(1)`) found user and organization objects.
    *   **`const html = await renderEnterpriseSubscriptionEmail(user.name || user.email, user.email)`**: Generates the HTML content for the email. It passes the user's name (or email if name is missing) and email address.
    *   **`const emailResult = await sendEmail({ ... })`**: Calls the `sendEmail` function with all the necessary parameters: recipient (`to`), subject, HTML content, sender (`from`), and email type.
    *   **`if (emailResult.success) { ... } else { ... }`**: Checks the `emailResult` from `sendEmail`.
        *   Logs an `info` message if the email was sent successfully.
        *   Logs a `warn` message if the email failed to send, including the error message.
*   **`else { logger.warn('[subscription.created] Could not find user or organization...', { ... }) }`**: If either the user or organization details could not be retrieved from the database, a warning is logged, indicating why the email couldn't be sent.
*   **`catch (emailError) { logger.error('[subscription.created] Error sending enterprise subscription email', { ... }) }`**: Catches any unexpected errors during the email sending process (e.g., issues with the email service itself) and logs them.

---

This comprehensive breakdown covers the purpose of the file, simplifies the logic, and explains each significant line, providing a deep understanding of its functionality within the application.