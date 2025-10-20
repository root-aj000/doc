This TypeScript file acts as a critical backend component for handling events related to user subscriptions, specifically reacting to Stripe webhooks for subscription creation and deletion. It integrates with a database (Drizzle ORM), a custom billing logic, and the Stripe API to ensure usage tracking is reset correctly and final overage charges are applied when subscriptions change status.

---

### **File Purpose and What It Does**

This file defines two main asynchronous functions: `handleSubscriptionCreated` and `handleSubscriptionDeleted`. These functions are designed to be called in response to webhook events from a payment processor like Stripe, specifically when a new subscription is established or an existing one is cancelled/deleted.

1.  **`handleSubscriptionCreated`**:
    *   **Purpose:** To detect if a user is transitioning from a free tier to a paid subscription plan.
    *   **Action:** If such a transition occurs, it resets the user's usage metrics in the system. This ensures that when a user starts paying, their usage counters (e.g., API calls, storage) begin fresh for their new billing cycle, preventing them from being billed for "free" usage or continuing with high usage without proper billing.

2.  **`handleSubscriptionDeleted`**:
    *   **Purpose:** To handle the final billing and usage reset when a subscription ends.
    *   **Action:** It calculates any remaining usage overages for the final billing period that haven't been billed yet. If there are unbilled overages, it creates a final invoice in Stripe to charge the customer. After billing (or if no billing is needed), it resets the user's usage metrics, similar to creation, but for cleanup.

In essence, this file ensures that the system's internal usage tracking and external billing via Stripe remain synchronized and accurate throughout the subscription lifecycle, especially during key transition points like starting a paid plan or ending a subscription.

---

### **Line-by-Line Explanation**

Let's break down the code in detail.

#### **Imports**

```typescript
import { db } from '@sim/db'
import { subscription } from '@sim/db/schema'
import { and, eq, ne } from 'drizzle-orm'
import { calculateSubscriptionOverage } from '@/lib/billing/core/billing'
import { requireStripeClient } from '@/lib/billing/stripe-client'
import { createLogger } from '@/lib/logs/console/logger'
import { getBilledOverageForSubscription, resetUsageForSubscription } from './invoices'
```

*   `import { db } from '@sim/db'`: Imports the database client instance, likely configured for Drizzle ORM, allowing the application to interact with its database.
*   `import { subscription } from '@sim/db/schema'`: Imports the Drizzle schema definition for the `subscription` table. This provides a type-safe way to reference columns and perform queries on the `subscription` table.
*   `import { and, eq, ne } from 'drizzle-orm'`: Imports helper functions from Drizzle ORM used for constructing database query conditions:
    *   `and`: Combines multiple conditions with a logical AND.
    *   `eq`: Checks if a column is *equal* to a value.
    *   `ne`: Checks if a column is *not equal* to a value.
*   `import { calculateSubscriptionOverage } from '@/lib/billing/core/billing'`: Imports a custom function responsible for calculating the total usage overage for a given subscription. This function contains the business logic for determining "extra" usage beyond a plan's limits.
*   `import { requireStripeClient } from '@/lib/billing/stripe-client'`: Imports a function that provides an initialized Stripe client instance. This client is used to interact with the Stripe API (e.g., creating invoices, retrieving subscription details).
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a function to create a logger instance. This is used for structured logging, helping with debugging and monitoring the application's behavior.
*   `import { getBilledOverageForSubscription, resetUsageForSubscription } from './invoices'`: Imports two functions from a local `./invoices` file:
    *   `getBilledOverageForSubscription`: Retrieves any overage that has already been billed for a subscription (e.g., through threshold billing or mid-period adjustments).
    *   `resetUsageForSubscription`: Clears or resets the accumulated usage data for a specific subscription and user.

#### **Logger Initialization**

```typescript
const logger = createLogger('StripeSubscriptionWebhooks')
```

*   `const logger = createLogger('StripeSubscriptionWebhooks')`: Initializes a logger instance with the name `'StripeSubscriptionWebhooks'`. This name helps identify logs originating from this specific part of the application.

#### **`handleSubscriptionCreated` Function**

