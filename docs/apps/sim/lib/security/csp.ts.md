Okay, let's break down this TypeScript code, which focuses on Content Security Policy (CSP) configuration.  CSP is a crucial security feature for web applications, mitigating the risk of XSS (Cross-Site Scripting) attacks by controlling the resources the browser is allowed to load.

**1. Purpose of this file:**

This file defines and manages the Content Security Policy (CSP) for a web application.  It provides:

*   **Configuration:**  Defines the allowed sources for various resource types (scripts, styles, images, etc.).
*   **Build-time and Runtime CSP:**  Supports both static CSP directives configured at build time (e.g., in `next.config.js`) and dynamic CSP that can be generated at runtime, incorporating environment variables.
*   **Utility Functions:**  Offers functions to build CSP strings, generate runtime CSP policies, and modify build-time CSP directives.
*   **Specific CSP policies:** Defines CSP policies for workflow execution endpoints.

**2. Simplifying Complex Logic**

The code aims to achieve a balance between strict security and application functionality. The central concept revolves around whitelisting trusted sources for different asset types to prevent the browser from loading malicious content. Dynamic CSP generation accounts for runtime variations (environment variables).

**3. Code Explanation (Line by Line):**

```typescript
import { env, getEnv } from '../env'
```

*   **`import { env, getEnv } from '../env'`:**  This line imports two things from a module located at `'../env'`.  We can infer that this module likely handles environment variable loading and management.
    *   `env`:  Probably a plain JavaScript object containing environment variables loaded at build time (or application start).
    *   `getEnv`:  Likely a function that retrieves environment variables, possibly with some default value handling or type checking. This is the *preferred* way to access env vars at runtime, so they can be dynamically injected.

```typescript
/**
 * Content Security Policy (CSP) configuration builder
 */
```

*   **`/** ... */`:** This is a JSDoc-style comment, providing a brief description of the file's purpose.

```typescript
function getHostnameFromUrl(url: string | undefined): string[] {
  if (!url) return []
  try {
    return [`https://${new URL(url).hostname}`]
  } catch {
    return []
  }
}
```

*   **`function getHostnameFromUrl(url: string | undefined): string[] { ... }`:** This defines a helper function named `getHostnameFromUrl`. It takes a URL string (or `undefined`) as input and returns an array of strings containing the hostname (including `https://` protocol).
    *   **`url: string | undefined`:**  The `url` parameter is typed as either a string or `undefined`.  This handles cases where the environment variable might not be set.
    *   **`if (!url) return []`:** If the URL is `undefined` or an empty string, the function immediately returns an empty array (`[]`).  This prevents errors when processing missing URLs.
    *   **`try { ... } catch { ... }`:**  A `try...catch` block is used for error handling.  Parsing a URL can fail if the input string is not a valid URL.
    *   **`return [`https://${new URL(url).hostname}`]`:** Inside the `try` block:
        *   `new URL(url)`: This creates a `URL` object from the input string.  The `URL` constructor automatically parses the URL into its components (protocol, hostname, path, etc.).
        *   `.hostname`: This accesses the hostname part of the parsed URL.
        *   ``https://${new URL(url).hostname}``:  This constructs a new string with the `https://` protocol prepended to the hostname.  This ensures that the CSP directive will allow connections only over HTTPS for that domain.
        *   `[`...`]` This puts the hostname into an array.
    *   **`return []`:**  If the `URL` constructor throws an error (meaning the URL is invalid), the `catch` block is executed, and the function returns an empty array.
    *   **`string[]`:**  The function's return type is `string[]`, indicating that it always returns an array of strings.  This array will either contain the extracted hostname or be empty if the input was invalid or missing.

```typescript
export interface CSPDirectives {
  'default-src'?: string[]
  'script-src'?: string[]
  'style-src'?: string[]
  'img-src'?: string[]
  'media-src'?: string[]
  'font-src'?: string[]
  'connect-src'?: string[]
  'frame-src'?: string[]
  'frame-ancestors'?: string[]
  'form-action'?: string[]
  'base-uri'?: string[]
  'object-src'?: string[]
}
```

