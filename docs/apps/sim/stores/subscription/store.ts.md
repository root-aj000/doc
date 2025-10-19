```typescript
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'
import { DEFAULT_FREE_CREDITS } from '@/lib/billing/constants'
import { createLogger } from '@/lib/logs/console/logger'
import type {
  BillingStatus,
  SubscriptionData,
  SubscriptionStore,
  UsageData,
  UsageLimitData,
} from '@/stores/subscription/types'

const logger = createLogger('SubscriptionStore')

const CACHE_DURATION = 30 * 1000

const defaultUsage: UsageData = {
  current: 0,
  limit: DEFAULT_FREE_CREDITS,
  percentUsed: 0,
  isWarning: false,
  isExceeded: false,
  billingPeriodStart: null,
  billingPeriodEnd: null,
  lastPeriodCost: 0,
}

export const useSubscriptionStore = create<SubscriptionStore>()(
  devtools(
    (set, get) => ({
      // State
      subscriptionData: null,
      usageLimitData: null,
      isLoading: false,
      error: null,
      lastFetched: null,

      // Core actions
      loadSubscriptionData: async () => {
        const state = get()

        // Check cache validity
        if (
          state.subscriptionData &&
          state.lastFetched &&
          Date.now() - state.lastFetched < CACHE_DURATION
        ) {
          logger.debug('Using cached subscription data')
          return state.subscriptionData
        }

        // Don't start multiple concurrent requests
        if (state.isLoading) {
          logger.debug('Subscription data already loading, skipping duplicate request')
          return get().subscriptionData
        }

        set({ isLoading: true, error: null })

        try {
          const response = await fetch('/api/billing?context=user')

          if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`)
          }

          const result = await response.json()
          const data = { ...result.data, billingBlocked: result.data?.billingBlocked ?? false }

          // Transform dates with error handling
          const transformedData: SubscriptionData = {
            ...data,
            periodEnd: data.periodEnd
              ? (() => {
                  try {
                    const date = new Date(data.periodEnd)
                    return Number.isNaN(date.getTime()) ? null : date
                  } catch {
                    return null
                  }
                })()
              : null,
            usage: {
              ...data.usage,
              billingPeriodStart: data.usage?.billingPeriodStart
                ? (() => {
                    try {
                      const date = new Date(data.usage.billingPeriodStart)
                      return Number.isNaN(date.getTime()) ? null : date
                    } catch {
                      return null
                    }
                  })()
                : null,
              billingPeriodEnd: data.usage?.billingPeriodEnd
                ? (() => {
                    try {
                      const date = new Date(data.usage.billingPeriodEnd)
                      return Number.isNaN(date.getTime()) ? null : date
                    } catch {
                      return null
                    }
                  })()
                : null,
            },
            billingBlocked: !!data.billingBlocked,
          }

          // Debug logging for billing periods
          logger.debug('Billing period data', {
            raw: {
              billingPeriodStart: data.usage?.billingPeriodStart,
              billingPeriodEnd: data.usage?.billingPeriodEnd,
            },
            transformed: {
              billingPeriodStart: transformedData.usage.billingPeriodStart,
              billingPeriodEnd: transformedData.usage.billingPeriodEnd,
            },
          })

          set({
            subscriptionData: transformedData,
            isLoading: false,
            error: null,
            lastFetched: Date.now(),
          })

          logger.debug('Subscription data loaded successfully')
          return transformedData
        } catch (error) {
          const errorMessage =
            error instanceof Error ? error.message : 'Failed to load subscription data'
          logger.error('Failed to load subscription data', { error })

          set({
            isLoading: false,
            error: errorMessage,
          })
          return null
        }
      },

      loadUsageLimitData: async () => {
        try {
          const response = await fetch('/api/usage?context=user')

          if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`)
          }

          const data = await response.json()

          // Transform dates
          const transformedData: UsageLimitData = {
            ...data,
            updatedAt: data.updatedAt ? new Date(data.updatedAt) : undefined,
          }

          set({ usageLimitData: transformedData })
          logger.debug('Usage limit data loaded successfully')
          return transformedData
        } catch (error) {
          logger.error('Failed to load usage limit data', { error })
          // Don't set error state for usage limit failures - subscription data is more critical
          return null
        }
      },

      updateUsageLimit: async (newLimit: number) => {
        try {
          const response = await fetch('/api/usage?context=user', {
            method: 'PUT',
            headers: {
              'Content-Type': 'application/json',
            },
            body: JSON.stringify({ limit: newLimit }),
          })

          if (!response.ok) {
            const errorData = await response.json()
            throw new Error(errorData.error || 'Failed to update usage limit')
          }

          // Refresh the store state to ensure consistency
          await get().refresh()

          logger.debug('Usage limit updated successfully', { newLimit })
          return { success: true }
        } catch (error) {
          const errorMessage =
            error instanceof Error ? error.message : 'Failed to update usage limit'
          logger.error('Failed to update usage limit', { error, newLimit })
          return { success: false, error: errorMessage }
        }
      },

      refresh: async () => {
        // Force refresh by clearing cache
        set({ lastFetched: null })
        await get().loadData()
      },

      // Load both subscription and usage limit data in parallel
      loadData: async () => {
        const state = get()

        // Check cache validity for subscription data
        if (
          state.subscriptionData &&
          state.lastFetched &&
          Date.now() - state.lastFetched < CACHE_DURATION
        ) {
          logger.debug('Using cached data')
          // Still load usage limit if not present
          if (!state.usageLimitData) {
            const usageLimitData = await get().loadUsageLimitData()
            return {
              subscriptionData: state.subscriptionData,
              usageLimitData: usageLimitData,
            }
          }
          return {
            subscriptionData: state.subscriptionData,
            usageLimitData: state.usageLimitData,
          }
        }

        // Don't start multiple concurrent requests
        if (state.isLoading) {
          logger.debug('Data already loading, skipping duplicate request')
          return {
            subscriptionData: get().subscriptionData,
            usageLimitData: get().usageLimitData,
          }
        }

        set({ isLoading: true, error: null })

        try {
          // Load both subscription and usage limit data in parallel
          const [subscriptionResponse, usageLimitResponse] = await Promise.all([
            fetch('/api/billing?context=user'),
            fetch('/api/usage?context=user'),
          ])

          if (!subscriptionResponse.ok) {
            throw new Error(`HTTP error! status: ${subscriptionResponse.status}`)
          }

          const subscriptionResult = await subscriptionResponse.json()
          const subscriptionData = subscriptionResult.data
          let usageLimitData = null

          if (usageLimitResponse.ok) {
            usageLimitData = await usageLimitResponse.json()
          } else {
            logger.warn('Failed to load usage limit data, using defaults')
          }

          // Transform subscription data dates with error handling
          const transformedSubscriptionData: SubscriptionData = {
            ...subscriptionData,
            periodEnd: subscriptionData.periodEnd
              ? (() => {
                  try {
                    const date = new Date(subscriptionData.periodEnd)
                    return Number.isNaN(date.getTime()) ? null : date
                  } catch {
                    return null
                  }
                })()
              : null,
            usage: {
              ...subscriptionData.usage,
              billingPeriodStart: subscriptionData.usage?.billingPeriodStart
                ? (() => {
                    try {
                      const date = new Date(subscriptionData.usage.billingPeriodStart)
                      return Number.isNaN(date.getTime()) ? null : date
                    } catch {
                      return null
                    }
                  })()
                : null,
              billingPeriodEnd: subscriptionData.usage?.billingPeriodEnd
                ? (() => {
                    try {
                      const date = new Date(subscriptionData.usage.billingPeriodEnd)
                      return Number.isNaN(date.getTime()) ? null : date
                    } catch {
                      return null
                    }
                  })()
                : null,
            },
          }

          // Debug logging for parallel billing periods
          logger.debug('Parallel billing period data', {
            raw: {
              billingPeriodStart: subscriptionData.usage?.billingPeriodStart,
              billingPeriodEnd: subscriptionData.usage?.billingPeriodEnd,
            },
            transformed: {
              billingPeriodStart: transformedSubscriptionData.usage.billingPeriodStart,
              billingPeriodEnd: transformedSubscriptionData.usage.billingPeriodEnd,
            },
          })

          // Transform usage limit data dates if present
          const transformedUsageLimitData: UsageLimitData | null = usageLimitData
            ? {
                ...usageLimitData,
                updatedAt: usageLimitData.updatedAt
                  ? new Date(usageLimitData.updatedAt)
                  : undefined,
              }
            : null

          set({
            subscriptionData: transformedSubscriptionData,
            usageLimitData: transformedUsageLimitData,
            isLoading: false,
            error: null,
            lastFetched: Date.now(),
          })

          logger.debug('Data loaded successfully in parallel')
          return {
            subscriptionData: transformedSubscriptionData,
            usageLimitData: transformedUsageLimitData,
          }
        } catch (error) {
          const errorMessage = error instanceof Error ? error.message : 'Failed to load data'
          logger.error('Failed to load data', { error })

          set({
            isLoading: false,
            error: errorMessage,
          })
          return {
            subscriptionData: null,
            usageLimitData: null,
          }
        }
      },

      clearError: () => {
        set({ error: null })
      },

      reset: () => {
        set({
          subscriptionData: null,
          usageLimitData: null,
          isLoading: false,
          error: null,
          lastFetched: null,
        })
      },

      // Computed getters
      getSubscriptionStatus: () => {
        const data = get().subscriptionData
        return {
          isPaid: data?.isPaid ?? false,
          isPro: data?.isPro ?? false,
          isTeam: data?.isTeam ?? false,
          isEnterprise: data?.isEnterprise ?? false,
          isFree: !(data?.isPaid ?? false),
          plan: data?.plan ?? 'free',
          status: data?.status ?? null,
          seats: data?.seats ?? null,
          metadata: data?.metadata ?? null,
        }
      },

      getUsage: () => {
        return get().subscriptionData?.usage ?? defaultUsage
      },

      getBillingStatus: (): BillingStatus => {
        const usage = get().getUsage()
        const blocked = get().subscriptionData?.billingBlocked
        if (blocked) return 'blocked'
        if (usage.isExceeded) return 'exceeded'
        if (usage.isWarning) return 'warning'
        return 'ok'
      },

      getRemainingBudget: () => {
        const usage = get().getUsage()
        return Math.max(0, usage.limit - usage.current)
      },

      getDaysRemainingInPeriod: () => {
        const usage = get().getUsage()
        if (!usage.billingPeriodEnd) return null

        const now = new Date()
        const endDate = usage.billingPeriodEnd
        const diffTime = endDate.getTime() - now.getTime()
        const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24))

        return Math.max(0, diffDays)
      },

      isAtLeastPro: () => {
        const status = get().getSubscriptionStatus()
        return status.isPro || status.isTeam || status.isEnterprise
      },

      isAtLeastTeam: () => {
        const status = get().getSubscriptionStatus()
        return status.isTeam || status.isEnterprise
      },

      canUpgrade: () => {
        const status = get().getSubscriptionStatus()
        return status.plan === 'free' || status.plan === 'pro'
      },
    }),
    { name: 'subscription-store' }
  )
)
```

## Detailed Explanation of the Code

This file defines a Zustand store called `useSubscriptionStore` that manages the subscription and usage data for a user. It handles fetching, caching, updating, and deriving information related to a user's subscription plan and resource usage.  It leverages `zustand` for state management and `zustand/middleware` for developer tools.  Let's break down the code step by step:

**1. Imports:**

*   `create` from `zustand`: This is the core function from Zustand used to create a store.  Zustand is a simple state management library.
*   `devtools` from `zustand/middleware`: This is middleware that enables the Redux DevTools extension for debugging the Zustand store. It provides features like time-travel debugging and state inspection.
*   `DEFAULT_FREE_CREDITS` from `@/lib/billing/constants`: This imports a constant representing the default number of free credits a user gets, likely defined in another file.
*   `createLogger` from `@/lib/logs/console/logger`:  Imports a custom logger function used for consistent logging throughout the store. This promotes better debugging and monitoring.
*   Type definitions from `@/stores/subscription/types`: These imports define the TypeScript types used within the store, ensuring type safety and better code maintainability:
    *   `BillingStatus`:  Likely an enum or type defining the possible billing states (e.g., 'ok', 'warning', 'exceeded', 'blocked').
    *   `SubscriptionData`:  A type defining the structure of the subscription data object, including details about the user's plan, billing period, and usage.
    *   `SubscriptionStore`:  A type defining the overall structure of the Zustand store, including the state properties and action methods.
    *   `UsageData`: A type definition for the usage data such as current usage, limits, and billing periods.
    *   `UsageLimitData`: A type defining the structure of the usage limit data object.

**2. Logger Initialization:**

*   `const logger = createLogger('SubscriptionStore')`: Creates an instance of the custom logger, tagged with "SubscriptionStore" for easy identification of logs originating from this file.

**3. Cache Duration:**

*   `const CACHE_DURATION = 30 * 1000`:  Defines the cache duration in milliseconds (30 seconds). This means the store will consider cached data valid for 30 seconds before attempting to fetch fresh data from the API.  This helps to reduce unnecessary API calls and improve performance.

**4. Default Usage Data:**

*   `const defaultUsage: UsageData = { ... }`: Defines the default values for the `UsageData` object when no data is available from the API.  This ensures that the store always has a valid `usage` object to work with, preventing potential errors.  It initializes `current` to 0, `limit` to `DEFAULT_FREE_CREDITS`, `percentUsed` to 0, `isWarning` and `isExceeded` to `false`, and the billing period dates to `null`. The `lastPeriodCost` defaults to `0`.

**5. Creating the Zustand Store:**

*   `export const useSubscriptionStore = create<SubscriptionStore>()( ... )`: This is the core of the code, creating the Zustand store and exporting a hook called `useSubscriptionStore` that components can use to access and update the store's state.
*   `devtools( (set, get) => ({ ... }), { name: 'subscription-store' } )`:  The `devtools` middleware wraps the store's definition, enabling Redux DevTools integration. The `name` option provides a label for the store in the DevTools. The function passed to `devtools` is where the store's state and actions are defined.
*   `(set, get) => ({ ... })`: This function receives two arguments:
    *   `set`: A function to update the store's state.  It's similar to `setState` in React class components.
    *   `get`: A function to retrieve the current store's state.

**6. Store State:**

*   `subscriptionData: null,`:  Stores the subscription data fetched from the API.  Initialized to `null` until data is loaded.  Type: `SubscriptionData | null`
*   `usageLimitData: null,`: Stores the usage limit data. Initialized to `null`. Type: `UsageLimitData | null`
*   `isLoading: false,`:  A boolean flag indicating whether the store is currently fetching data from the API.  This is useful for displaying loading indicators in the UI.
*   `error: null,`: Stores any error message that occurs during data fetching.  This allows the UI to display error messages to the user. Type: `string | null`
*   `lastFetched: null,`: Stores the timestamp of the last successful data fetch. Used for caching.  Type: `number | null`

**7. Store Actions:**

The store actions are functions that modify the store's state and/or perform asynchronous operations.

*   **`loadSubscriptionData: async () => { ... }`**: This action is responsible for fetching subscription data from the `/api/billing?context=user` endpoint.  It implements a caching mechanism to avoid unnecessary API calls.

    *   **Cache Check:** It first checks if the `subscriptionData` is already present, `lastFetched` is set, and the cached data is still valid (within `CACHE_DURATION`). If so, it returns the cached data.
    *   **Loading Check:**  It prevents multiple concurrent requests by checking the `isLoading` flag. If a request is already in progress, it returns the existing `subscriptionData`.
    *   **Fetching Data:**  If the cache is invalid or no data is loaded, it sets `isLoading` to `true`, fetches data from the API, and handles potential errors.
    *   **Data Transformation:** The `periodEnd`, `billingPeriodStart` and `billingPeriodEnd` properties are transformed from strings to `Date` objects. This is crucial because the API might return these values as strings, and the application likely needs them as `Date` objects for calculations and display.  The transformation includes error handling to gracefully handle invalid date formats.  It uses an Immediately Invoked Function Expression (IIFE) to wrap the date parsing logic, allowing for a `try...catch` block to handle potential errors during date conversion.  If the `Date` constructor fails to parse the string (e.g., due to an invalid format), the `catch` block returns `null`. The `Number.isNaN(date.getTime())` check is crucial to handle cases where the `new Date()` constructor successfully creates a `Date` object but the date is invalid.
    *   **Setting State:**  Upon successful data retrieval and transformation, it updates the store's state with the fetched data, sets `isLoading` to `false`, clears any previous errors, and updates the `lastFetched` timestamp.
    *   **Error Handling:** If the API request fails or an error occurs during data processing, it sets `isLoading` to `false` and stores the error message in the `error` state.

*   **`loadUsageLimitData: async () => { ... }`**:  Fetches usage limit data from the `/api/usage?context=user` endpoint.

    *   **Fetching Data:**  Fetches data from the API and handles potential errors.
    *   **Data Transformation:** Transforms the `updatedAt` property into a `Date` object.
    *   **Setting State:** Updates the `usageLimitData` state.
    *   **Error Handling:** Logs an error if the API request fails, but doesn't set an error in the store's state.  This is likely because subscription data is considered more critical, and failing to load usage limits shouldn't block the application.

*   **`updateUsageLimit: async (newLimit: number) => { ... }`**: Updates the usage limit on the server.

    *   **API Call:**  Makes a `PUT` request to the `/api/usage?context=user` endpoint to update the usage limit.
    *   **Refresh:** Calls the `refresh` action to reload the subscription data and usage limit data after a successful update.
    *   **Error Handling:**  Handles potential errors during the API call and returns a `success: false` object with an error message if the update fails.

*   **`refresh: async () => { ... }`**: Forces a refresh of the data by invalidating the cache (`lastFetched = null`) and then calling `loadData` to fetch fresh data.

*   **`loadData: async () => { ... }`**: This action is designed to load both subscription data and usage limit data.  It optimizes the process by loading the data in parallel and also implements caching.

    *   **Cache Check:** It checks if the `subscriptionData` is already present, `lastFetched` is set, and the cached data is still valid. If so, it returns the cached data and loads usageLimitData if it is not yet present.
    *   **Loading Check:** Prevents multiple concurrent requests by checking `isLoading`.
    *   **Parallel Data Fetching:** Uses `Promise.all` to fetch both subscription data and usage limit data concurrently. This improves performance by reducing the overall loading time.
    *   **Data Transformation:**  Transforms dates from string format to javascript `Date` objects.
    *   **Setting State:**  Updates the `subscriptionData`, `usageLimitData`, `isLoading`, `error`, and `lastFetched` states.
    *   **Error Handling:** Handles potential errors during the API calls and sets the `error` state accordingly.

*   **`clearError: () => { ... }`**:  Clears the `error` state, allowing the UI to reset error messages.

*   **`reset: () => { ... }`**: Resets the store to its initial state (null subscription data, null usage limit data, isLoading false, no error, and no last fetched timestamp).

**8. Computed Getters:**

These are functions that derive values from the store's state. They don't modify the state directly but provide a convenient way to access derived data.

*   **`getSubscriptionStatus: () => { ... }`**:  Extracts and derives various boolean flags and plan information from the `subscriptionData`.  It handles cases where `subscriptionData` is null by providing default values (e.g., `isPaid: false` if `subscriptionData` is null).
*   **`getUsage: () => { ... }`**: Returns the usage data. If `subscriptionData` is null, it returns the `defaultUsage` object.
*   **`getBillingStatus: (): BillingStatus => { ... }`**:  Determines the overall billing status based on the user's usage and subscription. Returns 'blocked', 'exceeded', 'warning', or 'ok'.
*   **`getRemainingBudget: () => { ... }`**:  Calculates the remaining budget based on the usage limit and current usage. Ensures the result is never negative by using `Math.max(0, ...)`.
*   **`getDaysRemainingInPeriod: () => { ... }`**: Calculates the number of days remaining in the current billing period. Returns `null` if `billingPeriodEnd` is not available.  Uses `Math.max(0, ...)` to ensure the result is never negative.
*   **`isAtLeastPro: () => { ... }`**: Checks if the user has a Pro, Team, or Enterprise subscription.
*   **`isAtLeastTeam: () => { ... }`**: Checks if the user has a Team or Enterprise subscription.
*   **`canUpgrade: () => { ... }`**:  Checks if the user can upgrade their plan (currently, if they are on a free or pro plan).

**Purpose of the file:**

The primary purpose of this file is to encapsulate the state management logic for user subscription and usage data. It centralizes data fetching, caching, updates, and derived calculations, providing a clean and consistent way for components to access and interact with this data.  This promotes code reusability, maintainability, and a better user experience by optimizing data fetching and providing derived values.

**Simplifying Complex Logic:**

The code simplifies complex logic in several ways:

*   **Caching:** Reduces the need for frequent API calls, improving performance.
*   **Parallel Data Fetching:** Loads subscription and usage data concurrently, reducing overall loading time.
*   **Data Transformation:** Handles data type conversions (e.g., string to Date) in a centralized location.
*   **Computed Getters:** Provides easy access to derived values, avoiding redundant calculations in components.
*   **Error Handling:** Centralizes error handling and provides a consistent way to display error messages.
*   **Zustand Abstraction**: Zustand greatly simplifies state management compared to using `useState` and `useEffect` throughout the application.  It avoids prop drilling and allows components to easily access and update the subscription state.

In summary, this file provides a well-structured and efficient way to manage user subscription and usage data using Zustand, caching, parallel data fetching, and computed getters. This approach improves performance, maintainability, and the overall user experience.