```typescript
/**
 * Handle new subscription creation - reset usage if transitioning from free to paid
 */
export async function handleSubscriptionCreated(subscriptionData: {
  id: string
  referenceId: string
  plan: string | null
  status: string
}) {
  try {
    const otherActiveSubscriptions = await db
      .select()
      .from(subscription)
      .where(
        and(
          eq(subscription.referenceId, subscriptionData.referenceId),
          eq(subscription.status, 'active'),
          ne(subscription.id, subscriptionData.id) // Exclude current subscription
        )
      )

    const wasFreePreviously = otherActiveSubscriptions.length === 0
    const isPaidPlan =
      subscriptionData.plan === 'pro' ||
      subscriptionData.plan === 'team' ||
      subscriptionData.plan === 'enterprise'

    if (wasFreePreviously && isPaidPlan) {
      logger.info('Detected free -> paid transition, resetting usage', {
        subscriptionId: subscriptionData.id,
        referenceId: subscriptionData.referenceId,
        plan: subscriptionData.plan,
      })

      await resetUsageForSubscription({
        plan: subscriptionData.plan,
        referenceId: subscriptionData.referenceId,
      })

      logger.info('Successfully reset usage for free -> paid transition', {
        subscriptionId: subscriptionData.id,
        referenceId: subscriptionData.referenceId,
        plan: subscriptionData.plan,
      })
    } else {
      logger.info('No usage reset needed', {
        subscriptionId: subscriptionData.id,
        referenceId: subscriptionData.referenceId,
        plan: subscriptionData.plan,
        wasFreePreviously,
        isPaidPlan,
        otherActiveSubscriptionsCount: otherActiveSubscriptions.length,
      })
    }
  } catch (error) {
    logger.error('Failed to handle subscription creation usage reset', {
      subscriptionId: subscriptionData.id,
      referenceId: subscriptionData.referenceId,
      error,
    })
    throw error
  }
}
```

*   `export async function handleSubscriptionCreated(subscriptionData: { ... })`: Defines an asynchronous function `handleSubscriptionCreated` that accepts an object `subscriptionData`. This object contains details about the newly created subscription from the webhook, including its `id`, a unique `referenceId` (likely for the user or organization), the `plan` name, and `status`.
*   `try { ... } catch (error) { ... }`: A standard `try-catch` block to gracefully handle any errors that occur during the function's execution, preventing the process from crashing and ensuring errors are logged.
*   `const otherActiveSubscriptions = await db.select().from(subscription).where(...)`: This line queries the database to find other *active* subscriptions associated with the same `referenceId` (e.g., a user or organization), *excluding* the newly created subscription (`subscriptionData.id`).
    *   `.select()`: Selects all columns.
    *   `.from(subscription)`: Specifies the `subscription` table.
    *   `.where(and(...))`: Applies multiple conditions joined by `AND`:
        *   `eq(subscription.referenceId, subscriptionData.referenceId)`: The `referenceId` must match that of the new subscription.
        *   `eq(subscription.status, 'active')`: The other subscriptions must be currently active.
        *   `ne(subscription.id, subscriptionData.id)`: Crucially, it excludes the *current* subscription being handled, to only look for *previous* or *concurrent* active subscriptions.
*   `const wasFreePreviously = otherActiveSubscriptions.length === 0`: This boolean variable is `true` if no other active subscriptions were found for the `referenceId`. This logic assumes that if a user has no *other* active subscriptions, they were previously on a free plan or had no subscription at all.
*   `const isPaidPlan = ...`: This boolean variable checks if the newly created subscription's `plan` is one of the recognized paid plans (`'pro'`, `'team'`, or `'enterprise'`).
*   `if (wasFreePreviously && isPaidPlan) { ... }`: This conditional block executes if *both* conditions are met: the user was previously free (or had no active paid plan) AND the new subscription is for a paid plan. This signifies a "free-to-paid" transition.
    *   `logger.info('Detected free -> paid transition, resetting usage', { ... })`: Logs an informational message indicating that a free-to-paid transition has been detected, including relevant subscription details.
    *   `await resetUsageForSubscription({ ... })`: Calls the imported `resetUsageForSubscription` function to clear or reset the usage data for this user/organization. This is crucial for starting a fresh billing cycle on the new paid plan.
    *   `logger.info('Successfully reset usage for free -> paid transition', { ... })`: Logs a success message after the usage reset.
