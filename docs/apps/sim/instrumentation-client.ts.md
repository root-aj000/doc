This TypeScript file is a client-side telemetry (data collection) script designed for web applications. It's responsible for gathering various events – like user interactions, performance metrics (Web Vitals), and errors – from the user's browser, sanitizing them to remove sensitive information, and then sending them in batches to a server for analysis.

## Purpose of this file

The primary goal of this file is to instrument a web application to collect anonymous usage and performance data. This data helps developers understand how users interact with the application, identify performance bottlenecks, and catch unhandled errors. Key aspects include:

1.  **Event Collection**: Capturing different types of events (e.g., feature usage, performance metrics, errors).
2.  **Batching**: Grouping multiple events together to reduce network requests and improve efficiency.
3.  **Data Sanitization**: Removing potentially sensitive information from events before they are sent, ensuring user privacy.
4.  **Resilient Sending**: Attempting to send data reliably, even when the user is navigating away from the page, using browser features like `navigator.sendBeacon`.
5.  **User Opt-out**: Respecting user preferences and environmental variables to disable telemetry.
6.  **Performance Monitoring**: Collecting important Web Vitals (LCP, CLS, FID) to measure the user experience.
7.  **Error Tracking**: Automatically reporting unhandled JavaScript errors and promise rejections.

## Simplified Complex Logic

At its core, the logic can be simplified into these steps:

1.  **Initialization**:
    *   Check if telemetry should be enabled based on environment variables or user's local storage preference.
    *   Set up constants for batching (size, interval).
    *   Create an empty array (`eventBatch`) to temporarily store events.

2.  **Collecting Events (`addToBatch`)**:
    *   When an event occurs (e.g., a user action, a performance measurement), it's passed to `addToBatch`.
    *   If telemetry is disabled, the event is ignored.
    *   The event is added to the `eventBatch` array.
    *   A timer is started to automatically send the batch after a certain interval (e.g., 10 seconds).
    *   If the batch reaches a maximum size (e.g., 50 events) *before* the timer expires, it's sent immediately.

3.  **Sanitizing Events (`sanitizeEvent`)**:
    *   Before sending, each event is processed by `sanitizeEvent`.
    *   This function looks for common sensitive keywords (like "password", "token") in event keys and string values.
    *   If found, the string values are replaced with `[redacted]`, and entire keys might be skipped. This is a crucial privacy measure.

4.  **Sending Batches (`flushBatch`)**:
    *   This function is called either when the batch size limit is reached, the batch timer expires, or when the user is about to leave the page.
    *   It takes all accumulated events from `eventBatch`, empties the array, and clears any pending timer.
    *   It then sanitizes these events.
    *   It attempts to send the events to the server using `navigator.sendBeacon` (preferred for reliability during page unload) or falls back to a regular `fetch` request.
    *   Errors during sending are silently ignored to avoid disrupting the user experience.

5.  **Browser Integration**:
    *   The script attaches itself to global browser objects (`window`) to expose a tracking function (`__SIM_TRACK_EVENT`) and its enabled status.
    *   It listens for browser events like `beforeunload` (user leaving the page) and `visibilitychange` (page tab hidden) to `flushBatch` ensuring data is sent.
    *   It uses the `PerformanceObserver` API to collect Web Vitals (LCP, CLS, FID) and registers global error handlers to catch and report JavaScript errors and unhandled promise rejections.

## Explanation of Each Line of Code

```typescript
/**
 * Sim Telemetry - Client-side Instrumentation
 */
```
This is a JSDoc comment describing the file's purpose: "Sim Telemetry" (likely "Simulation Telemetry") provides "Client-side Instrumentation" (code to gather data from the user's browser).

```typescript
import { env } from './lib/env'
```
This line imports the `env` object from a local module. This `env` object likely contains environment variables, similar to how Node.js's `process.env` works, but adapted for the client side.

```typescript
if (typeof window !== 'undefined') {
```
This check ensures that the code inside this block only runs in a browser environment, where `window` is defined. This is important for code that might be processed during server-side rendering (SSR) in frameworks like Next.js, where `window` does not exist.

