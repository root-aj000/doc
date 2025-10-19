This TypeScript file is the **central configuration for an application's authentication system**, built using the `better-auth` library. It defines how users sign up, sign in, manage sessions, and integrate with a multitude of third-party services like social logins (GitHub, Google), custom OAuth providers (Microsoft, Atlassian, Notion, etc.), Single Sign-On (SSO), and Stripe for billing and organization management.

It acts as the single source of truth for all authentication-related logic, including database interactions, email sending for verification/resets, session management, and authorization rules for organizations and subscriptions.

Let's break down each part of the code in detail.

---

## Detailed Code Explanation

### 1. Imports and Setup

This section brings in all the necessary modules and libraries the authentication system will rely on.

```typescript
import { sso } from '@better-auth/sso'
import { stripe } from '@better-auth/stripe'
import { db } from '@sim/db'
import * as schema from '@sim/db/schema'
import { betterAuth } from 'better-auth'
import { drizzleAdapter } from 'better-auth/adapters/drizzle'
import { nextCookies } from 'better-auth/next-js'
import {
  createAuthMiddleware,
  customSession,
  emailOTP,
  genericOAuth,
  oneTimeToken,
  organization,
} from 'better-auth/plugins'
import { and, eq } from 'drizzle-orm'
import { headers } from 'next/headers'
import Stripe from 'stripe'
import {
  getEmailSubject,
  renderInvitationEmail,
  renderOTPEmail,
  renderPasswordResetEmail,
} from '@/components/emails/render-email'
import { sendPlanWelcomeEmail } from '@/lib/billing'
import { authorizeSubscriptionReference } from '@/lib/billing/authorization'
import { handleNewUser } from '@/lib/billing/core/usage'
import { syncSubscriptionUsageLimits } from '@/lib/billing/organization'
import { getPlans } from '@/lib/billing/plans'
import { handleManualEnterpriseSubscription } from '@/lib/billing/webhooks/enterprise'
import {
  handleInvoiceFinalized,
  handleInvoicePaymentFailed,
  handleInvoicePaymentSucceeded,
} from '@/lib/billing/webhooks/invoices'
import {
  handleSubscriptionCreated,
  handleSubscriptionDeleted,
} from '@/lib/billing/webhooks/subscription'
import { sendEmail } from '@/lib/email/mailer'
import { getFromEmailAddress } from '@/lib/email/utils'
import { quickValidateEmail } from '@/lib/email/validation'
import { env, isTruthy } from '@/lib/env'
import { isBillingEnabled, isEmailVerificationEnabled } from '@/lib/environment'
import { createLogger } from '@/lib/logs/console/logger'
import { getBaseUrl } from '@/lib/urls/utils'
import { SSO_TRUSTED_PROVIDERS } from './sso/consts'
```

*   **`@better-auth/*` and `better-auth`:** These are the core authentication library imports.
    *   `sso`: Plugin for Single Sign-On (like SAML or OpenID Connect enterprise integrations).
    *   `stripe`: Plugin for integrating with Stripe for billing and subscriptions.
    *   `betterAuth`: The main function to configure the authentication system.
    *   `drizzleAdapter`: An adapter to connect `better-auth` with a Drizzle ORM database.
    *   `nextCookies`: A plugin specifically for Next.js to handle session cookies.
    *   `createAuthMiddleware`, `customSession`, `emailOTP`, `genericOAuth`, `oneTimeToken`, `organization`: Various plugins provided by `better-auth` for different authentication features (middleware, custom session data, email One-Time Passwords, generic OAuth, short-lived tokens, and organization management).
*   **`drizzle-orm`:**
    *   `db` from `@sim/db`: The Drizzle ORM database client instance.
    *   `schema` from `@sim/db/schema`: The database schema definitions (tables, relations) used by Drizzle.
    *   `and`, `eq`: Drizzle ORM functions used for constructing database queries (logical AND, equality comparison).
*   **`next/headers`:** Next.js utility to access request headers, primarily used here to get session information.
*   **`Stripe`:** The Stripe Node.js client library.
*   **Email Utilities (`@/components/emails`, `@/lib/email`):**
    *   `getEmailSubject`, `renderInvitationEmail`, `renderOTPEmail`, `renderPasswordResetEmail`: Functions to render HTML email templates for different purposes.
    *   `sendEmail`: A utility to send emails.
    *   `getFromEmailAddress`: Retrieves the sender's email address.
    *   `quickValidateEmail`: Performs basic email format validation.
