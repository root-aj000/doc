This `env.ts` file is a crucial component in a Next.js application, centralizing and validating environment variables for both server-side and client-side code. It uses `@t3-oss/env-nextjs` for powerful type-safe environment variable management, `zod` for schema validation, and `next-runtime-env` to gracefully handle environment variables injected at runtime (e.g., in Docker containers).

---

### **Purpose of this file**

The primary purpose of this `env.ts` file is to:

1.  **Centralize Environment Variable Management:** Provide a single source of truth for all environment variables used throughout the application.
2.  **Ensure Type Safety:** Use `zod` schemas to define the expected type, format, and optionality of each variable. This means if you try to use `env.PORT` it will correctly infer it as a `number | undefined` instead of just `string | undefined`, preventing common runtime errors.
3.  **Validate Variables:** Automatically validate that required environment variables are present and correctly formatted at application startup (or build time).
4.  **Support Universal Access:** Distinguish between server-only variables (like API keys), client-side variables (prefixed with `NEXT_PUBLIC_`), and shared variables, making them accessible in the correct contexts.
5.  **Handle Runtime Environment Injection:** Crucially, it integrates with `next-runtime-env` to allow `NEXT_PUBLIC_` variables to be injected *after* the application is built (e.g., in a Docker container), rather than being hardcoded at build time. This provides flexibility for deployment environments.

In essence, it makes working with environment variables less error-prone, more predictable, and easier to manage in a complex Next.js project.

---

### **Detailed Explanation**

Let's break down the code line by line and section by section.

#### **Imports**

```typescript
import { createEnv } from '@t3-oss/env-nextjs'
import { env as runtimeEnv } from 'next-runtime-env'
import { z } from 'zod'
```

*   `import { createEnv } from '@t3-oss/env-nextjs'`: This line imports the `createEnv` function from the `@t3-oss/env-nextjs` library. This is the core utility that allows you to define and validate your environment variables in a type-safe way for Next.js applications.
*   `import { env as runtimeEnv } from 'next-runtime-env'`: This imports the `env` object from `next-runtime-env`, but renames it to `runtimeEnv` to avoid a naming conflict with the `env` object we'll export later. `next-runtime-env` is specifically designed to make client-side environment variables available *at runtime* rather than being embedded at build time, which is essential for Docker deployments or dynamic environments.
*   `import { z } from 'zod'`: This imports `z` from the `zod` library. Zod is a TypeScript-first schema declaration and validation library. It's used here to define the expected shape and types of each environment variable.

#### **`getEnv` Helper Function**

```typescript
/**
 * Universal environment variable getter that works in both client and server contexts.
 * - Client-side: Uses next-runtime-env for runtime injection (supports Docker runtime vars)
 * - Server-side: Falls back to process.env when runtimeEnv returns undefined
 * - Provides seamless Docker runtime variable support for NEXT_PUBLIC_ vars
 */
const getEnv = (variable: string) => runtimeEnv(variable) ?? process.env[variable]
```

*   This block defines a helper function `getEnv`.
*   **Purpose:** This function is a clever abstraction to retrieve environment variables. It prioritizes variables potentially injected at runtime (via `next-runtime-env`) and falls back to standard `process.env` variables if the runtime-injected one isn't found. This ensures compatibility across different deployment scenarios, especially those involving Docker where client-side variables might be set *after* the build.
*   `const getEnv = (variable: string) => ...`: Declares a constant function `getEnv` that takes a `string` (the name of the environment variable) as an argument.
*   `runtimeEnv(variable)`: Attempts to get the environment variable using the `next-runtime-env` utility. If `next-runtime-env` has this variable (which it would for `NEXT_PUBLIC_` variables injected at runtime), it will return its value.
*   `?? process.env[variable]`: This is the [nullish coalescing operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing). It means: if `runtimeEnv(variable)` returns `null` or `undefined` (i.e., the variable wasn't injected at runtime or it's a server-only variable), then fall back to retrieving the variable directly from `process.env`. This ensures that server-side variables, and client-side variables that *were* available at build time, are still picked up correctly.