```typescript
  const TELEMETRY_STATUS_KEY = 'simstudio-telemetry-status'
```
Declares a constant string that will be used as the key to store and retrieve the user's telemetry preference in `localStorage`.

```typescript
  const BATCH_INTERVAL_MS = 10000 // Send batches every 10 seconds
```
Declares a constant for the time interval (in milliseconds) after which a batch of events should be sent, if not already sent due to size. Here, it's 10,000 ms, or 10 seconds.

```typescript
  const MAX_BATCH_SIZE = 50 // Max events per batch
```
Declares a constant for the maximum number of events that can be accumulated in the `eventBatch` before it's automatically flushed, regardless of the `BATCH_INTERVAL_MS`.

```typescript
  let telemetryEnabled = true
```
Initializes a mutable variable `telemetryEnabled` to `true`. This variable will control whether telemetry events are collected and sent. It's `true` by default, meaning telemetry is enabled unless explicitly disabled.

```typescript
  const eventBatch: any[] = []
```
Declares a constant array named `eventBatch`. This array will temporarily hold events collected from the application before they are sent to the server. The `any[]` type indicates it can hold elements of any type, though they are expected to be event objects.

```typescript
  let batchTimer: NodeJS.Timeout | null = null
```
Declares a mutable variable `batchTimer` which will hold the ID of a `setTimeout` timer. This timer is used to trigger `flushBatch` after `BATCH_INTERVAL_MS`. It's initialized to `null` because no timer is active initially. `NodeJS.Timeout` is often used for `setTimeout` return types in environments like Node.js, and sometimes makes its way into client-side TS configs, representing a `number` in browsers.

```typescript
  try {
```
Starts a `try` block to gracefully handle potential errors, especially when interacting with `localStorage`, which can sometimes fail (e.g., if storage limits are exceeded or security settings prevent access).

```typescript
    if (env.NEXT_TELEMETRY_DISABLED === '1') {
```
Checks if an environment variable `NEXT_TELEMETRY_DISABLED` is set to the string `'1'`. This provides a way to globally disable telemetry, perhaps during development or for specific deployments.

```typescript
      telemetryEnabled = false
```
If the environment variable is `'1'`, telemetry is explicitly disabled.

```typescript
    } else {
```
If the environment variable is *not* `'1'`, proceed to check user preferences.

```typescript
      const storedPreference = localStorage.getItem(TELEMETRY_STATUS_KEY)
```
Retrieves the user's saved telemetry preference from `localStorage` using the `TELEMETRY_STATUS_KEY`. This could be a setting the user made in the application.

```typescript
      if (storedPreference) {
```
Checks if a preference was actually found in `localStorage`.

```typescript
        const status = JSON.parse(storedPreference)
```
Parses the `storedPreference` string as a JSON object. It's assumed that the preference is stored as a JSON string.

```typescript
        telemetryEnabled = status.enabled
```
Updates `telemetryEnabled` based on the `enabled` property of the parsed status object. This allows users to opt-in or opt-out via an application setting.

```typescript
      }
    }
  } catch (_e) {
```
Catches any errors that occur within the `try` block (e.g., `localStorage` not available, `JSON.parse` failing). The `_e` indicates the error object is not used.

```typescript
    telemetryEnabled = false
```
If an error occurs during the preference retrieval/parsing, telemetry is disabled as a safe default to avoid unexpected behavior or data collection without explicit consent.

```typescript
  }
```
Ends the `try...catch` block.

```typescript
  /**
   * Add event to batch and schedule flush
   */
  function addToBatch(event: any): void {
```
Defines a function `addToBatch` that takes an `event` (of any type) and adds it to the `eventBatch`.

```typescript
    if (!telemetryEnabled) return
```
If `telemetryEnabled` is `false`, the function immediately exits, ignoring the event.

```typescript
    eventBatch.push(event)
```
Adds the new `event` to the `eventBatch` array.

