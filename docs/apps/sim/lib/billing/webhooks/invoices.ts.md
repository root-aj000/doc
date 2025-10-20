This TypeScript file acts as a critical backend component for handling Stripe invoice-related webhooks. Its primary purpose is to integrate Stripe's billing events with an internal database, managing user subscriptions, usage tracking, and billing states. Specifically, it processes events like successful payments, failed payments, and invoice finalization (which often triggers overage calculations).

The core responsibilities of this file include:
1.  **Tracking Usage and Overage:** Monitoring user-specific usage and calculating any overage costs incurred beyond their plan limits.
2.  **Billing Overage:** Creating new Stripe invoices for calculated overages.
3.  **Managing Billing Status:** Blocking users from accessing features if their payments fail and unblocking them upon successful payment.
4.  **Resetting Usage:** Clearing usage metrics at the end of a billing cycle to prepare for the next period.
5.  **Handling Different Plan Types:** Adapting its logic for individual (Pro), team, and enterprise plans, which have different billing behaviors (e.g., team plans aggregate usage across members, enterprise plans might not have overages).

---

### Key Concepts and Dependencies

Before diving into the code, let's understand some foundational elements:

*   **Drizzle ORM:** This file heavily uses `drizzle-orm` for database interactions. Drizzle provides a type-safe way to build SQL queries.
    *   `db`: The Drizzle database connection instance.
    *   `member`, `subscription as subscriptionTable`, `userStats`: These are Drizzle "schema" objects representing tables in the database.
        *   `subscriptionTable`: Stores information about user subscriptions, including their Stripe Subscription ID, plan type, and an internal `referenceId` (which could be a `userId` or `organizationId`).
        *   `member`: Links users (`userId`) to organizations (`organizationId`). Used for team plans.
        *   `userStats`: Stores usage-related statistics for individual users, such as `currentPeriodCost`, `lastPeriodCost`, `billedOverageThisPeriod`, `proPeriodCostSnapshot`, and `billingBlocked` status.
    *   `eq`, `inArray`: Drizzle functions for `WHERE` clause conditions (equality and checking if a value is in an array).
*   **Stripe:** The payment processing platform. This file listens for webhooks (events) from Stripe.
    *   `Stripe.Event`, `Stripe.Invoice`, `Stripe.Subscription`, `Stripe.Customer`, `Stripe.PaymentMethod`: TypeScript types for Stripe objects.
    *   `requireStripeClient`: A utility to get an authenticated Stripe client instance for making API calls.
*   **`calculateSubscriptionOverage`**: An external function (from `@/lib/billing/core/billing`) that calculates the *total* usage overage for a given subscription based on internal usage metrics and plan rules.
*   **Logging:** `createLogger` from `@/lib/logs/console/logger` is used for structured logging, which is crucial for debugging and monitoring production systems.

---

### Detailed Code Explanation

```typescript
import { db } from '@sim/db'
import { member, subscription as subscriptionTable, userStats } from '@sim/db/schema'
import { eq, inArray } from 'drizzle-orm'
import type Stripe from 'stripe'
import { calculateSubscriptionOverage } from '@/lib/billing/core/billing'
import { requireStripeClient } from '@/lib/billing/stripe-client'
import { createLogger } from '@/lib/logs/console/logger'
```
These lines import all necessary modules and types:
*   `db`: The database client instance, likely configured for Drizzle ORM.
*   `member`, `subscription as subscriptionTable`, `userStats`: Drizzle schema definitions for the `member`, `subscription`, and `userStats` tables in the database. `subscription` is aliased to `subscriptionTable` to avoid naming conflicts if a variable named `subscription` is used later.
*   `eq`, `inArray`: Drizzle ORM functions used to build `WHERE` clauses in database queries (e.g., `WHERE column = value` or `WHERE column IN (value1, value2)`).
*   `Stripe`: The type definition for the Stripe client and its objects, imported solely for type checking.
*   `calculateSubscriptionOverage`: A function imported from an internal billing library, responsible for determining how much usage overage a subscription has accumulated.
*   `requireStripeClient`: A function to get a configured Stripe API client, enabling interaction with Stripe's services.
*   `createLogger`: A utility to create a logger instance for structured logging.

```typescript
const logger = createLogger('StripeInvoiceWebhooks')
```
Initializes a logger instance specifically for this file, making it easier to filter logs related to Stripe invoice webhooks.

```typescript
const OVERAGE_INVOICE_TYPES = new Set<string>([
  'overage_billing',
  'overage_threshold_billing',
  'overage_threshold_billing_org',
])
```
Defines a `Set` containing specific string identifiers. These identifiers are expected to be used in the `metadata.type` field of Stripe invoices to flag them as related to overage billing. Using a `Set` allows for very efficient checking (`has()` method) if an invoice type is one of these predefined overage types.