#### **`createEnv` Configuration (The Core Logic)**

```typescript
// biome-ignore format: keep alignment for readability
export const env = createEnv({
  skipValidation: true,

  // ... (server, client, shared, experimental__runtimeEnv objects)
})
```

*   `export const env = createEnv({ ... })`: This is the main call to `createEnv`. It defines and configures how environment variables are handled for the entire application. The resulting `env` object will be exported and used throughout your codebase.
*   `skipValidation: true,`: **This is a critical setting for supporting `next-runtime-env`**. Normally, `@t3-oss/env-nextjs` validates all defined environment variables at build time. By setting `skipValidation: true`, we are telling it *not* to strictly validate everything at build time. This is necessary because some variables (especially `NEXT_PUBLIC_` ones meant for client-side) might not be present in `process.env` during the build process if they are intended to be injected *at runtime* (e.g., from Docker environment variables). The validation will effectively happen when the `getEnv` function attempts to retrieve them.

##### **`server` Object (Server-side Only Variables)**

```typescript
  server: {
    // Core Database & Authentication
    DATABASE_URL:                          z.string().url(),                       // Primary database connection string
    BETTER_AUTH_URL:                       z.string().url(),                       // Base URL for Better Auth service
    // ... many more server-side variables
    CRON_SECRET:                           z.string().optional(),                  // Secret for authenticating cron job requests
  },
```

*   This object defines environment variables that are **only available on the server-side**. If you try to access these variables on the client, you'll encounter an error (or `undefined` if accessed via `process.env`).
*   Each key-value pair defines an environment variable and its validation schema using `zod`.
    *   `DATABASE_URL: z.string().url()`: Defines `DATABASE_URL` as a required string that must be a valid URL. If it's missing or not a URL, validation will fail.
    *   `BETTER_AUTH_SECRET: z.string().min(32)`: Defines `BETTER_AUTH_SECRET` as a required string with a minimum length of 32 characters.
    *   `DISABLE_REGISTRATION: z.boolean().optional()`: Defines `DISABLE_REGISTRATION` as an optional boolean. If provided, `zod` will attempt to parse it as a boolean.
    *   `FREE_TIER_COST_LIMIT: z.number().optional()`: Defines `FREE_TIER_COST_LIMIT` as an optional number.
    *   `OVERAGE_THRESHOLD_DOLLARS: z.number().optional().default(50)`: Defines `OVERAGE_THRESHOLD_DOLLARS` as an optional number, and if it's not provided, it defaults to `50`.
    *   `LOG_LEVEL: z.enum(['DEBUG', 'INFO', 'WARN', 'ERROR']).optional()`: Defines `LOG_LEVEL` as an optional string that must be one of the specified enum values.
*   **Simplifying Complex Logic:** The `server` section is a straightforward list of variables. The "complexity" here lies in the sheer volume and categorization. The clear comments and grouping help in understanding which variables relate to which part of the application (e.g., `Core Database & Authentication`, `Copilot`, `Payment & Billing`, `Cloud Storage`). This makes it easier for developers to find and configure the correct variables.

##### **`client` Object (Client-side and Server-side Variables)**

```typescript
  client: {
    // Core Application URLs - Required for frontend functionality
    NEXT_PUBLIC_APP_URL:                   z.string().url(),                       // Base URL of the application (e.g., https://app.sim.ai)

    // Client-side Services
    NEXT_PUBLIC_SOCKET_URL:                z.string().url().optional(),            // WebSocket server URL for real-time features
    
    // ... many more client-side variables
    NEXT_PUBLIC_EMAIL_PASSWORD_SIGNUP_ENABLED: z.boolean().optional().default(true), // Control visibility of email/password login forms
  },
```