```typescript
    if (eventBatch.length >= MAX_BATCH_SIZE) {
```
Checks if the number of events in the batch has reached or exceeded the `MAX_BATCH_SIZE`.

```typescript
      flushBatch()
```
If the batch size limit is met, immediately call `flushBatch` to send the events.

```typescript
    } else if (!batchTimer) {
```
If the batch size limit is *not* met, check if there's no active `batchTimer`. This means no timer has been set yet for the current batch.

```typescript
      batchTimer = setTimeout(flushBatch, BATCH_INTERVAL_MS)
```
If no timer is active, set a new timer to call `flushBatch` after `BATCH_INTERVAL_MS` (10 seconds). The timer's ID is stored in `batchTimer`.

```typescript
    }
  }
```
Ends the `addToBatch` function.

```typescript
  /**
   * Sanitize event data to remove sensitive information
   */
  function sanitizeEvent(event: any): any {
```
Defines a function `sanitizeEvent` that recursively processes an event object to remove or redact sensitive information. It takes an `event` (of any type) and returns a sanitized version.

```typescript
    const patterns = ['password', 'token', 'secret', 'key', 'auth', 'credential', 'private']
```
Defines an array of strings representing keywords that, if found in keys or values, indicate sensitive data.

```typescript
    const sensitiveRe = new RegExp(patterns.join('|'), 'i')
```
Creates a regular expression from the `patterns` array. `patterns.join('|')` creates a string like "password|token|...", and `'i'` makes the regex case-insensitive. This regex will match any of the sensitive keywords.

```typescript
    const scrubString = (s: string) => (s && sensitiveRe.test(s) ? '[redacted]' : s)
```
Defines a helper arrow function `scrubString`. If the input string `s` is not null/empty and matches the `sensitiveRe` regex, it returns the string `'[redacted]'`; otherwise, it returns the original string `s`.

```typescript
    if (event == null) return event
```
If the `event` is `null` or `undefined`, return it as is.

```typescript
    if (typeof event === 'string') return scrubString(event)
```
If the `event` is a string, call `scrubString` on it and return the result.

```typescript
    if (typeof event !== 'object') return event
```
If the `event` is not a string, and not an object (e.g., number, boolean), return it as is. This handles primitive values that are not strings.

```typescript
    if (Array.isArray(event)) {
      return event.map((item) => sanitizeEvent(item))
    }
```
If the `event` is an array, recursively call `sanitizeEvent` on each item in the array and return a new array with the sanitized items.

```typescript
    const sanitized: Record<string, unknown> = {}
```
If the `event` is an object (but not an array), initialize an empty object `sanitized` to store the sanitized properties. `Record<string, unknown>` indicates it's an object with string keys and values of any type.

```typescript
    for (const [key, value] of Object.entries(event)) {
```
Iterates over each key-value pair of the original `event` object.

```typescript
      const lowerKey = key.toLowerCase()
```
Converts the current key to lowercase for case-insensitive matching.

```typescript
      if (patterns.some((p) => lowerKey.includes(p))) continue
```
Checks if any of the sensitive `patterns` are included in the `lowerKey`. If a sensitive keyword is found in the key itself, the loop `continue`s, skipping this key-value pair entirely (removing the sensitive key and its value).

```typescript
      if (typeof value === 'string') sanitized[key] = scrubString(value)
```
If the `value` is a string, sanitize it using `scrubString` and add it to the `sanitized` object under its original `key`.

```typescript
      else if (Array.isArray(value)) sanitized[key] = value.map((v) => sanitizeEvent(v))
```
If the `value` is an array, recursively call `sanitizeEvent` on each item in the array and store the resulting array.

```typescript
      else if (value && typeof value === 'object') sanitized[key] = sanitizeEvent(value)
```
If the `value` is a non-null object (and not an array), recursively call `sanitizeEvent` on this nested object.

