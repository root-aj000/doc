This TypeScript file, `billing/core/threshold-billing.ts`, plays a crucial role in a billing system. Its primary purpose is to **monitor user and organization usage against predefined thresholds and automatically generate and charge invoices via Stripe if the unbilled overage exceeds that threshold.**

In simpler terms, it's an automated "bill me now" system for unexpected high usage, preventing customers from accumulating massive unbilled costs until their regular billing cycle.

---

## Detailed Explanation

Let's break down the code section by section.

### 1. Imports and Setup

This section brings in all the necessary tools and configurations.

```typescript
import { db } from '@sim/db'
import { member, subscription, userStats } from '@sim/db/schema'
import { and, eq, inArray, sql } from 'drizzle-orm'
import type Stripe from 'stripe'
import { DEFAULT_OVERAGE_THRESHOLD } from '@/lib/billing/constants'
import { calculateSubscriptionOverage, getPlanPricing } from '@/lib/billing/core/billing'
import { getHighestPrioritySubscription } from '@/lib/billing/core/subscription'
import { requireStripeClient } from '@/lib/billing/stripe-client'
import { env } from '@/lib/env'
import { createLogger } from '@/lib/logs/console/logger'

const logger = createLogger('ThresholdBilling')

const OVERAGE_THRESHOLD = env.OVERAGE_THRESHOLD_DOLLARS || DEFAULT_OVERAGE_THRESHOLD
```

*   **`import { db } from '@sim/db'`**: Imports the Drizzle ORM database client, `db`, which is used to interact with the PostgreSQL database.
*   **`import { member, subscription, userStats } from '@sim/db/schema'`**: Imports the schema definitions for `member` (user-organization relationships), `subscription` (user/organization subscriptions), and `userStats` (usage statistics for users). These define the structure of tables in the database.
*   **`import { and, eq, inArray, sql } from 'drizzle-orm'`**: Imports specific functions from the Drizzle ORM library:
    *   `and`: Combines multiple `WHERE` clauses with an `AND` operator.
    *   `eq`: Checks for equality (`=`).
    *   `inArray`: Checks if a value is present in an array.
    *   `sql`: Allows writing raw SQL fragments, often used for database functions or atomic updates.
*   **`import type Stripe from 'stripe'`**: Imports the type definitions for the `stripe` library. This is used for type-checking and autocompletion when working with Stripe objects.
*   **`import { DEFAULT_OVERAGE_THRESHOLD } from '@/lib/billing/constants'`**: Imports a constant value, `DEFAULT_OVERAGE_THRESHOLD`, which provides a fallback for the overage threshold if an environment variable isn't set.
*   **`import { calculateSubscriptionOverage, getPlanPricing } from '@/lib/billing/core/billing'`**: Imports core billing logic functions:
    *   `calculateSubscriptionOverage`: Determines the total overage amount for a given subscription based on current usage.
    *   `getPlanPricing`: Retrieves pricing details for a specific plan.
*   **`import { getHighestPrioritySubscription } from '@/lib/billing/core/subscription'`**: Imports a utility function to find the most relevant (highest priority) active subscription for a user.
*   **`import { requireStripeClient } from '@/lib/billing/stripe-client'`**: Imports a function to initialize and provide an authenticated Stripe client, `stripe`, for interacting with the Stripe API.
*   **`import { env } from '@/lib/env'`**: Imports environment variables, allowing the application to read configuration from its environment (e.g., API keys, custom settings).
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a function to create a logger instance for structured logging.
*   **`const logger = createLogger('ThresholdBilling')`**: Initializes a logger instance specifically for this file, making it easier to track logs related to threshold billing.
*   **`const OVERAGE_THRESHOLD = env.OVERAGE_THRESHOLD_DOLLARS || DEFAULT_OVERAGE_THRESHOLD`**: Defines the `OVERAGE_THRESHOLD` constant. It first tries to read it from the environment variable `OVERAGE_THRESHOLD_DOLLARS`. If that variable is not set (e.g., `undefined` or `null`), it falls back to the `DEFAULT_OVERAGE_THRESHOLD` constant. This value represents the dollar amount of unbilled overage that must be reached before an invoice is triggered.