*   `else { ... }`: If the `if` condition is not met (i.e., not a free-to-paid transition), this block executes.
    *   `logger.info('No usage reset needed', { ... })`: Logs an informational message explaining why usage reset was not performed, providing context with `wasFreePreviously`, `isPaidPlan`, and the count of `otherActiveSubscriptions`.
*   `catch (error) { ... }`: If any error occurs within the `try` block:
    *   `logger.error('Failed to handle subscription creation usage reset', { ... })`: Logs an error message with details of the failing subscription and the error itself.
    *   `throw error`: Re-throws the error, indicating that the webhook processing failed and potentially triggering a retry from the webhook sender (e.g., Stripe).

#### **`handleSubscriptionDeleted` Function**

```typescript
/**
 * Handle subscription deletion/cancellation - bill for final period overages
 * This fires when a subscription reaches its cancel_at_period_end date or is cancelled immediately
 */
export async function handleSubscriptionDeleted(subscription: {
  id: string
  plan: string | null
  referenceId: string
  stripeSubscriptionId: string | null
  seats?: number | null
}) {
  try {
    const stripeSubscriptionId = subscription.stripeSubscriptionId || ''

    logger.info('Processing subscription deletion', {
      stripeSubscriptionId,
      subscriptionId: subscription.id,
    })

    // Calculate overage for the final billing period
    const totalOverage = await calculateSubscriptionOverage(subscription)
    const stripe = requireStripeClient()

    // Enterprise plans have no overages - just reset usage
    if (subscription.plan === 'enterprise') {
      await resetUsageForSubscription({
        plan: subscription.plan,
        referenceId: subscription.referenceId,
      })
      return
    }

    // Get already-billed overage from threshold billing
    const billedOverage = await getBilledOverageForSubscription(subscription)

    // Only bill the remaining unbilled overage
    const remainingOverage = Math.max(0, totalOverage - billedOverage)

    logger.info('Subscription deleted overage calculation', {
      subscriptionId: subscription.id,
      totalOverage,
      billedOverage,
      remainingOverage,
    })

    // Create final overage invoice if needed
    if (remainingOverage > 0 && stripeSubscriptionId) {
      const stripeSubscription = await stripe.subscriptions.retrieve(stripeSubscriptionId)
      const customerId = stripeSubscription.customer as string
      const cents = Math.round(remainingOverage * 100)

      // Use the subscription end date for the billing period
      const endedAt = stripeSubscription.ended_at || Math.floor(Date.now() / 1000)
      const billingPeriod = new Date(endedAt * 1000).toISOString().slice(0, 7)

      const itemIdemKey = `final-overage-item:${customerId}:${stripeSubscriptionId}:${billingPeriod}`
      const invoiceIdemKey = `final-overage-invoice:${customerId}:${stripeSubscriptionId}:${billingPeriod}`

      try {
        // Create a one-time invoice for the final overage
        const overageInvoice = await stripe.invoices.create(
          {
            customer: customerId,
            collection_method: 'charge_automatically',
            auto_advance: true, // Auto-finalize and attempt payment
            description: `Final overage charges for ${subscription.plan} subscription (${billingPeriod})`,
            metadata: {
              type: 'final_overage_billing',
              billingPeriod,
              subscriptionId: stripeSubscriptionId,
              cancelledAt: stripeSubscription.canceled_at?.toString() || '',
            },
          },
          { idempotencyKey: invoiceIdemKey }
        )

        // Add the overage line item
        await stripe.invoiceItems.create(
          {
            customer: customerId,
            invoice: overageInvoice.id,
            amount: cents,
            currency: 'usd',
            description: `Usage overage for ${subscription.plan} plan (Final billing period)`,
            metadata: {
              type: 'final_usage_overage',
              usage: remainingOverage.toFixed(2),
              totalOverage: totalOverage.toFixed(2),
              billedOverage: billedOverage.toFixed(2),
              billingPeriod,
            },
          },
          { idempotencyKey: itemIdemKey }
        )

        // Finalize the invoice (this will trigger payment collection)
        if (overageInvoice.id) {
          await stripe.invoices.finalizeInvoice(overageInvoice.id)
        }

        logger.info('Created final overage invoice for cancelled subscription', {
          subscriptionId: subscription.id,
          stripeSubscriptionId,
          invoiceId: overageInvoice.id,
          totalOverage,
          billedOverage,
          remainingOverage,
          cents,
          billingPeriod,
        })
      } catch (invoiceError) {
        logger.error('Failed to create final overage invoice', {
          subscriptionId: subscription.id,
          stripeSubscriptionId,
          totalOverage,
          billedOverage,
          remainingOverage,
          error: invoiceError,
        })
        // Don't throw - we don't want to fail the webhook
      }
    } else {
      logger.info('No overage to bill for cancelled subscription', {
        subscriptionId: subscription.id,
        plan: subscription.plan,
      })
    }

    // Reset usage after billing
    await resetUsageForSubscription({
      plan: subscription.plan,
      referenceId: subscription.referenceId,
    })

    // Note: better-auth's Stripe plugin already updates status to 'canceled' before calling this handler
    // We only need to handle overage billing and usage reset

    logger.info('Successfully processed subscription cancellation', {
      subscriptionId: subscription.id,
      stripeSubscriptionId,
      totalOverage,
    })
  } catch (error) {
    logger.error('Failed to handle subscription deletion', {
      subscriptionId: subscription.id,
      stripeSubscriptionId: subscription.stripeSubscriptionId || '',
      error,
    })
    throw error // Re-throw to signal webhook failure for retry
  }
}
```