*   **`export interface CSPDirectives { ... }`:**  This defines a TypeScript interface named `CSPDirectives`.  An interface describes the shape of an object.  In this case, it defines the structure for representing CSP directives.
    *   Each property in the interface corresponds to a specific CSP directive (e.g., `'default-src'`, `'script-src'`, `'style-src'`).
    *   The value of each property is `string[] | undefined`. This means that each directive can be associated with an array of strings (representing allowed sources) or be `undefined` if the directive is not explicitly set.  The `?` makes each property optional.
    *   **CSP Directives:**
        *   `default-src`:  Fallback for other directives when they are not explicitly defined.
        *   `script-src`:  Allowed sources for JavaScript code.
        *   `style-src`:  Allowed sources for CSS stylesheets.
        *   `img-src`:  Allowed sources for images.
        *   `media-src`:  Allowed sources for audio and video.
        *   `font-src`:  Allowed sources for fonts.
        *   `connect-src`:  Allowed sources for network requests (AJAX, WebSockets, etc.).  **Crucial for controlling API calls.**
        *   `frame-src`:  Allowed sources for `<frame>`, `<iframe>`, and `<object>` elements.
        *   `frame-ancestors`:  Specifies valid parents that may embed a page using `<frame>`, `<iframe>`, `<object>`, `<embed>`, or `<applet>`.
        *   `form-action`:  Allowed URLs for form submissions.
        *   `base-uri`:  Allowed URLs for the `<base>` element.
        *   `object-src`: Allowed sources for the `<object>`, `<embed>`, and `<applet>` elements.  Setting this to `'none'` is a good security practice.