*   This object defines environment variables that are **available on both the client-side and server-side**.
*   **Convention:** In Next.js, client-side environment variables *must* be prefixed with `NEXT_PUBLIC_`. This ensures that Next.js correctly exposes them to the browser bundle.
*   The validation schemas (`z.string().url()`, `z.boolean().optional()`, etc.) work the same way as in the `server` object.
*   `NEXT_PUBLIC_APP_URL: z.string().url()`: Defines the base URL of the application as a required URL, accessible everywhere.
*   `NEXT_PUBLIC_BRAND_PRIMARY_COLOR: z.string().regex(/^#[0-9A-Fa-f]{6}$/).optional()`: Defines a custom brand color as an optional string that must match a hexadecimal color format (e.g., `#RRGGBB`).
*   **Feature Flags:** Variables like `NEXT_PUBLIC_BILLING_ENABLED` and `NEXT_PUBLIC_SSO_ENABLED` act as feature flags, allowing features to be toggled on or off via environment variables, impacting UI and client-side logic.

##### **`shared` Object (Variables available on both server and client without `NEXT_PUBLIC_` prefix)**

```typescript
  shared: {
    NODE_ENV:                              z.enum(['development', 'test', 'production']).optional(), // Runtime environment
    NEXT_TELEMETRY_DISABLED:               z.string().optional(),                // Disable Next.js telemetry collection
  },
```

*   This object is for variables that are naturally available in `process.env` on both the server and client (though `NODE_ENV` is usually inlined at build time for the client) and don't require the `NEXT_PUBLIC_` prefix for client-side access.
*   `NODE_ENV: z.enum(['development', 'test', 'production']).optional()`: Defines the runtime environment as an optional enum value. This is a standard Node.js environment variable.

##### **`experimental__runtimeEnv` Object (Enabling Runtime Client Variables)**

```typescript
  experimental__runtimeEnv: {
    NEXT_PUBLIC_APP_URL: process.env.NEXT_PUBLIC_APP_URL,
    NEXT_PUBLIC_BILLING_ENABLED: process.env.NEXT_PUBLIC_BILLING_ENABLED,
    // ... all other NEXT_PUBLIC_ variables and shared variables
    NODE_ENV: process.env.NODE_ENV,
    NEXT_TELEMETRY_DISABLED: process.env.NEXT_TELEMETRY_DISABLED,
  },
```