*   `export async function handleSubscriptionDeleted(subscription: { ... })`: Defines an asynchronous function `handleSubscriptionDeleted` that receives a `subscription` object. This object contains details about the subscription being deleted/cancelled, including its internal `id`, `plan`, `referenceId`, `stripeSubscriptionId`, and optionally `seats`.
*   `try { ... } catch (error) { ... }`: A `try-catch` block for error handling.
*   `const stripeSubscriptionId = subscription.stripeSubscriptionId || ''`: Extracts the Stripe subscription ID, defaulting to an empty string if it's null, for consistency.
*   `logger.info('Processing subscription deletion', { ... })`: Logs the start of the deletion processing.
*   `const totalOverage = await calculateSubscriptionOverage(subscription)`: Calls the external `calculateSubscriptionOverage` function to determine the total usage overage accumulated during the final billing period of the subscription.
*   `const stripe = requireStripeClient()`: Initializes the Stripe client, enabling communication with the Stripe API.
*   `if (subscription.plan === 'enterprise') { ... }`: Checks if the plan is 'enterprise'.
    *   `await resetUsageForSubscription({ ... })`: If it's an enterprise plan, it resets usage immediately. The comment `Enterprise plans have no overages - just reset usage` indicates a business rule that enterprise customers are not charged for overages.
    *   `return`: Exits the function early after resetting usage, as no further billing is needed for enterprise plans.