```typescript
function parseDecimal(value: string | number | null | undefined): number {
  if (value === null || value === undefined) return 0
  return Number.parseFloat(value.toString())
}
```
A helper function to safely convert various input types (string, number, null, undefined) into a floating-point number.
*   `if (value === null || value === undefined) return 0`: If the input is `null` or `undefined`, it defaults to `0` to prevent errors.
*   `return Number.parseFloat(value.toString())`: Otherwise, it converts the value to a string first (to handle both numbers and strings consistently) and then uses `Number.parseFloat` to parse it into a floating-point number.

```typescript
/**
 * Get total billed overage for a subscription, handling team vs individual plans
 * For team plans: sums billedOverageThisPeriod across all members
 * For other plans: gets billedOverageThisPeriod for the user
 */
export async function getBilledOverageForSubscription(sub: {
  plan: string | null
  referenceId: string
}): Promise<number> {
  let billedOverage = 0

  if (sub.plan === 'team') {
    const members = await db
      .select({ userId: member.userId })
      .from(member)
      .where(eq(member.organizationId, sub.referenceId))

    const memberIds = members.map((m) => m.userId)

    if (memberIds.length > 0) {
      const memberStatsRows = await db
        .select({
          userId: userStats.userId,
          billedOverageThisPeriod: userStats.billedOverageThisPeriod,
        })
        .from(userStats)
        .where(inArray(userStats.userId, memberIds))

      for (const stats of memberStatsRows) {
        billedOverage += parseDecimal(stats.billedOverageThisPeriod)
      }
    }
  } else { // Handles 'pro', 'individual', etc.
    const userStatsRecords = await db
      .select({ billedOverageThisPeriod: userStats.billedOverageThisPeriod })
      .from(userStats)
      .where(eq(userStats.userId, sub.referenceId))
      .limit(1)

    if (userStatsRecords.length > 0) {
      billedOverage = parseDecimal(userStatsRecords[0].billedOverageThisPeriod)
    }
  }

  return billedOverage
}
```
This asynchronous function retrieves the total overage amount that has *already been billed* for a given subscription *during the current billing period*. This is important for calculating *remaining* overage to bill.

*   `sub: { plan: string | null; referenceId: string }`: The input `sub` object contains the `plan` type (e.g., 'team', 'pro') and a `referenceId` (which is `organizationId` for team plans, `userId` for individual plans).
*   `let billedOverage = 0`: Initializes a variable to store the calculated billed overage.
*   **`if (sub.plan === 'team')`**:
    *   `db.select({ userId: member.userId }).from(member).where(eq(member.organizationId, sub.referenceId))`: Fetches all user IDs (`userId`) that belong to the organization identified by `sub.referenceId`.
    *   `const memberIds = members.map((m) => m.userId)`: Extracts just the `userId` values into an array.
    *   `if (memberIds.length > 0)`: Proceeds only if the organization has members.
    *   `db.select({ ... }).from(userStats).where(inArray(userStats.userId, memberIds))`: Retrieves the `billedOverageThisPeriod` for each of these members from the `userStats` table.
    *   `for (const stats of memberStatsRows) { billedOverage += parseDecimal(stats.billedOverageThisPeriod) }`: Iterates through the results and sums up the `billedOverageThisPeriod` for all members using `parseDecimal` for safe conversion.
*   **`else`** (for non-team plans, e.g., 'pro' or 'individual'):
    *   `db.select({ billedOverageThisPeriod: userStats.billedOverageThisPeriod }).from(userStats).where(eq(userStats.userId, sub.referenceId)).limit(1)`: Fetches the `billedOverageThisPeriod` for a single user identified by `sub.referenceId`. `limit(1)` is used because we expect only one record per user.
    *   `if (userStatsRecords.length > 0) { billedOverage = parseDecimal(userStatsRecords[0].billedOverageThisPeriod) }`: If a record is found, the `billedOverageThisPeriod` is assigned to `billedOverage`.
*   `return billedOverage`: Returns the total billed overage for the subscription.