```typescript
// Build-time CSP directives (for next.config.ts)
export const buildTimeCSPDirectives: CSPDirectives = {
  'default-src': ["'self'"],

  'script-src': [
    "'self'",
    "'unsafe-inline'",
    "'unsafe-eval'",
    'https://*.google.com',
    'https://apis.google.com',
  ],

  'style-src': ["'self'", "'unsafe-inline'", 'https://fonts.googleapis.com'],

  'img-src': [
    "'self'",
    'data:',
    'blob:',
    'https://*.googleusercontent.com',
    'https://*.google.com',
    'https://*.atlassian.com',
    'https://cdn.discordapp.com',
    'https://*.githubusercontent.com',
    'https://*.s3.amazonaws.com',
    'https://s3.amazonaws.com',
    'https://github.com/*',
    ...(env.S3_BUCKET_NAME && env.AWS_REGION
      ? [`https://${env.S3_BUCKET_NAME}.s3.${env.AWS_REGION}.amazonaws.com`]
      : []),
    ...(env.S3_KB_BUCKET_NAME && env.AWS_REGION
      ? [`https://${env.S3_KB_BUCKET_NAME}.s3.${env.AWS_REGION}.amazonaws.com`]
      : []),
    ...(env.S3_CHAT_BUCKET_NAME && env.AWS_REGION
      ? [`https://${env.S3_CHAT_BUCKET_NAME}.s3.${env.AWS_REGION}.amazonaws.com`]
      : []),
    'https://*.amazonaws.com',
    'https://*.blob.core.windows.net',
    'https://github.com/*',
    ...getHostnameFromUrl(env.NEXT_PUBLIC_BRAND_LOGO_URL),
    ...getHostnameFromUrl(env.NEXT_PUBLIC_BRAND_FAVICON_URL),
  ],

  'media-src': ["'self'", 'blob:'],

  'font-src': ["'self'", 'https://fonts.gstatic.com'],

  'connect-src': [
    "'self'",
    env.NEXT_PUBLIC_APP_URL || '',
    env.OLLAMA_URL || 'http://localhost:11434',
    env.NEXT_PUBLIC_SOCKET_URL || 'http://localhost:3002',
    env.NEXT_PUBLIC_SOCKET_URL?.replace('http://', 'ws://').replace('https://', 'wss://') ||
      'ws://localhost:3002',
    'https://api.browser-use.com',
    'https://api.exa.ai',
    'https://api.firecrawl.dev',
    'https://*.googleapis.com',
    'https://*.amazonaws.com',
    'https://*.s3.amazonaws.com',
    'https://*.blob.core.windows.net',
    'https://*.atlassian.com',
    'https://*.supabase.co',
    'https://api.github.com',
    'https://github.com/*',
    ...getHostnameFromUrl(env.NEXT_PUBLIC_BRAND_LOGO_URL),
    ...getHostnameFromUrl(env.NEXT_PUBLIC_PRIVACY_URL),
    ...getHostnameFromUrl(env.NEXT_PUBLIC_TERMS_URL),
  ],

  'frame-src': ['https://drive.google.com', 'https://docs.google.com', 'https://*.google.com'],

  'frame-ancestors': ["'self'"],
  'form-action': ["'self'"],
  'base-uri': ["'self'"],
  'object-src': ["'none'"],
}
```

*   **`export const buildTimeCSPDirectives: CSPDirectives = { ... }`:**  This declares a constant variable named `buildTimeCSPDirectives`.  It's `export`ed, meaning it can be used by other modules.  It's assigned an object that conforms to the `CSPDirectives` interface. This object contains the *default*, build-time CSP configuration.  These are settings that are known *before* the application runs.

    *   **`'default-src': ["'self'"]`:**  The `default-src` directive is set to `["'self'"]`.  `'self'` means that resources can only be loaded from the same origin (domain, protocol, and port) as the web page itself.
    *   **`'script-src': [ ... ]`:**  The `script-src` directive specifies the allowed sources for JavaScript code.
        *   `'self'`:  Allows scripts from the same origin.
        *   `'unsafe-inline'`:  Allows inline JavaScript (JavaScript code embedded directly within HTML `<script>` tags or event attributes).  **This should generally be avoided for security reasons.**
        *   `'unsafe-eval'`:  Allows the use of `eval()` and related functions (e.g., `new Function()`).  **This should also be avoided for security reasons.**
        *   `https://*.google.com`, `https://apis.google.com`:  Allows scripts from Google domains.  The `*.` is a wildcard, allowing any subdomain.
    *   **`'style-src': [ ... ]`:** The `style-src` directive specifies allowed sources for stylesheets.
        *   `'self'`: Allows styles from the same origin.
        *   `'unsafe-inline'`: Allows inline styles (styles defined in `<style>` tags or `style` attributes).  **Generally avoid if possible.**
        *   `https://fonts.googleapis.com`: Allows stylesheets from Google Fonts.
    *   **`'img-src': [ ... ]`:**  The `img-src` directive specifies allowed sources for images.
        *   `'self'`: Allows images from the same origin.
        *   `data:`:  Allows images embedded as data URLs (e.g., `data:image/png;base64,...`).
        *   `blob:`:  Allows images loaded from `blob:` URLs (often used for dynamically created images).
        *   Various `https://*.domain.com` entries:  Allows images from specific domains and their subdomains.
        *   `...(env.S3_BUCKET_NAME && env.AWS_REGION ? [`https://${env.S3_BUCKET_NAME}.s3.${env.AWS_REGION}.amazonaws.com`] : [])`: This is a conditional expression.  If both `env.S3_BUCKET_NAME` and `env.AWS_REGION` are defined (truthy), it adds a source for an S3 bucket based on those environment variables. Otherwise, it adds an empty array (effectively adding nothing). This is repeated for `S3_KB_BUCKET_NAME` and `S3_CHAT_BUCKET_NAME`.
        *   `...getHostnameFromUrl(env.NEXT_PUBLIC_BRAND_LOGO_URL)`: This uses the `getHostnameFromUrl` function to extract the hostname from the `NEXT_PUBLIC_BRAND_LOGO_URL` environment variable and adds it to the list of allowed image sources. The spread operator (`...`) is used to insert the elements of the array returned by `getHostnameFromUrl` directly into the `img-src` array.
        *   `...getHostnameFromUrl(env.NEXT_PUBLIC_BRAND_FAVICON_URL)`: Same as above, but for the favicon URL.
    *   **`'media-src': ["'self'", 'blob:']`:** Allows media from the same origin and `blob:` URLs.
    *   **`'font-src': ["'self'", 'https://fonts.gstatic.com']`:** Allows fonts from the same origin and Google Fonts.
    *   **`'connect-src': [ ... ]`:**  The `connect-src` directive specifies the allowed sources for network connections (AJAX, WebSockets, etc.).  **This is a critical security directive.**
        *   `'self'`: Allows connections to the same origin.
        *   `env.NEXT_PUBLIC_APP_URL || ''`: Allows connections to the URL specified in the `NEXT_PUBLIC_APP_URL` environment variable. If the variable is not set, it defaults to an empty string (which might need further adjustment depending on the desired behavior).
        *   `env.OLLAMA_URL || 'http://localhost:11434'`: Allows connections to the OLLAMA URL specified in the `OLLAMA_URL` environment variable. If not set defaults to local host.
        *   `env.NEXT_PUBLIC_SOCKET_URL || 'http://localhost:3002'`: Allows connections to the socket URL.
        *   `env.NEXT_PUBLIC_SOCKET_URL?.replace('http://', 'ws://').replace('https://', 'wss://') || 'ws://localhost:3002'`:  This line dynamically constructs a WebSocket URL from the `NEXT_PUBLIC_SOCKET_URL` environment variable.  It replaces `http://` with `ws://` and `https://` with `wss://` to ensure that the WebSocket connection uses the appropriate protocol. If the original environment variable is not set, it defaults to `ws://localhost:3002`.
        *   Various `https://*.domain.com` entries:  Allows connections to specific domains and their subdomains.
        *   `...getHostnameFromUrl(env.NEXT_PUBLIC_BRAND_LOGO_URL)`:  Dynamically allows connections to the brand logo URL.
        *   `...getHostnameFromUrl(env.NEXT_PUBLIC_PRIVACY_URL)`: Dynamically allows connection to the privacy policy URL.
        *   `...getHostnameFromUrl(env.NEXT_PUBLIC_TERMS_URL)`: Dynamically allows connections to the terms of service URL.
    *   **`'frame-src': [ ... ]`:**  The `frame-src` directive specifies the allowed sources for `<frame>`, `<iframe>`, and `<object>` elements.
        *   Allows embedding frames from Google Drive, Google Docs and any subdomain of google.com.
    *   **`'frame-ancestors': ["'self'"]`:** This directive specifies the allowed origins for pages that can embed the current page in a `<frame>`, `<iframe>`, `<object>`, `<embed>`, or `<applet>`.  `'self'` means that only pages from the same origin can embed the current page.  This helps prevent clickjacking attacks.
    *   **`'form-action': ["'self'"]`:**  The `form-action` directive specifies the allowed URLs for form submissions.  `'self'` means that forms can only be submitted to the same origin.
    *   **`'base-uri': ["'self'"]`:**  The `base-uri` directive specifies the allowed URLs for the `<base>` element. `'self'` means that the `<base>` element can only specify a URL within the same origin.
    *   **`'object-src': ["'none'"]`:** The `object-src` directive controls the sources allowed for `<object>`, `<embed>`, and `<applet>` elements. Setting it to `['none']` effectively disables these elements, which is a good security practice because they can be used to load potentially malicious content.