*   `const billedOverage = await getBilledOverageForSubscription(subscription)`: Retrieves any overage that has *already* been billed for this subscription. This is important to avoid double-billing if overages were partially billed mid-period (e.g., via threshold billing).
*   `const remainingOverage = Math.max(0, totalOverage - billedOverage)`: Calculates the overage that still needs to be billed. It's the `totalOverage` minus any `billedOverage`, ensuring the result is not negative using `Math.max(0, ...)`.
*   `logger.info('Subscription deleted overage calculation', { ... })`: Logs the calculated overage amounts for debugging and monitoring.
*   `if (remainingOverage > 0 && stripeSubscriptionId) { ... }`: This block executes only if there's a positive `remainingOverage` to bill AND a valid `stripeSubscriptionId` exists.
    *   `const stripeSubscription = await stripe.subscriptions.retrieve(stripeSubscriptionId)`: Fetches the full subscription object from Stripe using its ID. This is necessary to get the customer ID and the actual `ended_at` timestamp.
    *   `const customerId = stripeSubscription.customer as string`: Extracts the customer ID from the retrieved Stripe subscription.
    *   `const cents = Math.round(remainingOverage * 100)`: Converts the `remainingOverage` (which is likely in dollars or the primary currency unit) into cents (or the smallest currency unit), as Stripe API usually expects amounts in cents.
    *   `const endedAt = stripeSubscription.ended_at || Math.floor(Date.now() / 1000)`: Determines the end date of the subscription. It prefers the `ended_at` timestamp from Stripe; if not available, it uses the current time.
    *   `const billingPeriod = new Date(endedAt * 1000).toISOString().slice(0, 7)`: Formats the `endedAt` timestamp into a "YYYY-MM" string, representing the billing month.
    *   `const itemIdemKey = ...` and `const invoiceIdemKey = ...`: These create [idempotency keys](https://stripe.com/docs/api/idempotent_requests). Idempotency keys are crucial for webhooks: if a webhook is received multiple times (e.g., due to network issues and retries), Stripe will process the request only once, preventing duplicate invoices or charges.
    *   `try { ... } catch (invoiceError) { ... }`: An inner `try-catch` block specifically for the Stripe invoice creation process.
        *   `const overageInvoice = await stripe.invoices.create(...)`: Creates a new *draft* invoice in Stripe.
            *   `customer: customerId`: Associates the invoice with the correct customer.
            *   `collection_method: 'charge_automatically'`: Configures Stripe to automatically attempt to charge the customer's payment method on file.
            *   `auto_advance: true`: Tells Stripe to automatically finalize the invoice (make it collectible) and attempt payment.
            *   `description`, `metadata`: Provides useful context and labels for the invoice in Stripe.
        *   `await stripe.invoiceItems.create(...)`: Adds an item to the newly created invoice. This item represents the actual overage charge.
            *   `customer: customerId`, `invoice: overageInvoice.id`: Links the invoice item to the customer and the specific invoice.
            *   `amount: cents`, `currency: 'usd'`: Specifies the charge amount and currency.
            *   `description`, `metadata`: Provides details for the line item.
        *   `if (overageInvoice.id) { await stripe.invoices.finalizeInvoice(overageInvoice.id) }`: Explicitly finalizes the invoice. While `auto_advance: true` *should* do this, explicitly calling `finalizeInvoice` ensures it happens, especially if `auto_advance` isn't fully reliable or for clarity. Finalizing triggers the payment attempt.
        *   `logger.info('Created final overage invoice for cancelled subscription', { ... })`: Logs success details about the created invoice.
        *   `catch (invoiceError) { ... }`: If an error occurs during invoice creation or finalization:
            *   `logger.error('Failed to create final overage invoice', { ... })`: Logs the error.
            *   `// Don't throw - we don't want to fail the webhook`: This is an important decision. It means that even if final billing fails, the rest of the webhook process (like usage reset) will continue, and the webhook won't signal failure. This is often done for non-critical failures or where manual intervention is preferred over repeated retries.
*   `else { ... }`: If `remainingOverage` is not greater than 0, or `stripeSubscriptionId` is missing.
    *   `logger.info('No overage to bill for cancelled subscription', { ... })`: Logs that no overage billing was required.
*   `await resetUsageForSubscription({ ... })`: Regardless of whether an invoice was created, this line calls `resetUsageForSubscription` to clear the usage data for the cancelled subscription. This is a cleanup step.
*   `logger.info('Successfully processed subscription cancellation', { ... })`: Logs the successful completion of the deletion handler.
*   `catch (error) { ... }`: The main `try-catch` block for the entire function.
    *   `logger.error('Failed to handle subscription deletion', { ... })`: Logs any unhandled errors.
    *   `throw error`: Re-throws the error, indicating a critical failure in the webhook processing and potentially prompting Stripe to retry the webhook.

---

This detailed breakdown clarifies the functionality of each part of the code, emphasizing the logic behind subscription lifecycle management, usage tracking, and integration with a third-party billing provider like Stripe.