```typescript
export async function resetUsageForSubscription(sub: { plan: string | null; referenceId: string }) {
  if (sub.plan === 'team' || sub.plan === 'enterprise') {
    const membersRows = await db
      .select({ userId: member.userId })
      .from(member)
      .where(eq(member.organizationId, sub.referenceId))

    for (const m of membersRows) {
      const currentStats = await db
        .select({ current: userStats.currentPeriodCost })
        .from(userStats)
        .where(eq(userStats.userId, m.userId))
        .limit(1)
      if (currentStats.length > 0) {
        const current = currentStats[0].current || '0'
        await db
          .update(userStats)
          .set({
            lastPeriodCost: current,
            currentPeriodCost: '0',
            billedOverageThisPeriod: '0',
          })
          .where(eq(userStats.userId, m.userId))
      }
    }
  } else { // Handles 'pro', 'individual', etc.
    const currentStats = await db
      .select({
        current: userStats.currentPeriodCost,
        snapshot: userStats.proPeriodCostSnapshot,
      })
      .from(userStats)
      .where(eq(userStats.userId, sub.referenceId))
      .limit(1)
    if (currentStats.length > 0) {
      // For Pro plans, combine current + snapshot for lastPeriodCost, then clear both
      const current = Number.parseFloat(currentStats[0].current?.toString() || '0')
      const snapshot = Number.parseFloat(currentStats[0].snapshot?.toString() || '0')
      const totalLastPeriod = (current + snapshot).toString()

      await db
        .update(userStats)
        .set({
          lastPeriodCost: totalLastPeriod,
          currentPeriodCost: '0',
          proPeriodCostSnapshot: '0', // Clear snapshot at period end
          billedOverageThisPeriod: '0', // Clear threshold billing tracker at period end
        })
        .where(eq(userStats.userId, sub.referenceId))
    }
  }
}
```
This asynchronous function resets usage tracking metrics for a subscription at the end of a billing period. The logic differs based on the plan type.

*   `sub: { plan: string | null; referenceId: string }`: Same input structure as `getBilledOverageForSubscription`.
*   **`if (sub.plan === 'team' || sub.plan === 'enterprise')`**:
    *   `db.select({ userId: member.userId }).from(member).where(eq(member.organizationId, sub.referenceId))`: Retrieves all member IDs for the organization.
    *   `for (const m of membersRows)`: Iterates through each member.
        *   `db.select({ current: userStats.currentPeriodCost }).from(userStats).where(eq(userStats.userId, m.userId)).limit(1)`: Gets the `currentPeriodCost` for the individual member.
        *   `const current = currentStats[0].current || '0'`: Safely gets the `currentPeriodCost` string.
        *   `db.update(userStats).set({ ... }).where(eq(userStats.userId, m.userId))`: Updates the member's `userStats`:
            *   `lastPeriodCost`: Set to the `currentPeriodCost` from the just-ended period.
            *   `currentPeriodCost`: Reset to `'0'`.
            *   `billedOverageThisPeriod`: Reset to `'0'`.
*   **`else`** (for non-team/enterprise plans, e.g., 'pro', 'individual'):
    *   `db.select({ current: userStats.currentPeriodCost, snapshot: userStats.proPeriodCostSnapshot }).from(userStats).where(eq(userStats.userId, sub.referenceId)).limit(1)`: Fetches `currentPeriodCost` and `proPeriodCostSnapshot` for the single user. The snapshot likely tracks an included usage or a specific metric for 'Pro' plans.
    *   `const current = Number.parseFloat(currentStats[0].current?.toString() || '0')`: Safely converts `currentPeriodCost` to a number.
    *   `const snapshot = Number.parseFloat(currentStats[0].snapshot?.toString() || '0')`: Safely converts `proPeriodCostSnapshot` to a number.
    *   `const totalLastPeriod = (current + snapshot).toString()`: For these plans, the `lastPeriodCost` is the sum of the `currentPeriodCost` and the `proPeriodCostSnapshot` from the just-ended period.
    *   `db.update(userStats).set({ ... }).where(eq(userStats.userId, sub.referenceId))`: Updates the user's `userStats`:
        *   `lastPeriodCost`: Set to the combined `totalLastPeriod` value.
        *   `currentPeriodCost`: Reset to `'0'`.
        *   `proPeriodCostSnapshot`: Reset to `'0'`.
        *   `billedOverageThisPeriod`: Reset to `'0'`.

```typescript
/**
 * Handle invoice payment succeeded webhook
 * We unblock any previously blocked users for this subscription.
 */
export async function handleInvoicePaymentSucceeded(event: Stripe.Event) {
  try {
    const invoice = event.data.object as Stripe.Invoice
```
This function is an asynchronous handler for Stripe's `invoice.payment_succeeded` webhook event. Its goal is to unblock users whose subscription payments have now been successfully processed.
*   `try...catch`: Encapsulates the logic to gracefully handle any errors during processing.
*   `const invoice = event.data.object as Stripe.Invoice`: Extracts the `Invoice` object from the Stripe webhook `event.data.object`.