### 2. Helper Function: `parseDecimal`

This utility ensures that numerical values from various sources are handled consistently.

```typescript
function parseDecimal(value: string | number | null | undefined): number {
  if (value === null || value === undefined) return 0
  return Number.parseFloat(value.toString())
}
```

*   **`function parseDecimal(...)`**: Defines a function that safely converts a value into a floating-point number.
*   **`value: string | number | null | undefined`**: Specifies that the `value` parameter can be a string, a number, `null`, or `undefined`.
*   **`if (value === null || value === undefined) return 0`**: If the input `value` is `null` or `undefined`, the function immediately returns `0`, preventing errors from trying to parse non-existent values.
*   **`return Number.parseFloat(value.toString())`**: For other valid inputs, it first converts the `value` to a string using `toString()` (important for numbers passed as numbers or potentially other types) and then uses `Number.parseFloat()` to parse it into a floating-point number.

### 3. Core Function: `createAndFinalizeOverageInvoice`

This is a critical function responsible for interacting with the Stripe API to create, finalize, and attempt to pay an invoice.

```typescript
async function createAndFinalizeOverageInvoice(
  stripe: ReturnType<typeof requireStripeClient>,
  params: {
    customerId: string
    stripeSubscriptionId: string
    amountCents: number
    description: string
    itemDescription: string
    metadata: Record<string, string>
    idempotencyKey: string
  }
): Promise<string> {
  const getPaymentMethodId = (
    pm: string | Stripe.PaymentMethod | null | undefined
  ): string | undefined => (typeof pm === 'string' ? pm : pm?.id)

  let defaultPaymentMethod: string | undefined
  try {
    const stripeSub = await stripe.subscriptions.retrieve(params.stripeSubscriptionId)
    const subDpm = getPaymentMethodId(stripeSub.default_payment_method)
    if (subDpm) {
      defaultPaymentMethod = subDpm
    } else {
      const custObj = await stripe.customers.retrieve(params.customerId)
      if (custObj && !('deleted' in custObj)) {
        const cust = custObj as Stripe.Customer
        const custDpm = getPaymentMethodId(cust.invoice_settings?.default_payment_method)
        if (custDpm) defaultPaymentMethod = custDpm
      }
    }
  } catch (e) {
    logger.error('Failed to retrieve subscription or customer', { error: e })
  }

  const invoice = await stripe.invoices.create(
    {
      customer: params.customerId,
      collection_method: 'charge_automatically',
      auto_advance: false,
      description: params.description,
      metadata: params.metadata,
      ...(defaultPaymentMethod ? { default_payment_method: defaultPaymentMethod } : {}),
    },
    { idempotencyKey: `${params.idempotencyKey}-invoice` }
  )

  await stripe.invoiceItems.create(
    {
      customer: params.customerId,
      invoice: invoice.id,
      amount: params.amountCents,
      currency: 'usd',
      description: params.itemDescription,
      metadata: params.metadata,
    },
    { idempotencyKey: params.idempotencyKey }
  )

  if (invoice.id) {
    const finalized = await stripe.invoices.finalizeInvoice(invoice.id)

    if (finalized.status === 'open' && finalized.id) {
      try {
        await stripe.invoices.pay(finalized.id, {
          payment_method: defaultPaymentMethod,
        })
      } catch (payError) {
        logger.error('Failed to auto-pay threshold overage invoice', {
          error: payError,
          invoiceId: finalized.id,
        })
      }
    }
  }

  return invoice.id || ''
}
```