*   **Simplifying Complex Logic:** This is where the magic for `next-runtime-env` and dynamic client-side variables happens, in conjunction with `skipValidation: true`.
*   **Purpose:** This configuration tells `@t3-oss/env-nextjs` *which* environment variables, specifically those accessible on the client (i.e., `NEXT_PUBLIC_` variables and shared variables), should be handled by the `next-runtime-env` library for runtime injection.
*   **How it Works:** Instead of inlining the values of `NEXT_PUBLIC_` variables into the client-side JavaScript bundle *at build time* (which is Next.js's default behavior), this `experimental__runtimeEnv` object explicitly lists them. By mapping `NEXT_PUBLIC_VAR_NAME: process.env.NEXT_PUBLIC_VAR_NAME`, we are essentially providing a *hint* to `@t3-oss/env-nextjs` and `next-runtime-env`. When `createEnv` is initialized, it will generate a JavaScript file that `next-runtime-env` uses. This file will contain code that fetches these listed `NEXT_PUBLIC_` variables *at runtime* from the server, rather than having them hardcoded in the client bundle.
*   This setup allows you to build your Next.js application once (e.g., in a CI/CD pipeline or Docker image) and then deploy it to multiple environments, configuring client-side `NEXT_PUBLIC_` variables dynamically at startup without rebuilding. This is a common requirement for microservices or containerized deployments.

#### **`isTruthy` and `isFalsy` Utility Functions**

```typescript
// Need this utility because t3-env is returning string for boolean values.
export const isTruthy = (value: string | boolean | number | undefined) =>
  typeof value === 'string' ? value.toLowerCase() === 'true' || value === '1' : Boolean(value)

// Utility to check if a value is explicitly false (defaults to false only if explicitly set)
export const isFalsy = (value: string | boolean | number | undefined) =>
  typeof value === 'string' ? value.toLowerCase() === 'false' || value === '0' : value === false
```

*   **Problem Solved:** Environment variables are always strings when read from `process.env`. Even if you define `z.boolean()`, there are nuances. These utilities are needed because `t3-env` (and `process.env` in general) often returns string representations for boolean values (e.g., `"true"`, `"false"`, `"1"`, `"0"`). Standard `Boolean("false")` evaluates to `true`, which is not what's desired.
*   `export const isTruthy = ...`: This function takes a value that could be a string, boolean, number, or undefined.
    *   `typeof value === 'string'`: Checks if the value is a string.
    *   `value.toLowerCase() === 'true' || value === '1'`: If it's a string, it considers it "truthy" if it's `"true"` (case-insensitive) or `"1"`.
    *   `: Boolean(value)`: If it's not a string, it uses the standard `Boolean()` constructor to convert it (e.g., `true` is `true`, `1` is `true`, `undefined` is `false`).
*   `export const isFalsy = ...`: Similar to `isTruthy`, this function checks for "falsy" values.
    *   `typeof value === 'string'`: Checks if the value is a string.
    *   `value.toLowerCase() === 'false' || value === '0'`: If it's a string, it considers it "falsy" if it's `"false"` (case-insensitive) or `"0"`.
    *   `: value === false`: If it's not a string, it only considers it "falsy" if it's *explicitly* `false` (e.g., `undefined` would not be `false`, but `0` would not be `false` here either. This implies a stricter check for *explicit* boolean `false`).

#### **Re-export `getEnv`**

```typescript
export { getEnv }
```

*   This line re-exports the `getEnv` utility function, making it available for other parts of the application to use if they need the specific `runtimeEnv` fallback logic. However, for variables defined within `createEnv`, accessing `env.YOUR_VAR` is generally preferred for its type safety and validation benefits.

---

### **Simplified Complex Logic**

The most complex parts of this file are:

1.  **Universal Variable Access (`getEnv` and `experimental__runtimeEnv`):**
    *   **The Problem:** Next.js client-side environment variables (`NEXT_PUBLIC_`) are usually *hardcoded* into the JavaScript bundle at build time. This means if you build your app for "development" with `VAR=DEV`, and then deploy that *same build* to production with `VAR=PROD`, the production app would still show `DEV` on the client. For Docker, you often want to build once and configure at runtime.
    *   **The Solution:**
        *   `next-runtime-env` provides a mechanism to fetch these `NEXT_PUBLIC_` variables from the server *after* the client-side bundle loads.
        *   The `experimental__runtimeEnv` block tells `@t3-oss/env-nextjs` *which* `NEXT_PUBLIC_` variables it should *not* hardcode at build time, but instead defer to `next-runtime-env`.
        *   `skipValidation: true` is set because these variables might not be present at build time, and their validation is deferred.
        *   The `getEnv` helper function acts as a smart wrapper that first checks if `next-runtime-env` has the variable (for runtime-injected `NEXT_PUBLIC_` vars) and then falls back to `process.env` (for server-only vars or `NEXT_PUBLIC_` vars that *were* available at build time).

2.  **String-to-Boolean Conversion (`isTruthy`, `isFalsy`):**
    *   **The Problem:** All environment variables come in as strings. Simply casting `Boolean(process.env.VAR)` can lead to incorrect results, as `Boolean("false")` is `true` in JavaScript.
    *   **The Solution:** The `isTruthy` and `isFalsy` functions provide robust, explicit string-to-boolean conversion, handling common string representations like `"true"`, `"false"`, `"1"`, and `"0"`, ensuring that boolean environment variables behave as expected.

By combining `zod`, `@t3-oss/env-nextjs`, and `next-runtime-env` with these custom helpers, the file creates a powerful, type-safe, and flexible environment variable management system crucial for modern Next.js applications deployed in dynamic environments.