```typescript
    const subscription = invoice.parent?.subscription_details?.subscription
    const stripeSubscriptionId = typeof subscription === 'string' ? subscription : subscription?.id
    if (!stripeSubscriptionId) {
      logger.info('No subscription found on invoice; skipping payment succeeded handler', {
        invoiceId: invoice.id,
      })
      return
    }
```
*   `const subscription = invoice.parent?.subscription_details?.subscription`: Attempts to get the associated subscription details from the invoice. Stripe's `invoice` object can have subscription information nested.
*   `const stripeSubscriptionId = typeof subscription === 'string' ? subscription : subscription?.id`: Determines the Stripe subscription ID. It could be a direct string or an object from which the `id` needs to be extracted.
*   `if (!stripeSubscriptionId) { ... return }`: If no `stripeSubscriptionId` can be determined, it logs an informational message and exits, as there's no subscription to manage.

```typescript
    const records = await db
      .select()
      .from(subscriptionTable)
      .where(eq(subscriptionTable.stripeSubscriptionId, stripeSubscriptionId))
      .limit(1)

    if (records.length === 0) return
    const sub = records[0]
```
*   `db.select().from(subscriptionTable).where(eq(subscriptionTable.stripeSubscriptionId, stripeSubscriptionId)).limit(1)`: Queries the internal `subscriptionTable` to find the corresponding subscription record using the `stripeSubscriptionId`.
*   `if (records.length === 0) return`: If no matching subscription is found in the database, it exits.
*   `const sub = records[0]`: Stores the found internal subscription record.

```typescript
    // Only reset usage here if the tenant was previously blocked; otherwise invoice.created already reset it
    let wasBlocked = false
    if (sub.plan === 'team' || sub.plan === 'enterprise') {
      const membersRows = await db
        .select({ userId: member.userId })
        .from(member)
        .where(eq(member.organizationId, sub.referenceId))
      const memberIds = membersRows.map((m) => m.userId)
      if (memberIds.length > 0) {
        const blockedRows = await db
          .select({ blocked: userStats.billingBlocked })
          .from(userStats)
          .where(inArray(userStats.userId, memberIds))

        wasBlocked = blockedRows.some((row) => !!row.blocked)
      }
    } else { // Handles 'pro', 'individual', etc.
      const row = await db
        .select({ blocked: userStats.billingBlocked })
        .from(userStats)
        .where(eq(userStats.userId, sub.referenceId))
        .limit(1)
      wasBlocked = row.length > 0 ? !!row[0].blocked : false
    }
```
This block determines if the subscription's users were previously blocked due to billing issues. This check is crucial because usage might have already been reset by an `invoice.created` event, and we only want to reset it *again* if a prior blocking action had occurred.
*   `let wasBlocked = false`: A flag to track if any user associated with the subscription was blocked.
*   **`if (sub.plan === 'team' || sub.plan === 'enterprise')`**:
    *   Retrieves all member `userId`s for the organization associated with the `sub.referenceId`.
    *   `if (memberIds.length > 0)`: If there are members, it fetches their `billingBlocked` status from `userStats`.
    *   `wasBlocked = blockedRows.some((row) => !!row.blocked)`: Checks if *any* member had `billingBlocked` set to `true`.
*   **`else`** (for individual plans):
    *   Fetches the `billingBlocked` status for the single user.
    *   `wasBlocked = row.length > 0 ? !!row[0].blocked : false`: Sets `wasBlocked` based on the user's `billingBlocked` status.

```typescript
    if (sub.plan === 'team' || sub.plan === 'enterprise') {
      const members = await db
        .select({ userId: member.userId })
        .from(member)
        .where(eq(member.organizationId, sub.referenceId))
      const memberIds = members.map((m) => m.userId)

      if (memberIds.length > 0) {
        await db
          .update(userStats)
          .set({ billingBlocked: false })
          .where(inArray(userStats.userId, memberIds))
      }
    } else { // Handles 'pro', 'individual', etc.
      await db
        .update(userStats)
        .set({ billingBlocked: false })
        .where(eq(userStats.userId, sub.referenceId))
    }
```
This section proceeds to unblock the users associated with the subscription, regardless of their prior `wasBlocked` state (as a payment succeeded event should always unblock).
*   **`if (sub.plan === 'team' || sub.plan === 'enterprise')`**:
    *   Retrieves all member `userId`s for the organization.
    *   `if (memberIds.length > 0)`: If there are members, it updates all their `userStats` records to set `billingBlocked` to `false`.
*   **`else`** (for individual plans):
    *   Updates the single user's `userStats` record to set `billingBlocked` to `false`.

```typescript
    if (wasBlocked) {
      await resetUsageForSubscription({ plan: sub.plan, referenceId: sub.referenceId })
    }
  } catch (error) {
    logger.error('Failed to handle invoice payment succeeded', { eventId: event.id, error })
    throw error
  }
}
```
*   `if (wasBlocked) { ... }`: *If* the users were previously blocked, this conditional call ensures `resetUsageForSubscription` is executed. This prevents usage metrics from being out of sync if they were blocked mid-period and then paid up.
*   `catch (error)`: Catches any errors during the process, logs them with the `eventId`, and re-throws the error to indicate that the webhook processing failed.