*   **`async function createAndFinalizeOverageInvoice(...)`**: An asynchronous function that takes the `stripe` client instance and a `params` object containing all the details needed for the invoice. It promises to return the `id` of the created Stripe invoice.
*   **`getPaymentMethodId` helper**: A small utility function nested inside. It safely extracts the ID of a Stripe payment method, which can either be a string (the ID directly) or a `Stripe.PaymentMethod` object.
*   **Retrieve Default Payment Method (`try...catch` block)**:
    *   **`const stripeSub = await stripe.subscriptions.retrieve(params.stripeSubscriptionId)`**: Fetches the full Stripe subscription object using the provided `stripeSubscriptionId`.
    *   **`const subDpm = getPaymentMethodId(stripeSub.default_payment_method)`**: Tries to get the default payment method ID directly from the subscription.
    *   If a `subDpm` is found, it's used.
    *   **`else { ... }`**: If no default payment method is on the subscription, it then tries to retrieve the customer object (`stripe.customers.retrieve(params.customerId)`).
    *   **`if (custObj && !('deleted' in custObj))`**: Checks if the customer object exists and hasn't been deleted.
    *   **`const custDpm = getPaymentMethodId(cust.invoice_settings?.default_payment_method)`**: Retrieves the default payment method from the customer's invoice settings.
    *   **`catch (e)`**: Logs any errors encountered during the retrieval of subscription or customer details.
*   **`const invoice = await stripe.invoices.create(...)`**: Creates a new invoice in Stripe.
    *   **`customer: params.customerId`**: Links the invoice to the specified Stripe customer.
    *   **`collection_method: 'charge_automatically'`**: Instructs Stripe to attempt to charge the customer's default payment method automatically.
    *   **`auto_advance: false`**: This is crucial. It tells Stripe *not* to automatically finalize and attempt to charge the invoice immediately after creation. We want to add an invoice item first.
    *   **`description`, `metadata`**: Descriptive text and key-value pairs for tracking purposes within Stripe.
    *   **`...(defaultPaymentMethod ? { default_payment_method: defaultPaymentMethod } : {})`**: Conditionally adds the `default_payment_method` to the invoice creation payload if one was successfully retrieved.
    *   **`{ idempotencyKey: \`${params.idempotencyKey}-invoice\` }`**: An idempotency key ensures that if this request is retried (e.g., due to a network error), Stripe will only process it once, preventing duplicate invoices. A suffix `-invoice` is added for clarity.
*   **`await stripe.invoiceItems.create(...)`**: Creates an item to be added to the newly created invoice.
    *   **`customer: params.customerId`**: Links the item to the customer.
    *   **`invoice: invoice.id`**: Links this item specifically to the `invoice` created in the previous step.
    *   **`amount: params.amountCents`, `currency: 'usd'`**: The charge amount in cents and the currency.
    *   **`description: params.itemDescription`**: Description for the invoice item.
    *   **`metadata: params.metadata`**: Additional metadata for the item.
    *   **`{ idempotencyKey: params.idempotencyKey }`**: Another idempotency key for the invoice item creation, preventing duplicate items.