```typescript
/**
 * Build CSP string from directives object
 */
export function buildCSPString(directives: CSPDirectives): string {
  return Object.entries(directives)
    .map(([directive, sources]) => {
      if (!sources || sources.length === 0) return ''
      const validSources = sources.filter((source: string) => source && source.trim() !== '')
      if (validSources.length === 0) return ''
      return `${directive} ${validSources.join(' ')}`
    })
    .filter(Boolean)
    .join('; ')
}
```

*   **`export function buildCSPString(directives: CSPDirectives): string { ... }`:**  This defines a function named `buildCSPString` that takes a `CSPDirectives` object as input and returns a string representing the complete CSP header value. This function is responsible for converting the structured `CSPDirectives` object into a string that can be used as the value of the `Content-Security-Policy` HTTP header.

    *   **`Object.entries(directives)`:**  This converts the `directives` object into an array of key-value pairs (entries).  Each entry is a two-element array where the first element is the directive name (e.g., `'script-src'`) and the second element is the array of sources (e.g., `["'self'", 'https://example.com']`).
    *   **`.map(([directive, sources]) => { ... })`:** This maps over the array of entries, transforming each entry into a string representing a single CSP directive.
        *   **`if (!sources || sources.length === 0) return ''`:** If the `sources` array is `undefined`, `null`, or empty, the function returns an empty string. This prevents directives with no sources from being included in the CSP header.
        *   **`const validSources = sources.filter((source: string) => source && source.trim() !== '')`:** This line filters the `sources` array to remove any empty or whitespace-only strings. This ensures that only valid sources are included in the CSP directive.
            *   `source && source.trim() !== ''` checks that the `source` is not null/undefined (truthy) and, after trimming whitespace, is not an empty string.
        *   **`if (validSources.length === 0) return ''`:** If after filtering, the `validSources` array is empty, it returns an empty string.
        *   **`return `${directive} ${validSources.join(' ')}``:**  This constructs the CSP directive string by concatenating the directive name, a space, and the sources joined by spaces. For example, if `directive` is `'script-src'` and `validSources` is `["'self'", 'https://example.com']`, the resulting string will be `'script-src' 'self' https://example.com`.
    *   **`.filter(Boolean)`:**  This filters the array of directive strings, removing any empty strings (which were returned when a directive had no valid sources). `Boolean` is used as a shorthand; it converts each element to a boolean, and only truthy values are kept.
    *   **`.join('; ')`:** This joins the remaining directive strings together with the separator `'; '` to create the final CSP header value.