```typescript
/**
 * Handle invoice payment failed webhook
 * This is triggered when a user's payment fails for any invoice (subscription or overage)
 */
export async function handleInvoicePaymentFailed(event: Stripe.Event) {
  try {
    const invoice = event.data.object as Stripe.Invoice
```
This function handles the `invoice.payment_failed` Stripe webhook event. Its purpose is to block users associated with a subscription when a payment fails.
*   `try...catch`: Error handling.
*   `const invoice = event.data.object as Stripe.Invoice`: Extracts the `Invoice` object.

```typescript
    const invoiceType = invoice.metadata?.type
    const isOverageInvoice = !!(invoiceType && OVERAGE_INVOICE_TYPES.has(invoiceType))
    let stripeSubscriptionId: string | undefined

    if (isOverageInvoice) {
      // Overage invoices store subscription ID in metadata
      stripeSubscriptionId = invoice.metadata?.subscriptionId as string | undefined
    } else {
      // Regular subscription invoices have it in parent.subscription_details
      const subscription = invoice.parent?.subscription_details?.subscription
      stripeSubscriptionId = typeof subscription === 'string' ? subscription : subscription?.id
    }
```
This block determines the `stripeSubscriptionId` from the invoice, accounting for custom overage invoices.
*   `const invoiceType = invoice.metadata?.type`: Retrieves the custom `type` from the invoice's metadata.
*   `const isOverageInvoice = !!(invoiceType && OVERAGE_INVOICE_TYPES.has(invoiceType))`: Checks if this invoice is an "overage" type using the `OVERAGE_INVOICE_TYPES` set. `!!` converts the result to a boolean.
*   **`if (isOverageInvoice)`**: If it's an overage invoice, the `subscriptionId` is expected to be in `invoice.metadata.subscriptionId`.
*   **`else`**: For regular subscription invoices, the `subscriptionId` is extracted from `invoice.parent.subscription_details.subscription` (similar to the `payment_succeeded` handler).

```typescript
    if (!stripeSubscriptionId) {
      logger.info('No subscription found on invoice; skipping payment failed handler', {
        invoiceId: invoice.id,
        isOverageInvoice,
      })
      return
    }
```
If no `stripeSubscriptionId` can be found, it logs and exits.

```typescript
    const customerId = invoice.customer as string
    const failedAmount = invoice.amount_due / 100 // Convert from cents to dollars
    const billingPeriod = invoice.metadata?.billingPeriod || 'unknown'
    const attemptCount = invoice.attempt_count || 1

    logger.warn('Invoice payment failed', {
      invoiceId: invoice.id,
      customerId,
      failedAmount,
      billingPeriod,
      attemptCount,
      customerEmail: invoice.customer_email,
      hostedInvoiceUrl: invoice.hosted_invoice_url,
      isOverageInvoice,
      invoiceType: isOverageInvoice ? 'overage' : 'subscription',
    })
```
This section logs a warning about the failed payment with various relevant details for debugging and monitoring.
*   `customerId`, `failedAmount` (converted to dollars), `billingPeriod`, `attemptCount`: Extracts these crucial details from the Stripe invoice.
*   `logger.warn(...)`: Logs the failure with context.