*   **`if (invoice.id)`**: Ensures an invoice ID was successfully generated before proceeding.
    *   **`const finalized = await stripe.invoices.finalizeInvoice(invoice.id)`**: Finalizes the invoice. This step is necessary to calculate taxes, discounts, and make the invoice ready for payment.
    *   **`if (finalized.status === 'open' && finalized.id)`**: Checks if the invoice was successfully finalized and is in an 'open' state (meaning it's ready to be paid) and has an ID.
    *   **`try { ... } catch (payError) { ... }`**: Attempts to pay the invoice automatically.
        *   **`await stripe.invoices.pay(finalized.id, { payment_method: defaultPaymentMethod })`**: Initiates payment for the finalized invoice, optionally specifying the `defaultPaymentMethod` found earlier.
        *   **`catch (payError)`**: Logs an error if the automatic payment fails, but the process continues as the invoice has still been created and finalized.
*   **`return invoice.id || ''`**: Returns the ID of the created Stripe invoice. If for some reason `invoice.id` is falsy, it returns an empty string.

### 4. Main Logic: `checkAndBillOverageThreshold(userId)`

This asynchronous function is designed to check a *single user's* current usage overage and bill them if it crosses the defined threshold.

```typescript
export async function checkAndBillOverageThreshold(userId: string): Promise<void> {
  try {
    const threshold = OVERAGE_THRESHOLD

    const userSubscription = await getHighestPrioritySubscription(userId)

    if (!userSubscription || userSubscription.status !== 'active') {
      logger.debug('No active subscription for threshold billing', { userId })
      return
    }

    if (
      !userSubscription.plan ||
      userSubscription.plan === 'free' ||
      userSubscription.plan === 'enterprise'
    ) {
      return
    }

    if (userSubscription.plan === 'team') {
      logger.debug('Team plan detected - triggering org-level threshold billing', {
        userId,
        organizationId: userSubscription.referenceId,
      })
      await checkAndBillOrganizationOverageThreshold(userSubscription.referenceId)
      return
    }

    await db.transaction(async (tx) => {
      const statsRecords = await tx
        .select()
        .from(userStats)
        .where(eq(userStats.userId, userId))
        .for('update')
        .limit(1)

      if (statsRecords.length === 0) {
        logger.warn('User stats not found for threshold billing', { userId })
        return
      }

      const stats = statsRecords[0]

      const currentOverage = await calculateSubscriptionOverage({
        id: userSubscription.id,
        plan: userSubscription.plan,
        referenceId: userSubscription.referenceId,
        seats: userSubscription.seats,
      })
      const billedOverageThisPeriod = parseDecimal(stats.billedOverageThisPeriod)
      const unbilledOverage = Math.max(0, currentOverage - billedOverageThisPeriod)

      logger.debug('Threshold billing check', {
        userId,
        plan: userSubscription.plan,
        currentOverage,
        billedOverageThisPeriod,
        unbilledOverage,
        threshold,
      })

      if (unbilledOverage < threshold) {
        return
      }

      const amountToBill = unbilledOverage

      const stripeSubscriptionId = userSubscription.stripeSubscriptionId
      if (!stripeSubscriptionId) {
        logger.error('No Stripe subscription ID found', { userId })
        return
      }

      const stripe = requireStripeClient()
      const stripeSubscription = await stripe.subscriptions.retrieve(stripeSubscriptionId)
      const customerId =
        typeof stripeSubscription.customer === 'string'
          ? stripeSubscription.customer
          : stripeSubscription.customer.id

      const periodEnd = userSubscription.periodEnd
        ? Math.floor(userSubscription.periodEnd.getTime() / 1000)
        : Math.floor(Date.now() / 1000)
      const billingPeriod = new Date(periodEnd * 1000).toISOString().slice(0, 7)

      const amountCents = Math.round(amountToBill * 100)
      const totalOverageCents = Math.round(currentOverage * 100)
      const idempotencyKey = `threshold-overage:${customerId}:${stripeSubscriptionId}:${billingPeriod}:${totalOverageCents}:${amountCents}`

      logger.info('Creating threshold overage invoice', {
        userId,
        plan: userSubscription.plan,
        amountToBill,
        billingPeriod,
        idempotencyKey,
      })

      const cents = amountCents

      const invoiceId = await createAndFinalizeOverageInvoice(stripe, {
        customerId,
        stripeSubscriptionId,
        amountCents: cents,
        description: `Threshold overage billing – ${billingPeriod}`,
        itemDescription: `Usage overage ($${amountToBill.toFixed(2)})`,
        metadata: {
          type: 'overage_threshold_billing',
          userId,
          subscriptionId: stripeSubscriptionId,
          billingPeriod,
          totalOverageAtTimeOfBilling: currentOverage.toFixed(2),
        },
        idempotencyKey,
      })

      await tx
        .update(userStats)
        .set({
          billedOverageThisPeriod: sql`${userStats.billedOverageThisPeriod} + ${amountToBill}`,
        })
        .where(eq(userStats.userId, userId))

      logger.info('Successfully created and finalized threshold overage invoice', {
        userId,
        amountBilled: amountToBill,
        invoiceId,
        newBilledTotal: billedOverageThisPeriod + amountToBill,
      })
    })
  } catch (error) {
    logger.error('Error in threshold billing check', {
      userId,
      error,
    })
  }
}
```

*   **`export async function checkAndBillOverageThreshold(userId: string): Promise<void>`**: The main exported function that initiates the individual user overage check. It takes a `userId` as input.
*   **`try { ... } catch (error) { ... }`**: A robust error handling block that logs any errors encountered during the entire billing process for the user.
*   **`const threshold = OVERAGE_THRESHOLD`**: Retrieves the configured overage threshold.
*   **`const userSubscription = await getHighestPrioritySubscription(userId)`**: Fetches the user's primary subscription.
*   **Initial Subscription Checks**:
    *   **`if (!userSubscription || userSubscription.status !== 'active')`**: If no active subscription is found, it logs and exits.
    *   **`if (!userSubscription.plan || userSubscription.plan === 'free' || userSubscription.plan === 'enterprise')`**: Skips 'free' and 'enterprise' plans, as their billing models differ or don't involve usage overage.
    *   **`if (userSubscription.plan === 'team')`**: If the user is on a 'team' plan, individual billing is bypassed, and the request is delegated to `checkAndBillOrganizationOverageThreshold` using the `referenceId` (which is the organization ID for team plans). This ensures team usage is billed at the organization level.
*   **`await db.transaction(async (tx) => { ... })`**: All subsequent database operations are wrapped in a Drizzle transaction. This ensures that either all changes are successfully applied, or none are, maintaining data integrity, especially important in financial operations. The `tx` parameter is a transaction-specific database client.
    *   **`const statsRecords = await tx .select().from(userStats).where(eq(userStats.userId, userId)).for('update').limit(1)`**: Selects the `userStats` record for the given `userId`. `for('update')` is crucial here; it locks this row in the database, preventing other processes from modifying it until the transaction commits or rolls back, thus avoiding race conditions during calculation and update.
    *   **`if (statsRecords.length === 0)`**: Logs a warning and exits the transaction if user stats are not found.
    *   **`const stats = statsRecords[0]`**: Retrieves the single `userStats` record.
    *   **`const currentOverage = await calculateSubscriptionOverage(...)`**: Calculates the total theoretical overage for the user's subscription based on current usage.
    *   **`const billedOverageThisPeriod = parseDecimal(stats.billedOverageThisPeriod)`**: Gets the amount of overage already billed for the *current billing period* from the user's stats.
    *   **`const unbilledOverage = Math.max(0, currentOverage - billedOverageThisPeriod)`**: Calculates the *unbilled* overage by subtracting what's already been billed from the `currentOverage`. `Math.max(0, ...)` ensures the value is never negative.
    *   **Logging current state**: `logger.debug` outputs detailed information about the overage calculation.
    *   **`if (unbilledOverage < threshold)`**: If the `unbilledOverage` doesn't meet the `threshold`, there's nothing to bill, so the transaction exits.
    *   **`const amountToBill = unbilledOverage`**: The amount decided to be billed.
    *   **Stripe Preparation**:
        *   **`const stripeSubscriptionId = userSubscription.stripeSubscriptionId`**: Retrieves the Stripe subscription ID. An error is logged if it's missing.
        *   **`const stripe = requireStripeClient()`**: Gets the Stripe client.
        *   **`const stripeSubscription = await stripe.subscriptions.retrieve(stripeSubscriptionId)`**: Fetches the Stripe subscription object.
        *   **`const customerId = ...`**: Extracts the Stripe customer ID from the subscription object (which can be a string ID or an expanded customer object).
        *   **`const periodEnd = ...`**: Determines the end of the current billing period, used for context in the invoice. Falls back to `Date.now()` if not available.
        *   **`const billingPeriod = ...`**: Formats the billing period into a 'YYYY-MM' string.
        *   **`const amountCents = Math.round(amountToBill * 100)`**: Converts the dollar amount to cents for Stripe.
        *   **`const totalOverageCents = Math.round(currentOverage * 100)`**: Converts the total overage to cents for idempotency key.
        *   **`const idempotencyKey = ...`**: Generates a unique `idempotencyKey` for this specific billing event, incorporating customer, subscription, billing period, and amounts to prevent duplicate charges if the request is retried.
    *   **`const invoiceId = await createAndFinalizeOverageInvoice(...)`**: Calls the helper function to create and finalize the Stripe invoice with all the prepared details.
    *   **`await tx.update(userStats).set({ billedOverageThisPeriod: sql`${userStats.billedOverageThisPeriod} + ${amountToBill}` }).where(eq(userStats.userId, userId))`**: **This is the database update step within the transaction.** It increments the `billedOverageThisPeriod` in the user's stats by the `amountToBill`. Using `sql` ensures an atomic, correct update directly in the database.
    *   **Logging success**: `logger.info` logs the successful creation and finalization of the invoice.
*   **`catch (error)`**: Logs any errors that occurred during the `checkAndBillOverageThreshold` process.

### 5. Main Logic: `checkAndBillOrganizationOverageThreshold(organizationId)`

This asynchronous function mirrors the user billing logic but handles overage for entire organizations on 'team' plans.

```typescript
export async function checkAndBillOrganizationOverageThreshold(
  organizationId: string
): Promise<void> {
  logger.info('=== ENTERED checkAndBillOrganizationOverageThreshold ===', { organizationId })

  try {
    const threshold = OVERAGE_THRESHOLD

    logger.debug('Starting organization threshold billing check', { organizationId, threshold })

    const orgSubscriptions = await db
      .select()
      .from(subscription)
      .where(and(eq(subscription.referenceId, organizationId), eq(subscription.status, 'active')))
      .limit(1)

    if (orgSubscriptions.length === 0) {
      logger.debug('No active subscription for organization', { organizationId })
      return
    }

    const orgSubscription = orgSubscriptions[0]
    logger.debug('Found organization subscription', {
      organizationId,
      plan: orgSubscription.plan,
      seats: orgSubscription.seats,
      stripeSubscriptionId: orgSubscription.stripeSubscriptionId,
    })

    if (orgSubscription.plan !== 'team') {
      logger.debug('Organization plan is not team, skipping', {
        organizationId,
        plan: orgSubscription.plan,
      })
      return
    }

    const members = await db
      .select({ userId: member.userId, role: member.role })
      .from(member)
      .where(eq(member.organizationId, organizationId))

    logger.debug('Found organization members', {
      organizationId,
      memberCount: members.length,
      members: members.map((m) => ({ userId: m.userId, role: m.role })),
    })

    if (members.length === 0) {
      logger.warn('No members found for organization', { organizationId })
      return
    }

    const owner = members.find((m) => m.role === 'owner')
    if (!owner) {
      logger.error('No owner found for organization', { organizationId })
      return
    }

    logger.debug('Found organization owner, starting transaction', {
      organizationId,
      ownerId: owner.userId,
    })

    await db.transaction(async (tx) => {
      const ownerStatsLock = await tx
        .select()
        .from(userStats)
        .where(eq(userStats.userId, owner.userId))
        .for('update')
        .limit(1)

      if (ownerStatsLock.length === 0) {
        logger.error('Owner stats not found', { organizationId, ownerId: owner.userId })
        return
      }

      let totalTeamUsage = parseDecimal(ownerStatsLock[0].currentPeriodCost)
      const totalBilledOverage = parseDecimal(ownerStatsLock[0].billedOverageThisPeriod)

      const nonOwnerIds = members.filter((m) => m.userId !== owner.userId).map((m) => m.userId)

      if (nonOwnerIds.length > 0) {
        const memberStatsRows = await tx
          .select({
            userId: userStats.userId,
            currentPeriodCost: userStats.currentPeriodCost,
          })
          .from(userStats)
          .where(inArray(userStats.userId, nonOwnerIds))

        for (const stats of memberStatsRows) {
          totalTeamUsage += parseDecimal(stats.currentPeriodCost)
        }
      }

      const { basePrice: basePricePerSeat } = getPlanPricing(orgSubscription.plan)
      const basePrice = basePricePerSeat * (orgSubscription.seats || 1)
      const currentOverage = Math.max(0, totalTeamUsage - basePrice)
      const unbilledOverage = Math.max(0, currentOverage - totalBilledOverage)

      logger.debug('Organization threshold billing check', {
        organizationId,
        totalTeamUsage,
        basePrice,
        currentOverage,
        totalBilledOverage,
        unbilledOverage,
        threshold,
      })

      if (unbilledOverage < threshold) {
        return
      }

      const amountToBill = unbilledOverage

      const stripeSubscriptionId = orgSubscription.stripeSubscriptionId
      if (!stripeSubscriptionId) {
        logger.error('No Stripe subscription ID for organization', { organizationId })
        return
      }

      const stripe = requireStripeClient()
      const stripeSubscription = await stripe.subscriptions.retrieve(stripeSubscriptionId)
      const customerId =
        typeof stripeSubscription.customer === 'string'
          ? stripeSubscription.customer
          : stripeSubscription.customer.id

      const periodEnd = orgSubscription.periodEnd
        ? Math.floor(orgSubscription.periodEnd.getTime() / 1000)
        : Math.floor(Date.now() / 1000)
      const billingPeriod = new Date(periodEnd * 1000).toISOString().slice(0, 7)
      const amountCents = Math.round(amountToBill * 100)
      const totalOverageCents = Math.round(currentOverage * 100)

      const idempotencyKey = `threshold-overage-org:${customerId}:${stripeSubscriptionId}:${billingPeriod}:${totalOverageCents}:${amountCents}`

      logger.info('Creating organization threshold overage invoice', {
        organizationId,
        amountToBill,
        billingPeriod,
      })

      const cents = amountCents

      const invoiceId = await createAndFinalizeOverageInvoice(stripe, {
        customerId,
        stripeSubscriptionId,
        amountCents: cents,
        description: `Team threshold overage billing – ${billingPeriod}`,
        itemDescription: `Team usage overage ($${amountToBill.toFixed(2)})`,
        metadata: {
          type: 'overage_threshold_billing_org',
          organizationId,
          subscriptionId: stripeSubscriptionId,
          billingPeriod,
          totalOverageAtTimeOfBilling: currentOverage.toFixed(2),
        },
        idempotencyKey,
      })

      await tx
        .update(userStats)
        .set({
          billedOverageThisPeriod: sql`${userStats.billedOverageThisPeriod} + ${amountToBill}`,
        })
        .where(eq(userStats.userId, owner.userId))

      logger.info('Successfully created and finalized organization threshold overage invoice', {
        organizationId,
        ownerId: owner.userId,
        amountBilled: amountToBill,
        invoiceId,
      })
    })
  } catch (error) {
    logger.error('Error in organization threshold billing', {
      organizationId,
      error,
    })
  }
}
```

*   **`export async function checkAndBillOrganizationOverageThreshold(organizationId: string): Promise<void>`**: The exported function to check and bill organization overage. It takes an `organizationId`.
*   **Logging entry**: `logger.info` marks the entry into this function.
*   **`try { ... } catch (error) { ... }`**: Similar error handling as the user-specific function, logging errors with the `organizationId`.
*   **`const threshold = OVERAGE_THRESHOLD`**: Retrieves the configured overage threshold.
*   **Fetch Organization Subscription**:
    *   **`const orgSubscriptions = await db.select().from(subscription).where(and(eq(subscription.referenceId, organizationId), eq(subscription.status, 'active'))).limit(1)`**: Fetches the active subscription associated with the `organizationId`.
    *   **Checks**: Logs and exits if no active subscription is found or if the plan is not 'team'.
*   **Fetch Members**:
    *   **`const members = await db.select({ userId: member.userId, role: member.role }).from(member).where(eq(member.organizationId, organizationId))`**: Retrieves all members belonging to the organization.
    *   **Checks**: Logs and exits if no members are found or if no 'owner' role is found among the members (the owner is crucial for associating the billing).
*   **`await db.transaction(async (tx) => { ... })`**: Similar database transaction to ensure atomicity.
    *   **`const ownerStatsLock = await tx.select().from(userStats).where(eq(userStats.userId, owner.userId)).for('update').limit(1)`**: This is key: it locks the `userStats` record of the *organization owner*. The organization's collective overage will be tracked and billed against the owner's stats.
    *   **`if (ownerStatsLock.length === 0)`**: Logs an error if the owner's stats are not found.
    *   **Calculate Total Team Usage**:
        *   **`let totalTeamUsage = parseDecimal(ownerStatsLock[0].currentPeriodCost)`**: Starts summing usage with the owner's `currentPeriodCost`.
        *   **`const totalBilledOverage = parseDecimal(ownerStatsLock[0].billedOverageThisPeriod)`**: Retrieves the total overage already billed for this period from the *owner's* stats.
        *   **`const nonOwnerIds = members.filter(...).map(...)`**: Collects IDs of all members *except* the owner.
        *   **`if (nonOwnerIds.length > 0) { ... }`**: If there are non-owner members:
            *   **`const memberStatsRows = await tx.select(...).from(userStats).where(inArray(userStats.userId, nonOwnerIds))`**: Fetches `currentPeriodCost` for all non-owner members within the transaction.
            *   **`for (const stats of memberStatsRows) { totalTeamUsage += parseDecimal(stats.currentPeriodCost) }`**: Iterates through non-owner member stats and adds their `currentPeriodCost` to `totalTeamUsage`.
        *   **`const { basePrice: basePricePerSeat } = getPlanPricing(orgSubscription.plan)`**: Gets the base price per seat for the organization's plan.
        *   **`const basePrice = basePricePerSeat * (orgSubscription.seats || 1)`**: Calculates the total base price for the organization, considering the number of seats.
        *   **`const currentOverage = Math.max(0, totalTeamUsage - basePrice)`**: Calculates the total current overage for the entire team.
        *   **`const unbilledOverage = Math.max(0, currentOverage - totalBilledOverage)`**: Determines the unbilled overage for the organization.
    *   **Logging and Threshold Check**: Similar to individual user billing.
    *   **Stripe Preparation**: Similar to individual user billing, but `idempotencyKey` includes `-org` and `metadata` indicates `type: 'overage_threshold_billing_org'`.
    *   **`const invoiceId = await createAndFinalizeOverageInvoice(...)`**: Calls the helper to create and finalize the Stripe invoice for the organization.
    *   **`await tx.update(userStats).set({ billedOverageThisPeriod: sql`${userStats.billedOverageThisPeriod} + ${amountToBill}` }).where(eq(userStats.userId, owner.userId))`**: **Crucially, updates the `billedOverageThisPeriod` in the `userStats` record of the *owner* of the organization**, reflecting the organization-level overage bill.
    *   **Logging success**: Logs the successful billing, explicitly mentioning the `ownerId`.
*   **`catch (error)`**: Logs any errors specific to organization threshold billing.

---

### Summary

This file encapsulates the robust logic for proactive billing of usage overages. It intelligently distinguishes between individual users and organizations (teams), aggregates usage data, applies database transactions for data consistency, leverages Stripe for invoice creation and payment attempts, and provides extensive logging for monitoring and debugging. By using idempotency keys and clear error handling, it ensures a reliable and resilient billing process for dynamic usage-based pricing models.