*   **Billing Utilities (`@/lib/billing`):** A comprehensive set of functions to handle various billing-related operations, interacting with Stripe and potentially the application's own database.
    *   `sendPlanWelcomeEmail`: Sends a welcome email after a plan subscription.
    *   `authorizeSubscriptionReference`: Authorizes a subscription using a given reference ID.
    *   `handleNewUser`: Initializes billing-related stats/data for a new user.
    *   `syncSubscriptionUsageLimits`: Updates usage limits based on the user's subscription.
    *   `getPlans`: Retrieves available subscription plans.
    *   `handleManualEnterpriseSubscription`: Handles manual enterprise subscription events.
    *   `handleInvoiceFinalized`, `handleInvoicePaymentFailed`, `handleInvoicePaymentSucceeded`: Handle different Stripe invoice events.
    *   `handleSubscriptionCreated`, `handleSubscriptionDeleted`: Handle Stripe subscription lifecycle events.
*   **Environment and Core Utilities (`@/lib/env`, `@/lib/environment`, `@/lib/logs`, `@/lib/urls`):**
    *   `env`, `isTruthy`: Access environment variables and check for truthiness.
    *   `isBillingEnabled`, `isEmailVerificationEnabled`: Feature flags based on environment configuration.
    *   `createLogger`: A utility for logging messages.
    *   `getBaseUrl`: Retrieves the application's base URL.
*   **`SSO_TRUSTED_PROVIDERS`:** A constant imported from a local file, likely a list of trusted SSO providers for account linking.

---

### 2. Logger and Stripe Client Initialization

```typescript
const logger = createLogger('Auth')

// Only initialize Stripe if the key is provided
// This allows local development without a Stripe account
const validStripeKey = env.STRIPE_SECRET_KEY

let stripeClient = null
if (validStripeKey) {
  stripeClient = new Stripe(env.STRIPE_SECRET_KEY || '', {
    apiVersion: '2025-08-27.basil',
  })
}
```

*   **`const logger = createLogger('Auth')`**: Initializes a logger instance specifically for the authentication module, making it easier to track auth-related events in logs.
*   **`const validStripeKey = env.STRIPE_SECRET_KEY`**: Checks if the Stripe secret key is provided in the environment variables.
*   **`let stripeClient = null`**: Declares a variable to hold the Stripe client instance, initialized to `null`.
*   **`if (validStripeKey) { ... }`**: This conditional block ensures that the Stripe client is **only initialized if a secret key is available**. This is a good practice for local development or environments where Stripe might not be configured, preventing errors.
    *   `stripeClient = new Stripe(env.STRIPE_SECRET_KEY || '', { ... })`: Creates a new instance of the `Stripe` client using the secret key.
    *   `apiVersion: '2025-08-27.basil'`: Specifies the Stripe API version to use. This ensures consistent behavior even if Stripe updates its API.

---

### 3. `betterAuth` Configuration (`export const auth = betterAuth({...})`)

This is the core of the file, where the `better-auth` library is configured with all its settings, integrations, and custom logic.

#### A. Base Configuration