```typescript
    // Block users after first payment failure
    if (attemptCount >= 1) {
      logger.error('Payment failure - blocking users', {
        invoiceId: invoice.id,
        customerId,
        attemptCount,
        isOverageInvoice,
        stripeSubscriptionId,
      })

      const records = await db
        .select()
        .from(subscriptionTable)
        .where(eq(subscriptionTable.stripeSubscriptionId, stripeSubscriptionId))
        .limit(1)

      if (records.length > 0) {
        const sub = records[0]
        if (sub.plan === 'team' || sub.plan === 'enterprise') {
          const members = await db
            .select({ userId: member.userId })
            .from(member)
            .where(eq(member.organizationId, sub.referenceId))
          const memberIds = members.map((m) => m.userId)

          if (memberIds.length > 0) {
            await db
              .update(userStats)
              .set({ billingBlocked: true })
              .where(inArray(userStats.userId, memberIds))
          }
          logger.info('Blocked team/enterprise members due to payment failure', {
            organizationId: sub.referenceId,
            memberCount: members.length,
            isOverageInvoice,
          })
        } else { // Handles 'pro', 'individual', etc.
          await db
            .update(userStats)
            .set({ billingBlocked: true })
            .where(eq(userStats.userId, sub.referenceId))
          logger.info('Blocked user due to payment failure', {
            userId: sub.referenceId,
            isOverageInvoice,
          })
        }
      } else {
        logger.warn('Subscription not found in database for failed payment', {
          stripeSubscriptionId,
          invoiceId: invoice.id,
        })
      }
    }
  } catch (error) {
    logger.error('Failed to handle invoice payment failed', {
      eventId: event.id,
      error,
    })
    throw error // Re-throw to signal webhook failure
  }
}
```
This is the core blocking logic for payment failures.
*   `if (attemptCount >= 1)`: The system decides to block users if there has been at least one payment attempt and it failed.
*   `logger.error('Payment failure - blocking users', { ... })`: Logs the initiation of the blocking process.
*   `db.select().from(subscriptionTable)...`: Retrieves the internal `subscriptionTable` record.
*   `if (records.length > 0)`: If a subscription is found:
    *   **`if (sub.plan === 'team' || sub.plan === 'enterprise')`**:
        *   Retrieves all member `userId`s for the organization.
        *   `if (memberIds.length > 0)`: Updates all members' `userStats` to set `billingBlocked` to `true`.
        *   Logs the blocking action.
    *   **`else`** (for individual plans):
        *   Updates the single user's `userStats` to set `billingBlocked` to `true`.
        *   Logs the blocking action.
*   `else { logger.warn('Subscription not found...') }`: If the subscription is not found in the database, it logs a warning.
*   `catch (error)`: Catches errors, logs them, and re-throws to signal webhook failure to Stripe (which might trigger retries).

```typescript
/**
 * Handle base invoice finalized → create a separate overage-only invoice
 * Note: Enterprise plans no longer have overages
 */
export async function handleInvoiceFinalized(event: Stripe.Event) {
  try {
    const invoice = event.data.object as Stripe.Invoice
    // Only run for subscription renewal invoices (cycle boundary)
    const subscription = invoice.parent?.subscription_details?.subscription
    const stripeSubscriptionId = typeof subscription === 'string' ? subscription : subscription?.id
    if (!stripeSubscriptionId) {
      logger.info('No subscription found on invoice; skipping finalized handler', {
        invoiceId: invoice.id,
      })
      return
    }
    if (invoice.billing_reason && invoice.billing_reason !== 'subscription_cycle') return
```
This asynchronous function handles the `invoice.finalized` Stripe webhook event. This is the most complex handler, responsible for calculating and billing overages, and then resetting usage for the new billing cycle.
*   `try...catch`: Error handling.
*   `const invoice = event.data.object as Stripe.Invoice`: Extracts the `Invoice` object.
*   `const stripeSubscriptionId = ...`: Extracts the `stripeSubscriptionId` (similar to previous handlers).
*   `if (!stripeSubscriptionId) { ... return }`: Logs and exits if no `stripeSubscriptionId` is found.
*   `if (invoice.billing_reason && invoice.billing_reason !== 'subscription_cycle') return`: **Crucially**, this handler *only* proceeds if the invoice's `billing_reason` is `subscription_cycle`. This ensures it only runs at the end of a billing period for subscription renewals, not for one-off invoices or other reasons.

```typescript
    const records = await db
      .select()
      .from(subscriptionTable)
      .where(eq(subscriptionTable.stripeSubscriptionId, stripeSubscriptionId))
      .limit(1)

    if (records.length === 0) return
    const sub = records[0]

    // Enterprise plans have no overages - reset usage and exit
    if (sub.plan === 'enterprise') {
      await resetUsageForSubscription({ plan: sub.plan, referenceId: sub.referenceId })
      return
    }
```
*   Retrieves the internal `subscriptionTable` record.
*   `if (records.length === 0) return`: Exits if no matching subscription is found.
*   `const sub = records[0]`: Stores the found subscription record.
*   **`if (sub.plan === 'enterprise') { ... return }`**: Enterprise plans are explicitly excluded from overage billing. If it's an enterprise plan, it just resets the usage and exits.

```typescript
    const stripe = requireStripeClient()
    const periodEnd =
      invoice.lines?.data?.[0]?.period?.end || invoice.period_end || Math.floor(Date.now() / 1000)
    const billingPeriod = new Date(periodEnd * 1000).toISOString().slice(0, 7)
```
*   `const stripe = requireStripeClient()`: Gets an instance of the Stripe API client.
*   `const periodEnd = ...`: Determines the end time of the billing period for the invoice. It tries multiple fields (`invoice.lines[0].period.end`, `invoice.period_end`) and falls back to the current time if necessary. This is a Unix timestamp.
*   `const billingPeriod = ...`: Converts `periodEnd` into a `YYYY-MM` string format for consistent metadata.