```typescript
/**
 * Generate runtime CSP header with dynamic environment variables (safer approach)
 * This maintains compatibility with existing inline scripts while fixing Docker env var issues
 */
export function generateRuntimeCSP(): string {
  const socketUrl = getEnv('NEXT_PUBLIC_SOCKET_URL') || 'http://localhost:3002'
  const socketWsUrl =
    socketUrl.replace('http://', 'ws://').replace('https://', 'wss://') || 'ws://localhost:3002'
  const appUrl = getEnv('NEXT_PUBLIC_APP_URL') || ''
  const ollamaUrl = getEnv('OLLAMA_URL') || 'http://localhost:11434'

  const brandLogoDomains = getHostnameFromUrl(getEnv('NEXT_PUBLIC_BRAND_LOGO_URL'))
  const brandFaviconDomains = getHostnameFromUrl(getEnv('NEXT_PUBLIC_BRAND_FAVICON_URL'))
  const privacyDomains = getHostnameFromUrl(getEnv('NEXT_PUBLIC_PRIVACY_URL'))
  const termsDomains = getHostnameFromUrl(getEnv('NEXT_PUBLIC_TERMS_URL'))

  const allDynamicDomains = [
    ...brandLogoDomains,
    ...brandFaviconDomains,
    ...privacyDomains,
    ...termsDomains,
  ]
  const uniqueDynamicDomains = Array.from(new Set(allDynamicDomains))
  const dynamicDomainsStr = uniqueDynamicDomains.join(' ')
  const brandLogoDomain = brandLogoDomains[0] || ''
  const brandFaviconDomain = brandFaviconDomains[0] || ''

  return `
    default-src 'self';
    script-src 'self' 'unsafe-inline' 'unsafe-eval' https://*.google.com https://apis.google.com;
    style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
    img-src 'self' data: blob: https://*.googleusercontent.com https://*.google.com https://*.atlassian.com https://cdn.discordapp.com https://*.githubusercontent.com ${brandLogoDomain} ${brandFaviconDomain};
    media-src 'self' blob:;
    font-src 'self' https://fonts.gstatic.com;
    connect-src 'self' ${appUrl} ${ollamaUrl} ${socketUrl} ${socketWsUrl} https://api.browser-use.com https://api.exa.ai https://api.firecrawl.dev https://*.googleapis.com https://*.amazonaws.com https://*.s3.amazonaws.com https://*.blob.core.windows.net https://api.github.com https://github.com/* https://*.atlassian.com https://*.supabase.co ${dynamicDomainsStr};
    frame-src https://drive.google.com https://docs.google.com https://*.google.com;
    frame-ancestors 'self';
    form-action 'self';
    base-uri 'self';
    object-src 'none';
  `
    .replace(/\s{2,}/g, ' ')
    .trim()
}
```

*   **`export function generateRuntimeCSP(): string { ... }`:** This defines a function named `generateRuntimeCSP` that generates a CSP string at runtime.  This is important because it allows the CSP to be dynamically configured based on environment variables.
    *   It retrieves the value of several environment variables using `getEnv()`, providing default values if the variables are not set.  This is safer than directly accessing `env` because `getEnv` can handle cases where the env var is missing.
        *   `socketUrl`: The base URL for socket connections.
        *   `socketWsUrl`: The WebSocket URL, derived from `socketUrl`.
        *   `appUrl`: The application URL.
        *   `ollamaUrl`: The URL for the Ollama service.
    *   It uses `getHostnameFromUrl` to extract the hostname from several brand-related URLs (logo, favicon, privacy policy, terms of service).
    *   It then constructs a string that represents the CSP header value.  This string includes both static and dynamic parts.
        *   The dynamic parts are the values of the environment variables and the hostnames extracted from the brand-related URLs.
    *   **.replace(/\s{2,}/g, ' ')**: This replaces multiple spaces with a single space using a regular expression. This cleans up the generated CSP string, making it more readable and preventing potential parsing issues.
    *   **.trim()**: This removes any leading or trailing whitespace from the CSP string. This ensures that the CSP string is a valid value for the `Content-Security-Policy` header.
    *   **Why Runtime CSP?**  Using a runtime CSP is generally preferred for values that are not known at build time. This is essential for applications deployed in different environments with varying URLs or API endpoints.

```typescript
/**
 * Get the main CSP policy string (build-time)
 */
export function getMainCSPPolicy(): string {
  return buildCSPString(buildTimeCSPDirectives)
}
```