```typescript
export const auth = betterAuth({
  baseURL: getBaseUrl(),
  trustedOrigins: [
    getBaseUrl(),
    ...(env.NEXT_PUBLIC_SOCKET_URL ? [env.NEXT_PUBLIC_SOCKET_URL] : []),
  ].filter(Boolean),
  database: drizzleAdapter(db, {
    provider: 'pg',
    schema,
  }),
  session: {
    cookieCache: {
      enabled: true,
      maxAge: 24 * 60 * 60, // 24 hours in seconds
    },
    expiresIn: 30 * 24 * 60 * 60, // 30 days (how long a session can last overall)
    updateAge: 24 * 60 * 60, // 24 hours (how often to refresh the expiry)
    freshAge: 60 * 60, // 1 hour (or set to 0 to disable completely)
  },
  databaseHooks: {
    user: {
      create: {
        after: async (user) => {
          logger.info('[databaseHooks.user.create.after] User created, initializing stats', {
            userId: user.id,
          })

          try {
            await handleNewUser(user.id)
          } catch (error) {
            logger.error('[databaseHooks.user.create.after] Failed to initialize user stats', {
              userId: user.id,
              error,
            })
          }
        },
      },
    },
    session: {
      create: {
        before: async (session) => {
          try {
            // Find the first organization this user is a member of
            const members = await db
              .select()
              .from(schema.member)
              .where(eq(schema.member.userId, session.userId))
              .limit(1)

            if (members.length > 0) {
              logger.info('Found organization for user', {
                userId: session.userId,
                organizationId: members[0].organizationId,
              })

              return {
                data: {
                  ...session,
                  activeOrganizationId: members[0].organizationId,
                },
              }
            }
            logger.info('No organizations found for user', {
              userId: session.userId,
            })
            return { data: session }
          } catch (error) {
            logger.error('Error setting active organization', {
              error,
              userId: session.userId,
            })
            return { data: session }
          }
        },
      },
    },
  },
  account: {
    accountLinking: {
      enabled: true,
      allowDifferentEmails: true,
      trustedProviders: [
        // Standard OAuth providers
        'google',
        'github',
        'email-password',
        'confluence',
        'supabase',
        'x',
        'notion',
        'microsoft',
        'slack',
        'reddit',

        // Common SSO provider patterns
        ...SSO_TRUSTED_PROVIDERS,
      ],
    },
  },
  // ... rest of the config
})
```

*   **`baseURL: getBaseUrl()`**: Sets the base URL for the application, which `better-auth` uses for redirects and internal links.
*   **`trustedOrigins`**: An array of URLs that the authentication system considers safe for redirects. This includes the application's base URL and optionally a WebSocket URL if configured. `.filter(Boolean)` removes any `null` or `undefined` entries if `env.NEXT_PUBLIC_SOCKET_URL` is not set.
*   **`database: drizzleAdapter(db, { provider: 'pg', schema })`**: Integrates `better-auth` with the database using Drizzle ORM.
    *   `db`: The Drizzle database client instance.
    *   `provider: 'pg'`: Specifies PostgreSQL as the database provider.
    *   `schema`: Provides Drizzle with the database schema definitions (tables like `user`, `session`, `account`, etc.) so `better-auth` knows how to interact with them.
*   **`session`**: Configures how user sessions are managed.
    *   `cookieCache.enabled`: Enables caching of session data in cookies for performance.
    *   `cookieCache.maxAge`: How long the session data can be cached in the cookie (24 hours).
    *   `expiresIn`: The maximum total duration a session can last without being refreshed (30 days).
    *   `updateAge`: How often the session's expiry should be refreshed (24 hours). If a user is active within this period, their session expiry is extended.
    *   `freshAge`: How long a session is considered "fresh" without needing to be validated against the database (1 hour). Setting to 0 disables this.
*   **`databaseHooks`**: Allows executing custom logic before or after specific database operations.
    *   **`user.create.after`**: This hook runs *after* a new user record has been created in the database.
        *   It logs the creation and then attempts to call `handleNewUser(user.id)`. This likely initializes billing or usage statistics for the newly registered user.
        *   Includes robust `try...catch` blocks for error logging.
    *   **`session.create.before`**: This hook runs *before* a new session record is created for a user.
        *   It queries the `member` table to find if the user is part of any organization (`db.select().from(schema.member).where(eq(schema.member.userId, session.userId)).limit(1)`).
        *   If an organization is found, it adds an `activeOrganizationId` to the session data (`return { data: { ...session, activeOrganizationId: members[0].organizationId } }`). This pre-populates the session with contextual organization information.
        *   Logs whether an organization was found or not.
        *   Includes `try...catch` for error handling, ensuring the session is still created even if this hook fails.
*   **`account.accountLinking`**: Configures how user accounts from different authentication providers (e.g., GitHub and Google) can be linked together under a single user profile.
    *   `enabled: true`: Allows account linking.
    *   `allowDifferentEmails: true`: Permits linking accounts even if they use different email addresses, which can be useful but also has security implications (should be carefully considered).
    *   `trustedProviders`: A list of providers whose accounts can be linked. This includes standard OAuth providers and a dynamically imported list of SSO providers.

#### B. Authentication Methods