```typescript
    // Compute overage (only for team and pro plans), before resetting usage
    const totalOverage = await calculateSubscriptionOverage(sub)

    // Get already-billed overage from threshold billing
    const billedOverage = await getBilledOverageForSubscription(sub)

    // Only bill the remaining unbilled overage
    const remainingOverage = Math.max(0, totalOverage - billedOverage)

    logger.info('Invoice finalized overage calculation', {
      subscriptionId: sub.id,
      totalOverage,
      billedOverage,
      remainingOverage,
      billingPeriod,
    })
```
This is the core overage calculation logic.
*   `const totalOverage = await calculateSubscriptionOverage(sub)`: Calls the external function to determine the *total* usage overage that occurred during the billing period.
*   `const billedOverage = await getBilledOverageForSubscription(sub)`: Calls the local helper function to get the amount of overage that was *already billed* during the period (e.g., if there's a real-time "threshold billing" system).
*   `const remainingOverage = Math.max(0, totalOverage - billedOverage)`: Calculates the *net* overage that still needs to be billed. `Math.max(0, ...)` ensures it's not negative.
*   `logger.info(...)`: Logs the results of the overage calculation for transparency.

```typescript
    if (remainingOverage > 0) {
      const customerId = String(invoice.customer)
      const cents = Math.round(remainingOverage * 100)
      const itemIdemKey = `overage-item:${customerId}:${stripeSubscriptionId}:${billingPeriod}`
      const invoiceIdemKey = `overage-invoice:${customerId}:${stripeSubscriptionId}:${billingPeriod}`
```
If there's `remainingOverage` to bill, this block prepares to create a new Stripe invoice.
*   `customerId`, `cents` (converted from dollars), `itemIdemKey`, `invoiceIdemKey`: Extracts the customer ID and prepares idempotent keys. Idempotent keys are crucial for Stripe API calls to prevent duplicate actions if a request is retried. They ensure that if the same request is sent multiple times, Stripe processes it only once.

```typescript
      // Inherit billing settings from the Stripe subscription/customer for autopay
      const getPaymentMethodId = (
        pm: string | Stripe.PaymentMethod | null | undefined
      ): string | undefined => (typeof pm === 'string' ? pm : pm?.id)

      let collectionMethod: 'charge_automatically' | 'send_invoice' = 'charge_automatically'
      let defaultPaymentMethod: string | undefined
      try {
        const stripeSub = await stripe.subscriptions.retrieve(stripeSubscriptionId)
        if (stripeSub.collection_method === 'send_invoice') {
          collectionMethod = 'send_invoice'
        }
        const subDpm = getPaymentMethodId(stripeSub.default_payment_method)
        if (subDpm) {
          defaultPaymentMethod = subDpm
        } else if (collectionMethod === 'charge_automatically') {
          const custObj = await stripe.customers.retrieve(customerId)
          if (custObj && !('deleted' in custObj)) {
            const cust = custObj as Stripe.Customer
            const custDpm = getPaymentMethodId(cust.invoice_settings?.default_payment_method)
            if (custDpm) defaultPaymentMethod = custDpm
          }
        }
      } catch (e) {
        logger.error('Failed to retrieve subscription or customer', { error: e })
      }
```
This complex block attempts to inherit the billing settings (how the invoice should be paid and with which payment method) from the customer's *main* subscription or customer object in Stripe. This ensures the overage invoice behaves consistently with their existing billing setup.
*   `getPaymentMethodId`: A small helper to safely extract a payment method ID from various Stripe API response formats.
*   `collectionMethod`, `defaultPaymentMethod`: Variables initialized to default to automatic charging.
*   `try...catch`: Protects against failures during Stripe API calls.
*   `stripe.subscriptions.retrieve(stripeSubscriptionId)`: Retrieves the main Stripe subscription object.
*   It checks `stripeSub.collection_method` to see if the customer prefers `send_invoice` (manual payment) or `charge_automatically`.
*   It then tries to get the `default_payment_method` from the subscription.
*   If no default payment method is found on the subscription but `collectionMethod` is `charge_automatically`, it then attempts to retrieve the customer and get their `invoice_settings.default_payment_method`.
*   `logger.error(...)`: Logs if retrieving subscription/customer details fails.

```typescript
      // Create a draft invoice first so we can attach the item directly
      const overageInvoice = await stripe.invoices.create(
        {
          customer: customerId,
          collection_method: collectionMethod,
          auto_advance: false,
          ...(defaultPaymentMethod ? { default_payment_method: defaultPaymentMethod } : {}),
          metadata: {
            type: 'overage_billing',
            billingPeriod,
            subscriptionId: stripeSubscriptionId,
          },
        },
        { idempotencyKey: invoiceIdemKey }
      )

      // Attach the item to this invoice
      await stripe.invoiceItems.create(
        {
          customer: customerId,
          invoice: overageInvoice.id,
          amount: cents,
          currency: 'usd',
          description: `Usage Based Overage – ${billingPeriod}`,
          metadata: {
            type: 'overage_billing',
            billingPeriod,
            subscriptionId: stripeSubscriptionId,
          },
        },
        { idempotencyKey: itemIdemKey }
      )
```
These blocks create the actual overage invoice and add the overage amount as an item to it.
*   `stripe.invoices.create(...)`: Creates a *draft* Stripe invoice.
    *   `customer`: Links it to the correct customer.
    *   `collection_method`, `default_payment_method`: Inherited from the main subscription/customer.
    *   `auto_advance: false`: Ensures the invoice is *not* immediately finalized or sent/charged, giving us control.
    *   `metadata`: Custom metadata to identify this as an overage invoice, including the `billingPeriod` and `subscriptionId`.
    *   `{ idempotencyKey: invoiceIdemKey }`: Uses the generated idempotent key.
*   `stripe.invoiceItems.create(...)`: Adds an item to the *newly created draft* overage invoice.
    *   `customer`, `invoice`: Links the item to the correct customer and the draft invoice.
    *   `amount`, `currency`, `description`: Details of the overage item.
    *   `metadata`: Again, custom metadata for the item.
    *   `{ idempotencyKey: itemIdemKey }`: Uses the generated idempotent key for the invoice item.

```typescript
      // Finalize to trigger autopay (if charge_automatically and a PM is present)
      const draftId = overageInvoice.id
      if (typeof draftId !== 'string' || draftId.length === 0) {
        logger.error('Stripe created overage invoice without id; aborting finalize')
      } else {
        const finalized = await stripe.invoices.finalizeInvoice(draftId)
        // Some manual invoices may remain open after finalize; ensure we pay immediately when possible
        if (collectionMethod === 'charge_automatically' && finalized.status === 'open') {
          try {
            const payId = finalized.id
            if (typeof payId !== 'string' || payId.length === 0) {
              logger.error('Finalized invoice missing id')
              throw new Error('Finalized invoice missing id')
            }
            await stripe.invoices.pay(payId, {
              payment_method: defaultPaymentMethod,
            })
          } catch (payError) {
            logger.error('Failed to auto-pay overage invoice', {
              error: payError,
              invoiceId: finalized.id,
            })
          }
        }
      }
    }
```
This finalizes the overage invoice and attempts to pay it if the `collectionMethod` is automatic.
*   `const draftId = overageInvoice.id`: Gets the ID of the created draft invoice.
*   `if (typeof draftId !== 'string' || draftId.length === 0) { ... }`: Basic validation for the invoice ID.
*   `const finalized = await stripe.invoices.finalizeInvoice(draftId)`: Finalizes the draft invoice. This makes it a real, billable invoice in Stripe.
*   `if (collectionMethod === 'charge_automatically' && finalized.status === 'open')`: If the invoice is set to be charged automatically *and* it's still in an `open` status (meaning it wasn't immediately paid upon finalization, which can happen for various reasons), then:
    *   `try...catch`: For the payment attempt.
    *   `await stripe.invoices.pay(payId, { payment_method: defaultPaymentMethod })`: Explicitly attempts to pay the invoice using the determined `defaultPaymentMethod`.
    *   `catch (payError)`: Logs if auto-payment fails.

```typescript
    // Finally, reset usage for this subscription after overage handling
    await resetUsageForSubscription({ plan: sub.plan, referenceId: sub.referenceId })
  } catch (error) {
    logger.error('Failed to handle invoice finalized', { error })
    throw error
  }
}
```
*   `await resetUsageForSubscription({ plan: sub.plan, referenceId: sub.referenceId })`: **After all overage calculation and billing is complete**, the usage statistics for the subscription are reset for the *new* billing period using the `resetUsageForSubscription` helper. This prepares the system for tracking usage in the upcoming cycle.
*   `catch (error)`: Catches any errors during the entire `handleInvoiceFinalized` process, logs them, and re-throws the error to signal webhook failure.

---

### Summary

This file is a sophisticated webhook handler that ensures your internal user and subscription data (in `userStats`, `member`, `subscriptionTable`) remains synchronized with Stripe's billing events. It manages the lifecycle of usage tracking, from calculating and billing overages to resetting counters for the next period, and importantly, enforces billing-related access restrictions by blocking or unblocking users based on payment outcomes. The detailed handling of different plan types (`team`, `enterprise`, `pro`/`individual`) and the use of Stripe's idempotency keys demonstrate robust design for a complex billing system.