```typescript
      else sanitized[key] = value
```
For any other type of value (numbers, booleans, `null`/`undefined` that aren't top-level), add it to `sanitized` as is.

```typescript
    }
    return sanitized
  }
```
Ends the loop and returns the `sanitized` object.

```typescript
  /**
   * Flush batch of events to server
   */
  function flushBatch(): void {
```
Defines the `flushBatch` function responsible for sending the collected events.

```typescript
    if (eventBatch.length === 0) return
```
If `eventBatch` is empty, there's nothing to send, so exit the function.

```typescript
    const batch = eventBatch.splice(0, eventBatch.length)
```
Creates a new array `batch` containing all events from `eventBatch` and simultaneously empties `eventBatch`. `splice(0, eventBatch.length)` removes all elements starting from index 0.

```typescript
    if (batchTimer) {
      clearTimeout(batchTimer)
      batchTimer = null
    }
```
If a `batchTimer` is active, clear it (stop the pending timeout) and reset `batchTimer` to `null`. This prevents the timer from firing after the batch has already been sent.

```typescript
    const sanitizedBatch = batch.map(sanitizeEvent)
```
Applies the `sanitizeEvent` function to each event in the `batch` to create a `sanitizedBatch`.

```typescript
    const payload = JSON.stringify({
      category: 'batch',
      action: 'client_events',
      events: sanitizedBatch,
      timestamp: Date.now(),
    })
```
Constructs the final payload object to be sent. It's a JSON string containing a generic `category` and `action` for the batch, the `sanitizedBatch` of events, and a `timestamp` of when the batch was prepared.

```typescript
    const payloadSize = new Blob([payload]).size
```
Calculates the size of the `payload` string in bytes. This is done by creating a `Blob` (Binary Large Object) from the string and then accessing its `size` property. This is necessary because `sendBeacon` has size limitations.

```typescript
    const MAX_BEACON_SIZE = 64 * 1024 // 64KB
```
Defines a constant for the maximum recommended size for a `sendBeacon` payload, typically 64KB.

```typescript
    if (navigator.sendBeacon && payloadSize < MAX_BEACON_SIZE) {
```
Checks if the browser supports `navigator.sendBeacon` and if the `payloadSize` is within the recommended limit. `sendBeacon` is ideal for sending data when a page is unloading because it's non-blocking and the browser guarantees the request will be sent.

```typescript
      const sent = navigator.sendBeacon('/api/telemetry', payload)
```
Attempts to send the `payload` to the `/api/telemetry` endpoint using `sendBeacon`. `sendBeacon` returns `true` if the browser successfully queued the request, `false` otherwise.

```typescript
      if (!sent) {
```
If `sendBeacon` returns `false` (meaning it failed to queue the request, perhaps due to internal browser limits or network issues, though less common than size limits), then fall back to `fetch`.

```typescript
        fetch('/api/telemetry', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: payload,
          keepalive: true,
        }).catch(() => {
          // Silently fail
        })
```
If `sendBeacon` fails or isn't supported, send the payload using a standard `fetch` POST request.
*   `method: 'POST'` specifies it's a POST request.
*   `headers` indicates the content type is JSON.
*   `body` is the JSON `payload`.
*   `keepalive: true` is crucial here. It hints to the browser that this `fetch` request should outlive the page, making it more reliable for sending data during page unload, similar to `sendBeacon`.
*   `.catch(() => { /* Silently fail */ })` handles any network errors silently, as telemetry should not interfere with the user experience.

```typescript
      }
    } else {
```
If `navigator.sendBeacon` is not supported or the payload is too large, directly use `fetch`.

```typescript
      fetch('/api/telemetry', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: payload,
        keepalive: true,
      }).catch(() => {
        // Silently fail
      })
    }
  }
```
Ends the `flushBatch` function.

```typescript
  window.addEventListener('beforeunload', flushBatch)
```
Registers an event listener that calls `flushBatch` just before the user navigates away from the page or closes the tab. This ensures any pending events are sent.

```typescript
  window.addEventListener('visibilitychange', () => {
```
Registers an event listener for `visibilitychange`, which fires when the visibility state of the document changes (e.g., the user switches tabs, minimizes the browser).

```typescript
    if (document.visibilityState === 'hidden') {
      flushBatch()
    }
  })
```
If the document's visibility state becomes `'hidden'`, it means the page is no longer active in the foreground, so `flushBatch` is called to send any pending events.

```typescript
  /**
   * Global event tracking function
   */

  ;(window as any).__SIM_TELEMETRY_ENABLED = telemetryEnabled
```
Exposes the `telemetryEnabled` status globally on the `window` object under the key `__SIM_TELEMETRY_ENABLED`. The `(window as any)` cast is used to tell TypeScript that we are intentionally adding a custom property to the global `window` object, which it doesn't know about by default.

```typescript
  ;(window as any).__SIM_TRACK_EVENT = (eventName: string, properties?: any) => {
```
Exposes a global function `__SIM_TRACK_EVENT` on the `window` object. This function will be the primary API for other parts of the application to track custom events. It takes an `eventName` (string) and optional `properties` (any type).

```typescript
    if (!telemetryEnabled) return
```
If telemetry is disabled, this tracking function does nothing.

```typescript
    addToBatch({
      category: 'feature_usage',
      action: eventName,
      timestamp: Date.now(),
      ...(properties || {}),
    })
  }
```
Constructs an event object with `category: 'feature_usage'`, the provided `action` (which is the `eventName`), a current `timestamp`, and then spreads any additional `properties` into the object. This constructed event is then passed to `addToBatch`.

```typescript
  if (telemetryEnabled) {
```
This entire block of code will only execute if telemetry is currently enabled.

```typescript
    const shouldTrackVitals = Math.random() < 0.1
```
Calculates a random boolean `shouldTrackVitals`. It will be `true` approximately 10% of the time. This is a common technique for sampling performance data to reduce the amount of data collected and server load.

```typescript
    if (shouldTrackVitals) {
```
If `shouldTrackVitals` is `true`, proceed with setting up Web Vitals tracking.

```typescript
      window.addEventListener(
        'load',
        () => {
          if (typeof PerformanceObserver !== 'undefined') {
```
Sets up an event listener for the `load` event (when the entire page has loaded) to start observing performance metrics. It also checks if `PerformanceObserver` API is supported by the browser.

```typescript
            const lcpObserver = new PerformanceObserver((list) => {
              const entries = list.getEntries()
              const lastEntry = entries[entries.length - 1]

              if (lastEntry) {
                addToBatch({
                  category: 'performance',
                  action: 'web_vital',
                  label: 'LCP',
                  value: (lastEntry as any).startTime || 0,
                  entryType: 'largest-contentful-paint',
                  timestamp: Date.now(),
                })
              }

              lcpObserver.disconnect()
            })
```
**Largest Contentful Paint (LCP) Observer**:
*   Creates a `PerformanceObserver` specifically for 'largest-contentful-paint' entries.
*   The callback function `(list)` receives a `PerformanceObserverEntryList`.
*   It gets all entries and takes the `lastEntry`, which typically represents the final LCP value.
*   If an `lastEntry` exists, an event is added to the batch with details about LCP, its `startTime` (the value), and type.
*   `lcpObserver.disconnect()`: LCP usually stabilizes, so once the value is reported, the observer can be disconnected to save resources.

```typescript
            let clsValue = 0
            const clsObserver = new PerformanceObserver((list) => {
              for (const entry of list.getEntries()) {
                if (!(entry as any).hadRecentInput) {
                  clsValue += (entry as any).value || 0
                }
              }
            })
```
**Cumulative Layout Shift (CLS) Observer**:
*   `clsValue` is initialized to 0 and will accumulate layout shifts.
*   A `PerformanceObserver` is created for 'layout-shift' entries.
*   The callback iterates through each `entry`.
*   `!(entry as any).hadRecentInput`: Only layout shifts that are *not* a direct result of user input are counted towards CLS (to avoid penalizing expected shifts).
*   `clsValue += (entry as any).value || 0`: The `value` of each relevant layout shift is added to `clsValue`. CLS accumulates over the entire page lifecycle.

```typescript
            const fidObserver = new PerformanceObserver((list) => {
              const entries = list.getEntries()

              for (const entry of entries) {
                const fidValue =
                  ((entry as any).processingStart || 0) - ((entry as any).startTime || 0)

                addToBatch({
                  category: 'performance',
                  action: 'web_vital',
                  label: 'FID',
                  value: fidValue,
                  entryType: 'first-input',
                  timestamp: Date.now(),
                })
              }

              fidObserver.disconnect()
            })
```
**First Input Delay (FID) Observer**:
*   A `PerformanceObserver` is created for 'first-input' entries.
*   The callback iterates through the entries (typically there's only one relevant FID entry).
*   `fidValue = (entry as any).processingStart - (entry as any).startTime`: Calculates FID by subtracting the input event's `startTime` from when its processing actually began.
*   An event is added to the batch with details about FID.
*   `fidObserver.disconnect()`: FID is generally reported once for the first input, so the observer is disconnected afterwards.

```typescript
            window.addEventListener('beforeunload', () => {
              if (clsValue > 0) {
                addToBatch({
                  category: 'performance',
                  action: 'web_vital',
                  label: 'CLS',
                  value: clsValue,
                  entryType: 'layout-shift',
                  timestamp: Date.now(),
                })
              }
              clsObserver.disconnect()
            })
```
When the `beforeunload` event fires, the accumulated `clsValue` is reported as a Web Vital event *if* it's greater than 0. The `clsObserver` is then disconnected.

```typescript
            lcpObserver.observe({ type: 'largest-contentful-paint', buffered: true })
            clsObserver.observe({ type: 'layout-shift', buffered: true })
            fidObserver.observe({ type: 'first-input', buffered: true })
```
Starts observing the respective performance entry types.
*   `type`: Specifies which type of performance entries to observe.
*   `buffered: true`: Tells the observer to include any entries that were generated *before* the observer was created (useful if the observer is initialized after some performance events have already occurred).

```typescript
          }
        },
        { once: true }
      )
    }
```
The `window.addEventListener('load', ...)` is configured with `{ once: true }`, meaning the callback function will execute only once when the page loads, and then the listener will automatically be removed.

```typescript
    window.addEventListener('error', (event) => {
```
Registers an event listener for global `error` events, which capture unhandled JavaScript errors that occur on the page.

```typescript
      if (telemetryEnabled && !event.defaultPrevented) {
```
Checks if telemetry is enabled and if the default action for the error event (`event.defaultPrevented`) has *not* been prevented by another script. This ensures we don't report errors that are intentionally handled elsewhere.

```typescript
        addToBatch({
          category: 'error',
          action: 'unhandled_error',
          message: event.error?.message || event.message || 'Unknown error',
          url: window.location.pathname,
          timestamp: Date.now(),
        })
      }
    })
```
If an unhandled error occurs, an event is added to the batch with:
*   `category: 'error'`
*   `action: 'unhandled_error'`
*   `message`: The error message, preferring `event.error.message`, then `event.message`, defaulting to 'Unknown error'.
*   `url`: The current page's path.
*   `timestamp`: The current time.

```typescript
    window.addEventListener('unhandledrejection', (event) => {
```
Registers an event listener for `unhandledrejection` events, which occur when a JavaScript Promise is rejected but no `.catch()` handler is provided.

```typescript
      if (telemetryEnabled) {
        addToBatch({
          category: 'error',
          action: 'unhandled_rejection',
          message: event.reason?.message || String(event.reason) || 'Unhandled promise rejection',
          url: window.location.pathname,
          timestamp: Date.now(),
        })
      }
    })
  }
}
```
If an unhandled promise rejection occurs, an event is added to the batch with:
*   `category: 'error'`
*   `action: 'unhandled_rejection'`
*   `message`: The reason for the rejection, preferring `event.reason.message`, then converting `event.reason` to a string, defaulting to 'Unhandled promise rejection'.
*   `url`: The current page's path.
*   `timestamp`: The current time.
This completes the `if (typeof window !== 'undefined')` block.