```typescript
  socialProviders: {
    github: {
      clientId: env.GITHUB_CLIENT_ID as string,
      clientSecret: env.GITHUB_CLIENT_SECRET as string,
      scopes: ['user:email', 'repo'],
    },
    google: {
      clientId: env.GOOGLE_CLIENT_ID as string,
      clientSecret: env.GOOGLE_CLIENT_SECRET as string,
      scopes: [
        'https://www.googleapis.com/auth/userinfo.email',
        'https://www.googleapis.com/auth/userinfo.profile',
      ],
    },
  },
  emailAndPassword: {
    enabled: true,
    requireEmailVerification: isEmailVerificationEnabled,
    sendVerificationOnSignUp: false,
    throwOnMissingCredentials: true,
    throwOnInvalidCredentials: true,
    sendResetPassword: async ({ user, url, token }, request) => {
      const username = user.name || ''

      const html = await renderPasswordResetEmail(username, url)

      const result = await sendEmail({
        to: user.email,
        subject: getEmailSubject('reset-password'),
        html,
        from: getFromEmailAddress(),
        emailType: 'transactional',
      })

      if (!result.success) {
        throw new Error(`Failed to send reset password email: ${result.message}`)
      }
    },
  },
  // ... rest of the config
```

*   **`socialProviders`**: Configures standard social login providers.
    *   **`github`**:
        *   `clientId`, `clientSecret`: Credentials for the GitHub OAuth application, loaded from environment variables.
        *   `scopes`: Permissions requested from GitHub (e.g., user email, repository access).
    *   **`google`**:
        *   `clientId`, `clientSecret`: Credentials for the Google OAuth application.
        *   `scopes`: Permissions requested from Google (e.g., user email, profile information).
*   **`emailAndPassword`**: Configures traditional email and password authentication.
    *   `enabled: true`: Activates this authentication method.
    *   `requireEmailVerification`: Determines if new sign-ups require email verification, based on an environment flag.
    *   `sendVerificationOnSignUp: false`: Disables automatic email verification sending upon initial sign-up (this might be handled by the `emailOTP` plugin later).
    *   `throwOnMissingCredentials`, `throwOnInvalidCredentials`: Ensures strong error handling for login attempts.
    *   **`sendResetPassword`**: A custom function that `better-auth` calls when a user requests a password reset.
        *   It renders a password reset email using `renderPasswordResetEmail`, passing the user's name and the reset URL.
        *   It then sends this email using the `sendEmail` utility with a specific subject and sender.
        *   If the email fails to send, it throws an error.

#### C. Global Hooks

```typescript
  hooks: {
    before: createAuthMiddleware(async (ctx) => {
      if (ctx.path.startsWith('/sign-up') && isTruthy(env.DISABLE_REGISTRATION))
        throw new Error('Registration is disabled, please contact your admin.')

      if (
        (ctx.path.startsWith('/sign-in') || ctx.path.startsWith('/sign-up')) &&
        (env.ALLOWED_LOGIN_EMAILS || env.ALLOWED_LOGIN_DOMAINS)
      ) {
        const requestEmail = ctx.body?.email?.toLowerCase()

        if (requestEmail) {
          let isAllowed = false

          if (env.ALLOWED_LOGIN_EMAILS) {
            const allowedEmails = env.ALLOWED_LOGIN_EMAILS.split(',').map((email) =>
              email.trim().toLowerCase()
            )
            isAllowed = allowedEmails.includes(requestEmail)
          }

          if (!isAllowed && env.ALLOWED_LOGIN_DOMAINS) {
            const allowedDomains = env.ALLOWED_LOGIN_DOMAINS.split(',').map((domain) =>
              domain.trim().toLowerCase()
            )
            const emailDomain = requestEmail.split('@')[1]
            isAllowed = emailDomain && allowedDomains.includes(emailDomain)
          }

          if (!isAllowed) {
            throw new Error('Access restricted. Please contact your administrator.')
          }
        }
      }

      return
    }),
  },
  // ... rest of the config
```