*   **`export function getMainCSPPolicy(): string { ... }`:** This function is a simple wrapper that returns the CSP string generated from the `buildTimeCSPDirectives` object using the `buildCSPString` function.  This is the main CSP policy that's applied to the application.

```typescript
/**
 * Permissive CSP for workflow execution endpoints
 */
export function getWorkflowExecutionCSPPolicy(): string {
  return "default-src * 'unsafe-inline' 'unsafe-eval'; connect-src *;"
}
```

*   **`export function getWorkflowExecutionCSPPolicy(): string { ... }`:** This function returns a *very* permissive CSP string.  It's specifically designed for workflow execution endpoints.

    *   **`default-src * 'unsafe-inline' 'unsafe-eval'; connect-src *;`:** This CSP allows everything from any source (`*`), including inline scripts and `eval()`.  This is **extremely insecure** and should only be used in very specific, controlled environments where the risks are well understood and mitigated.  It's likely used for rapid prototyping or testing where security is not the primary concern.
    *   **Warning:**  Using such a permissive CSP in a production environment is highly discouraged.

```typescript
/**
 * Add a source to a specific directive (modifies build-time directives)
 */
export function addCSPSource(directive: keyof CSPDirectives, source: string): void {
  if (!buildTimeCSPDirectives[directive]) {
    buildTimeCSPDirectives[directive] = []
  }
  if (!buildTimeCSPDirectives[directive]!.includes(source)) {
    buildTimeCSPDirectives[directive]!.push(source)
  }
}
```

*   **`export function addCSPSource(directive: keyof CSPDirectives, source: string): void { ... }`:** This function allows you to add a source to a specific CSP directive at *runtime*.  It modifies the `buildTimeCSPDirectives` object directly.

    *   **`directive: keyof CSPDirectives`:**  The `directive` parameter is typed as `keyof CSPDirectives`.  This means that it can only be one of the property names defined in the `CSPDirectives` interface (e.g., `'script-src'`, `'img-src'`, etc.).  This provides type safety, ensuring that you can't accidentally specify an invalid directive name.
    *   **`source: string`:**  The `source` parameter is the string representing the source to add (e.g., `'https://example.com'`).
    *   **`if (!buildTimeCSPDirectives[directive]) { buildTimeCSPDirectives[directive] = [] }`:**  If the specified directive doesn't already exist in the `buildTimeCSPDirectives` object, this code creates it as an empty array.
    *   **`if (!buildTimeCSPDirectives[directive]!.includes(source)) { buildTimeCSPDirectives[directive]!.push(source) }`:** This line checks if the source is already present in the directive's source array. If it's not, the source is added to the array. The `!` is a non-null assertion operator. It tells the TypeScript compiler that the value is not `null` or `undefined`, even though the type definition allows it. This is used because TypeScript knows `buildTimeCSPDirectives[directive]` could be undefined, but we've already checked for that case in the previous `if` statement.

```typescript
/**
 * Remove a source from a specific directive (modifies build-time directives)
 */
export function removeCSPSource(directive: keyof CSPDirectives, source: string): void {
  if (buildTimeCSPDirectives[directive]) {
    buildTimeCSPDirectives[directive] = buildTimeCSPDirectives[directive]!.filter(
      (s: string) => s !== source
    )
  }
}
```

*   **`export function removeCSPSource(directive: keyof CSPDirectives, source: string): void { ... }`:** This function removes a source from a specific CSP directive.  It also modifies the `buildTimeCSPDirectives` object directly.

    *   **`directive: keyof CSPDirectives`:** Same as in `addCSPSource`.
    *   **`source: string`:** Same as in `addCSPSource`.
    *   **`if (buildTimeCSPDirectives[directive]) { ... }`:** This checks if the directive exists in the `buildTimeCSPDirectives` object.
    *   **`buildTimeCSPDirectives[directive] = buildTimeCSPDirectives[directive]!.filter((s: string) => s !== source)`:** This line filters the source array for the specified directive, removing the specified source. The `!` is the non-null assertion operator (as explained previously). The filter keeps all sources `s` where `s !== source`.

**In Summary:**

This code provides a comprehensive system for managing CSP in a web application. It balances security with flexibility by allowing for both build-time and runtime configuration. The utility functions make it easy to generate, modify, and apply CSP policies. However, special attention should be given to the use of `'unsafe-inline'` and `'unsafe-eval'`, as well as the very permissive `getWorkflowExecutionCSPPolicy`.  These should only be used when absolutely necessary and with a thorough understanding of the security implications.