*   **`hooks.before`**: This is a global middleware that runs *before* any authentication route is processed by `better-auth`. It uses `createAuthMiddleware` to wrap the custom logic.
    *   **Registration Disabling**: If the path starts with `/sign-up` and the `DISABLE_REGISTRATION` environment variable is truthy, it throws an error, preventing new user registrations.
    *   **Email/Domain Whitelisting**: If the path starts with `/sign-in` or `/sign-up` AND `ALLOWED_LOGIN_EMAILS` or `ALLOWED_LOGIN_DOMAINS` environment variables are set:
        *   It extracts the email from the request body.
        *   It checks if the email is in the `ALLOWED_LOGIN_EMAILS` list (comma-separated).
        *   If not allowed by email, it checks if the email's domain is in the `ALLOWED_LOGIN_DOMAINS` list (comma-separated).
        *   If the email or domain is not found in the allowed lists, it throws an "Access restricted" error.
    *   `return`: If no errors are thrown, the middleware completes successfully, and the authentication process continues.

#### D. Plugins

Plugins extend `better-auth`'s functionality. This section contains a substantial amount of configuration for various integrations.

```typescript
  plugins: [
    nextCookies(),
    oneTimeToken({
      expiresIn: 24 * 60 * 60, // 24 hours - Socket.IO handles connection persistence with heartbeats
    }),
    customSession(async ({ user, session }) => ({
      user,
      session,
    })),
    emailOTP({
      sendVerificationOTP: async (data: {
        email: string
        otp: string
        type: 'sign-in' | 'email-verification' | 'forget-password'
      }) => {
        if (!isEmailVerificationEnabled) {
          logger.info('Skipping email verification')
          return
        }
        try {
          if (!data.email) {
            throw new Error('Email is required')
          }

          const validation = quickValidateEmail(data.email)
          if (!validation.isValid) {
            logger.warn('Email validation failed', {
              email: data.email,
              reason: validation.reason,
              checks: validation.checks,
            })
            throw new Error(
              validation.reason ||
                "We are unable to deliver the verification email to that address. Please make sure it's valid and able to receive emails."
            )
          }

          const html = await renderOTPEmail(data.otp, data.email, data.type)

          const result = await sendEmail({
            to: data.email,
            subject: getEmailSubject(data.type),
            html,
            from: getFromEmailAddress(),
            emailType: 'transactional',
          })

          if (!result.success && result.message.includes('no email service configured')) {
            logger.info('🔑 VERIFICATION CODE FOR LOGIN/SIGNUP', {
              email: data.email,
              otp: data.otp,
              type: data.type,
              validation: validation.checks,
            })
            return
          }

          if (!result.success) {
            throw new Error(`Failed to send verification code: ${result.message}`)
          }
        } catch (error) {
          logger.error('Error sending verification code:', {
            error,
            email: data.email,
          })
          throw error
        }
      },
      sendVerificationOnSignUp: false,
      otpLength: 6, // Explicitly set the OTP length
      expiresIn: 15 * 60, // 15 minutes in seconds
    }),
    // ... many genericOAuth providers
    // ... conditional SSO plugin
    // ... conditional Stripe plugin
    // ... conditional Organization plugin
  ],
  // ... rest of the config
```

*   **`nextCookies()`**: Integrates `better-auth` with Next.js's cookie handling for sessions.
*   **`oneTimeToken({ expiresIn: 24 * 60 * 60 })`**: Configures a plugin for generating and managing one-time tokens, typically used for stateless authentication or specific integrations (like Socket.IO connections). The token expires in 24 hours.
*   **`customSession(async ({ user, session }) => ({ user, session }))`**: A plugin that allows customizing the structure or content of the session object. In this case, it's a passthrough, meaning it returns the user and session objects as they are, but demonstrates the capability to add/modify data.
*   **`emailOTP(...)`**: Configures the email One-Time Password (OTP) functionality.
    *   `sendVerificationOTP`: This is a critical custom function for sending OTP emails.
        *   It first checks `isEmailVerificationEnabled` and skips if disabled.
        *   Validates the email using `quickValidateEmail`.
        *   Renders an OTP email using `renderOTPEmail` with the OTP, email, and type (sign-in, verification, password reset).
        *   Sends the email via `sendEmail`.
        *   **Important Fallback**: If `sendEmail` fails and the message indicates "no email service configured," it logs the OTP directly to the console. This is incredibly useful for local development and testing when an actual email service might not be set up.
        *   Throws an error if email sending fails for other reasons.
        *   Includes comprehensive error logging.
    *   `sendVerificationOnSignUp: false`: Explicitly disables OTP sending on sign-up (likely handled elsewhere or by specific flows).
    *   `otpLength: 6`: Sets the OTP to be 6 digits long.
    *   `expiresIn: 15 * 60`: Sets the OTP expiry to 15 minutes.

#### E. `genericOAuth` Plugin (Simplified Explanation)

This is the largest and most complex part of the configuration, supporting a vast array of third-party OAuth 2.0 and OpenID Connect providers.

```typescript
    genericOAuth({
      config: [
        // GitHub Repository Access
        { /* ... GitHub-repo config ... */ },

        // Multiple Google Providers for various scopes
        { /* ... google-email config ... */ },
        { /* ... google-calendar config ... */ },
        { /* ... google-drive config ... */ },
        { /* ... google-docs config ... */ },
        { /* ... google-sheets config ... */ },
        { /* ... google-forms config ... */ },
        { /* ... google-vault config ... */ },

        // Multiple Microsoft Providers for various scopes
        { /* ... microsoft-teams config ... */ },
        { /* ... microsoft-excel config ... */ },
        { /* ... microsoft-planner config ... */ },
        { /* ... outlook config ... */ },
        { /* ... onedrive config ... */ },
        { /* ... sharepoint config ... */ },

        // Wealthbox (CRM)
        { /* ... wealthbox config ... */ },

        // Supabase
        { /* ... supabase config ... */ },

        // X (Twitter)
        { /* ... x config ... */ },

        // Confluence (Atlassian)
        { /* ... confluence config ... */ },

        // Discord
        { /* ... discord config ... */ },

        // Jira (Atlassian)
        { /* ... jira config ... */ },

        // Airtable
        { /* ... airtable config ... */ },

        // Notion
        { /* ... notion config ... */ },

        // Reddit
        { /* ... reddit config ... */ },

        // Linear (Issue Tracking)
        { /* ... linear config ... */ },

        // Slack
        { /* ... slack config ... */ },
      ],
    }),
    // ... conditional SSO, Stripe, Organization plugins
```

The `genericOAuth` plugin allows defining custom OAuth 2.0 configurations for providers not directly supported by `better-auth`'s `socialProviders` or for specific use cases requiring different scopes or `redirectURI`s for the same provider (e.g., multiple Google integrations).

Each object in the `config` array represents a single OAuth provider integration. Common properties for each provider include:

*   **`providerId`**: A unique identifier for this specific integration (e.g., `github-repo`, `google-calendar`).
*   **`clientId`, `clientSecret`**: The application's credentials for the OAuth provider, loaded from environment variables.
*   **`authorizationUrl`**: The URL where the user is redirected to authorize the application.
*   **`tokenUrl`**: The URL used to exchange the authorization code for access and refresh tokens.
*   **`userInfoUrl`**: The URL used to fetch user profile information after obtaining tokens.
*   **`scopes`**: An array of permissions requested from the user. These vary greatly depending on the provider and the specific data/actions the application needs.
*   **`redirectURI`**: The URL where the user is redirected after authorizing the application. This must match the configured redirect URI in the OAuth provider's settings.
*   **`responseType`, `accessType`, `prompt`, `authentication`, `pkce`**: Standard OAuth/OpenID Connect parameters controlling the flow (e.g., `code` for authorization code flow, `offline` for refresh tokens, `consent` to always prompt the user, `pkce` for enhanced security).
*   **`getUserInfo` (Custom Logic)**: This is a crucial and often custom function for many providers.
    *   For some providers (e.g., Wealthbox, Supabase, Slack), there might not be a standard `userInfoUrl` that returns all necessary user details, or the application might only need specific details.
    *   This function takes the `tokens` (access token, refresh token, id token) and constructs a `User` object (ID, name, email, image, etc.) based on information derived from the tokens or additional API calls.
    *   **Example: GitHub-repo** makes a `fetch` call to `https://api.github.com/user` and potentially `https://api.github.com/user/emails` to get the primary email.
    *   **Example: Wealthbox/Supabase/Slack** generate a dummy email/name and a unique ID, often extracting details from the `idToken` if available, because these might be integrations for API access rather than full user logins.
    *   **Example: Linear** uses a GraphQL API call to fetch viewer information.
    *   This demonstrates the flexibility of `genericOAuth` to handle various provider eccentricities.

#### F. Conditional Plugins (`sso`, `stripe`, `organization`)

These plugins are only included in the `better-auth` configuration if specific environment conditions are met.

```typescript
    // Include SSO plugin when enabled
    ...(env.SSO_ENABLED ? [sso()] : []),
    // Only include the Stripe plugin when billing is enabled
    ...(isBillingEnabled && stripeClient
      ? [
          stripe({
            stripeClient,
            stripeWebhookSecret: env.STRIPE_WEBHOOK_SECRET || '',
            createCustomerOnSignUp: true,
            onCustomerCreate: async ({ stripeCustomer, user }) => { /* ... */ },
            subscription: { /* ... */ },
            onEvent: async (event: Stripe.Event) => { /* ... */ },
          }),
          organization({
            allowUserToCreateOrganization: async (user) => { /* ... */ },
            membershipLimit: 50,
            beforeInvite: async ({ organization }: { organization: { id: string } }) => { /* ... */ },
            sendInvitationEmail: async (data: any) => { /* ... */ },
            organizationCreation: { /* ... */ },
          }),
        ]
      : []),
  ],
  // ... rest of the config
```

*   **`...(env.SSO_ENABLED ? [sso()] : [])`**:
    *   This uses a JavaScript spread syntax (`...`) and a ternary operator.
    *   If the `SSO_ENABLED` environment variable is truthy, the `sso()` plugin (for Single Sign-On) is added to the `plugins` array. Otherwise, an empty array is added, effectively skipping the plugin.
*   **`...(isBillingEnabled && stripeClient ? [stripe({...}), organization({...})] : [])`**:
    *   This is another conditional plugin block.
    *   If `isBillingEnabled` is true AND `stripeClient` was successfully initialized (meaning `STRIPE_SECRET_KEY` was provided), then both the `stripe()` and `organization()` plugins are added.
    *   **`stripe({...})`**: This configures the integration with Stripe for billing.
        *   `stripeClient`: The initialized Stripe client instance.
        *   `stripeWebhookSecret`: The secret for verifying Stripe webhooks, loaded from environment variables.
        *   `createCustomerOnSignUp: true`: Automatically creates a Stripe customer record when a new user signs up.
        *   `onCustomerCreate`: A hook that runs after a Stripe customer is created. Logs the event.
        *   **`subscription`**: A sub-object specifically for managing subscriptions.
            *   `enabled: true`: Activates subscription features.
            *   `plans: getPlans()`: Retrieves the available subscription plans from a utility function.
            *   `authorizeReference`: Custom logic to authorize a subscription based on a reference ID (e.g., user ID).
            *   `getCheckoutSessionParams`: Allows customizing parameters for Stripe Checkout sessions (e.g., setting adjustable quantity for a 'team' plan).
            *   `onSubscriptionComplete`, `onSubscriptionUpdate`, `onSubscriptionDeleted`: These are crucial lifecycle hooks that execute custom logic when a Stripe subscription changes its state. They log events and call various billing utility functions (e.g., `handleSubscriptionCreated`, `syncSubscriptionUsageLimits`, `sendPlanWelcomeEmail`, `handleSubscriptionDeleted`) to keep the application's database in sync and perform post-event actions.
        *   **`onEvent`**: This is a global webhook handler for *all* Stripe events.
            *   It logs the received webhook event type.
            *   Uses a `switch` statement to handle specific event types:
                *   `invoice.payment_succeeded`, `invoice.payment_failed`, `invoice.finalized`: Calls corresponding utility functions (`handleInvoicePaymentSucceeded`, etc.).
                *   `customer.subscription.created`: Specifically calls `handleManualEnterpriseSubscription`, indicating that initial subscription creation (e.g., for enterprise plans that aren't self-service checkout) is handled here.
            *   Notes that `customer.subscription.deleted` is handled by the `onSubscriptionDeleted` callback within the `subscription` object, not here.
            *   Logs and handles errors for webhook processing.
    *   **`organization({...})`**: This plugin manages organizational structures within the application, likely tied to billing and member management.
        *   `allowUserToCreateOrganization`: A function that determines if a user is allowed to create a new organization. It checks the database for active 'team' or 'enterprise' subscriptions associated with the user.
        *   `membershipLimit: 50`: A default or soft limit for the number of members in an organization.
        *   **`beforeInvite`**: A critical hook that runs *before* an invitation is sent to a new member.
            *   It checks the organization's active 'team' or 'enterprise' subscriptions to determine the `seatLimit`.
            *   It then counts existing members and pending invitations.
            *   If the `totalCount` (members + pending invites) exceeds the `seatLimit`, it throws an error, preventing the invitation from being sent. This enforces billing-related seat limits.
        *   `sendInvitationEmail`: A custom function to send organization invitation emails. It renders an invitation email using `renderInvitationEmail` with details like inviter name, organization name, invite URL, and the invitee's email, then sends it via `sendEmail`.
        *   `organizationCreation.afterCreate`: A hook that runs after an organization is successfully created, primarily used for logging.

#### G. Page Routes

```typescript
  pages: {
    signIn: '/login',
    signUp: '/signup',
    error: '/error',
    verify: '/verify',
  },
}) // Closes betterAuth call
```

*   **`pages`**: Defines the application's custom routes for various authentication states, allowing `better-auth` to redirect users to these specific pages.
    *   `signIn`: The login page.
    *   `signUp`: The registration page.
    *   `error`: A general error page.
    *   `verify`: A page for email verification or OTP entry.

---

### 4. Exports

These functions provide convenient ways to interact with the configured authentication system from other parts of the Next.js application.

```typescript
export async function getSession() {
  const hdrs = await headers()
  return await auth.api.getSession({
    headers: hdrs,
  })
}

export const signIn = auth.api.signInEmail
export const signUp = auth.api.signUpEmail
```

*   **`export async function getSession()`**:
    *   This asynchronous function retrieves the current user session.
    *   `const hdrs = await headers()`: It fetches the incoming request headers using Next.js's `headers()` utility. These headers contain the session cookie.
    *   `return await auth.api.getSession({ headers: hdrs })`: It calls the `getSession` method from `better-auth`'s API, passing the request headers so the library can extract and validate the session cookie. This returns the active session data (or `null` if no session).
*   **`export const signIn = auth.api.signInEmail`**: Exports a direct reference to `better-auth`'s email sign-in function. This simplifies calling the sign-in logic from components or API routes.
*   **`export const signUp = auth.api.signUpEmail`**: Exports a direct reference to `better-auth`'s email sign-up function, similar to `signIn`.

---

## Simplified Complex Logic

1.  **Centralized Auth Configuration**: Instead of spreading authentication logic across many files, this single file orchestrates everything. It tells the `better-auth` library exactly how to behave for users, databases, emails, and external services.
2.  **Modular Features with Plugins**: Features like social logins, email OTP, Stripe billing, and organization management aren't built from scratch. They're enabled and configured using `better-auth`'s powerful plugin system, making the setup much cleaner and extensible.
3.  **Smart Session Management**: Sessions aren't just simple cookies. They have expiry dates, refresh mechanisms, and even a fast cache to ensure users stay logged in efficiently without constantly hitting the database.
4.  **Database Hooks for Business Logic**: The system automatically runs custom code *after* a new user is created (e.g., setting up billing data) or *before* a session starts (e.g., figuring out which organization a user is active in). This keeps related logic close to the data events.
5.  **Robust Email Handling**: Password resets and OTPs use email templates and a dedicated email sending service, with smart fallback logging to the console if the email service isn't fully configured (great for local testing!).
6.  **Extensive Third-Party Integrations (`genericOAuth`)**: Instead of hand-coding OAuth flows for dozens of services, `better-auth` handles the complex parts. For unique requirements (like specific API calls to get user info), custom `getUserInfo` functions are provided.
7.  **Conditional Feature Activation**: Billing and Single Sign-On are only enabled if specific environment variables are set. This allows for flexible deployments (e.g., a free tier app without billing, or a dev environment without SSO).
8.  **Subscription & Organization Enforcement**: The Stripe and Organization plugins work together to ensure that user access, seat limits, and features are correctly tied to active subscriptions, preventing users from exceeding their plan's allowances. For example, `beforeInvite` explicitly checks remaining seats before allowing a new invitation.

In essence, this file leverages the `better-auth` library to provide a comprehensive, secure, and highly customizable authentication and authorization solution, integrating deeply with the application's database, email services, and various third-party platforms for a seamless user experience.