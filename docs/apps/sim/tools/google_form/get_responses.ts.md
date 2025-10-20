This TypeScript file defines a "tool" configuration for interacting with the Google Forms API. Specifically, it provides a way to retrieve responses from a Google Form, either a single specific response or a list of responses.

This tool is designed to be pluggable into a larger system that manages various integrations or "tools." It encapsulates all the necessary metadata, parameters, request logic, and response transformation for getting Google Forms responses.

Let's break down each part of the code:

---

### **Purpose of this File**

The primary purpose of `getResponsesTool.ts` is to describe and configure a specific operation: **fetching responses from Google Forms**. It serves as a blueprint for an integration or automation system, detailing:

1.  **What it does:** Retrieves either a single response or a list of responses from a Google Form.
2.  **What it's called:** "Google Forms: Get Responses"
3.  **What it needs:** OAuth2 authentication (`accessToken`), a `formId`, and optionally a `responseId` (for a single response) or `pageSize` (for listing responses).
4.  **How to call the Google Forms API:** It dynamically constructs the correct API endpoint URL and sets the necessary authentication headers.
5.  **How to process the API's response:** It handles both successful and error responses, normalizes the often complex Google Forms API data structure into a flatter, more usable format, and distinguishes between single response and list response payloads.

In essence, it's a self-contained definition for a Google Forms "Get Responses" capability within a broader platform.

---

### **Detailed Explanation**

```typescript
import type {
  GoogleFormsGetResponsesParams,
  GoogleFormsResponse,
  GoogleFormsResponseList,
} from '@/tools/google_form/types'
import { buildGetResponseUrl, buildListResponsesUrl } from '@/tools/google_form/utils'
import type { ToolConfig } from '@/tools/types'
```

*   **`import type { ... } from '@/tools/google_form/types'`**: This line imports TypeScript type definitions.
    *   `GoogleFormsGetResponsesParams`: Defines the structure of the input parameters this tool expects.
    *   `GoogleFormsResponse`: Represents the structure of a single response object from the Google Forms API.
    *   `GoogleFormsResponseList`: Represents the structure of an API response that contains a list of Google Forms responses.
    *   These types are crucial for ensuring type safety throughout the tool's configuration.
*   **`import { buildGetResponseUrl, buildListResponsesUrl } from '@/tools/google_form/utils'`**: This line imports utility functions.
    *   `buildGetResponseUrl`: A helper function responsible for constructing the API endpoint URL for fetching a *single* Google Form response.
    *   `buildListResponsesUrl`: A helper function for constructing the API endpoint URL for fetching a *list* of Google Form responses.
*   **`import type { ToolConfig } from '@/tools/types'`**: This imports the core `ToolConfig` type, which defines the expected structure for any tool within the larger system. This file's `getResponsesTool` object must conform to this type.

---

```typescript
export const getResponsesTool: ToolConfig<GoogleFormsGetResponsesParams> = {
  id: 'google_forms_get_responses',
  name: 'Google Forms: Get Responses',
  description: 'Retrieve a single response or list responses from a Google Form',
  version: '1.0.0',
```

*   **`export const getResponsesTool: ToolConfig<GoogleFormsGetResponsesParams> = { ... }`**: This declares and exports a constant variable named `getResponsesTool`. It's explicitly typed as `ToolConfig`, specialized for `GoogleFormsGetResponsesParams`, meaning it's a tool configuration object that expects Google Forms response parameters.
*   **`id: 'google_forms_get_responses'`**: A unique identifier for this specific tool.
*   **`name: 'Google Forms: Get Responses'`**: A human-readable name for the tool, typically displayed in a user interface.
*   **`description: 'Retrieve a single response or list responses from a Google Form'`**: A brief explanation of what the tool does.
*   **`version: '1.0.0'`**: The version of this tool definition.

---

```typescript
  oauth: {
    required: true,
    provider: 'google-forms',
    additionalScopes: [],
  },
```

*   **`oauth: { ... }`**: This section specifies the OAuth2 authentication requirements for this tool.
*   **`required: true`**: Indicates that authentication is mandatory to use this tool.
*   **`provider: 'google-forms'`**: Specifies the OAuth provider that should be used (e.g., Google's OAuth endpoint configured specifically for Google Forms access).
*   **`additionalScopes: []`**: An empty array, meaning no extra, non-standard OAuth scopes are required beyond what the `'google-forms'` provider inherently handles (e.g., `https://www.googleapis.com/auth/forms.responses.readonly`).

---

```typescript
  params: {
    accessToken: {
      type: 'string',
      required: true,
      visibility: 'hidden',
      description: 'OAuth2 access token',
    },
    formId: {
      type: 'string',
      required: true,
      visibility: 'user-only',
      description: 'The ID of the Google Form',
    },
    responseId: {
      type: 'string',
      required: false,
      visibility: 'user-only',
      description: 'If provided, returns this specific response',
    },
    pageSize: {
      type: 'number',
      required: false,
      visibility: 'user-only',
      description:
        'Maximum number of responses to return (service may return fewer). Defaults to 5000.',
    },
  },
```

*   **`params: { ... }`**: This defines the input parameters that the user or calling system needs to provide to the tool. Each parameter has a specific schema.
    *   **`accessToken`**:
        *   `type: 'string'`: It's expected to be a string.
        *   `required: true`: This parameter *must* be provided.
        *   `visibility: 'hidden'`: This parameter should not be exposed directly to end-users in a UI; it's typically managed internally by the platform.
        *   `description: 'OAuth2 access token'`: Explains its purpose.
    *   **`formId`**:
        *   `type: 'string'`: Expected to be a string.
        *   `required: true`: This parameter *must* be provided.
        *   `visibility: 'user-only'`: This parameter should be visible and configurable by the end-user.
        *   `description: 'The ID of the Google Form'`: Explains its purpose.
    *   **`responseId`**:
        *   `type: 'string'`: Expected to be a string.
        *   `required: false`: This parameter is optional. If provided, the tool will fetch a single specific response.
        *   `visibility: 'user-only'`: Visible to the user.
        *   `description: 'If provided, returns this specific response'`: Explains its conditional use.
    *   **`pageSize`**:
        *   `type: 'number'`: Expected to be a number.
        *   `required: false`: This parameter is optional. If `responseId` is *not* provided, this controls the number of responses returned when listing.
        *   `visibility: 'user-only'`: Visible to the user.
        *   `description: 'Maximum number of responses to return (service may return fewer). Defaults to 5000.'`: Explains its purpose and default behavior.

---

```typescript
  request: {
    url: (params: GoogleFormsGetResponsesParams) =>
      params.responseId
        ? buildGetResponseUrl({ formId: params.formId, responseId: params.responseId })
        : buildListResponsesUrl({ formId: params.formId, pageSize: params.pageSize }),
    method: 'GET',
    headers: (params: GoogleFormsGetResponsesParams) => ({
      Authorization: `Bearer ${params.accessToken}`,
      'Content-Type': 'application/json',
    }),
  },
```

*   **`request: { ... }`**: This section defines how the HTTP request to the Google Forms API should be constructed.
    *   **`url: (params: GoogleFormsGetResponsesParams) => ...`**: This is a function that dynamically generates the API endpoint URL based on the provided `params`.
        *   **`params.responseId ? ... : ...`**: This is a ternary operator, checking if `responseId` is present in the parameters.
            *   If `params.responseId` exists (truthy), it means we want a single response. It calls `buildGetResponseUrl` with the `formId` and `responseId`.
            *   If `params.responseId` does *not* exist (falsy), it means we want a list of responses. It calls `buildListResponsesUrl` with the `formId` and `pageSize`.
    *   **`method: 'GET'`**: Specifies that the HTTP request method should be `GET` (for retrieving data).
    *   **`headers: (params: GoogleFormsGetResponsesParams) => ({ ... })`**: This is a function that dynamically generates the HTTP headers for the request.
        *   **`Authorization: `Bearer ${params.accessToken}``**: Sets the `Authorization` header with the OAuth2 access token, using the standard Bearer token scheme. This authenticates the request to Google Forms.
        *   **`'Content-Type': 'application/json'`**: Specifies that the client is sending JSON data (though for a GET request, this is often less critical than for POST/PUT, it's good practice for API interaction).

---

```typescript
  transformResponse: async (response: Response, params?: GoogleFormsGetResponsesParams) => {
    const data = (await response.json()) as unknown

    if (!response.ok) {
      let errorMessage = response.statusText || 'Failed to fetch responses'
      if (data && typeof data === 'object') {
        const record = data as Record<string, unknown>
        const error = record.error as { message?: string } | undefined
        if (error?.message) {
          errorMessage = error.message
        }
      }

      return {
        success: false,
        output: (data as Record<string, unknown>) || {},
        error: errorMessage,
      }
    }
```

*   **`transformResponse: async (response: Response, params?: GoogleFormsGetResponsesParams) => { ... }`**: This is an `async` function responsible for processing the raw HTTP response from the Google Forms API. It takes the `Response` object and the original `params` (which are optional here but useful for context).
*   **`const data = (await response.json()) as unknown`**:
    *   `await response.json()`: Parses the raw HTTP response body as JSON. Since `response.json()` returns a Promise, `await` is used to wait for the parsing to complete.
    *   `as unknown`: The parsed JSON data is initially typed as `unknown` for maximum type safety. This means TypeScript doesn't assume its shape, forcing explicit type narrowing later.
*   **`if (!response.ok)`**: Checks if the HTTP response status code indicates success (i.e., status code 200-299). If `response.ok` is `false`, it means an HTTP error occurred (e.g., 400 Bad Request, 401 Unauthorized, 404 Not Found, 500 Server Error).
    *   **`let errorMessage = response.statusText || 'Failed to fetch responses'`**: Initializes `errorMessage` with the HTTP status text (e.g., "Not Found") or a generic error message if `statusText` is empty.
    *   **`if (data && typeof data === 'object') { ... }`**: If the error response itself contains a JSON body (which Google APIs often do for errors), this block attempts to extract a more specific error message.
        *   **`const record = data as Record<string, unknown>`**: Casts the `unknown` `data` to a generic object `Record<string, unknown>` to access its properties.
        *   **`const error = record.error as { message?: string } | undefined`**: Tries to access a nested `error` object, expecting it might have a `message` property.
        *   **`if (error?.message) { errorMessage = error.message }`**: If a specific error message is found within `data.error.message`, it updates `errorMessage` with this more detailed information.
    *   **`return { success: false, output: (data as Record<string, unknown>) || {}, error: errorMessage, }`**: If an error occurred, this returns a standardized error object:
        *   `success: false`: Indicates the operation failed.
        *   `output`: Includes the raw error data (if any) for debugging.
        *   `error`: The extracted error message.

---

```typescript
    // Normalize answers into a flat key/value map per response
    const normalizeAnswerContainer = (container: unknown): unknown => {
      if (!container || typeof container !== 'object') return container
      const record = container as Record<string, unknown>
      const answers = record.answers as unknown[] | undefined
      if (Array.isArray(answers)) {
        const values = answers.map((entry) => {
          if (entry && typeof entry === 'object') {
            const er = entry as Record<string, unknown>
            if (typeof er.value !== 'undefined') return er.value
          }
          return entry
        })
        return values.length === 1 ? values[0] : values
      }
      return container
    }
```

*   **`const normalizeAnswerContainer = (container: unknown): unknown => { ... }`**: This helper function is designed to simplify the structure of a specific "answer container" object from Google Forms. Google Forms often nests the actual answer value under properties like `textAnswers` or `multipleChoiceAnswers`, which then contain an `answers` array, where each entry might have a `value` property.
    *   **`if (!container || typeof container !== 'object') return container`**: Basic validation: if the input is not an object or is null/undefined, return it as-is.
    *   **`const record = container as Record<string, unknown>`**: Casts the container to a generic object to access its properties.
    *   **`const answers = record.answers as unknown[] | undefined`**: Tries to extract an `answers` array from the container.
    *   **`if (Array.isArray(answers))`**: If an `answers` array is found:
        *   **`const values = answers.map((entry) => { ... })`**: It maps over each `entry` in the `answers` array.
            *   **`if (entry && typeof entry === 'object') { ... }`**: If an entry is an object, it attempts to extract its `value` property.
            *   **`const er = entry as Record<string, unknown>; if (typeof er.value !== 'undefined') return er.value`**: If `entry.value` exists, it returns that specific value.
            *   **`return entry`**: Otherwise, it returns the entire `entry`.
        *   **`return values.length === 1 ? values[0] : values`**: If only one value was extracted, it returns that value directly (e.g., a string or number). If multiple values were extracted (e.g., for a multi-select question), it returns an array of values.
    *   **`return container`**: If no `answers` array was found, the original `container` is returned.
*   **Simplified Logic:** This function essentially "flattens" the nested structure of individual answers, trying to pull out the direct value(s) provided by the user, making it easier to consume.

---

```typescript
    const normalizeAnswers = (answers: unknown): Record<string, unknown> => {
      if (!answers || typeof answers !== 'object') return {}
      const src = answers as Record<string, unknown>
      const out: Record<string, unknown> = {}
      for (const [questionId, answerObj] of Object.entries(src)) {
        if (answerObj && typeof answerObj === 'object') {
          const aRec = answerObj as Record<string, unknown>
          // Find first *Answers property that contains an answers array
          const key = Object.keys(aRec).find(
            (k) => k.toLowerCase().endsWith('answers') && Array.isArray((aRec[k] as any)?.answers)
          )
          if (key) {
            out[questionId] = normalizeAnswerContainer(aRec[key])
            continue
          }
        }
        out[questionId] = answerObj as unknown
      }
      return out
    }
```

*   **`const normalizeAnswers = (answers: unknown): Record<string, unknown> => { ... }`**: This helper function takes the `answers` object for an entire response (which is keyed by Google's internal `questionId`) and applies `normalizeAnswerContainer` to each specific answer.
    *   **`if (!answers || typeof answers !== 'object') return {}`**: Basic validation: if `answers` is not a valid object, return an empty object.
    *   **`const src = answers as Record<string, unknown>`**: Casts the `answers` input to a generic object.
    *   **`const out: Record<string, unknown> = {}`**: Initializes an empty object to store the normalized answers.
    *   **`for (const [questionId, answerObj] of Object.entries(src))`**: Iterates through each `questionId` and its corresponding `answerObj` in the source `answers` object.
        *   **`if (answerObj && typeof answerObj === 'object') { ... }`**: Checks if the `answerObj` for a specific question is a valid object.
            *   **`const aRec = answerObj as Record<string, unknown>`**: Casts `answerObj` to a generic object.
            *   **`const key = Object.keys(aRec).find((k) => k.toLowerCase().endsWith('answers') && Array.isArray((aRec[k] as any)?.answers))`**: This is a crucial line. It tries to find a property within `aRec` whose key ends with "answers" (case-insensitive, e.g., `textAnswers`, `multipleChoiceAnswers`) AND whose value is an object that itself contains an `answers` array. This identifies the specific "answer container" within the question's raw data. `(aRec[k] as any)?.answers` uses `any` temporarily for flexibility in accessing potentially undefined nested properties.
            *   **`if (key) { out[questionId] = normalizeAnswerContainer(aRec[key]); continue; }`**: If such a key is found, it applies `normalizeAnswerContainer` to that specific nested container and stores the result in `out` under the `questionId`. `continue` moves to the next question.
        *   **`out[questionId] = answerObj as unknown`**: If no specific `*Answers` container was found, the original `answerObj` is stored as-is.
    *   **`return out`**: Returns the object containing all normalized answers, keyed by `questionId`.
*   **Simplified Logic:** This function iterates through all questions in a response and attempts to find the "deepest" part of the Google Forms answer structure where the actual user input resides. It then uses `normalizeAnswerContainer` to extract and flatten those inputs.

---

```typescript
    const normalizeResponse = (r: GoogleFormsResponse): Record<string, unknown> => ({
      responseId: r.responseId,
      createTime: r.createTime,
      lastSubmittedTime: r.lastSubmittedTime,
      answers: normalizeAnswers(r.answers as unknown),
    })
```

*   **`const normalizeResponse = (r: GoogleFormsResponse): Record<string, unknown> => ({ ... })`**: This helper function takes a single `GoogleFormsResponse` object and transforms it into a cleaner, more generalized `Record<string, unknown>` format.
    *   It extracts `responseId`, `createTime`, and `lastSubmittedTime` directly.
    *   **`answers: normalizeAnswers(r.answers as unknown)`**: Crucially, it processes the raw `r.answers` object using the `normalizeAnswers` function to get a flattened, readable set of answers.
*   **Simplified Logic:** This function provides a clean, consistent output structure for a single Google Form response, making its data easily accessible.

---

```typescript
    // Distinguish single vs list response shapes
    const isList = (obj: unknown): obj is GoogleFormsResponseList =>
      !!obj && typeof obj === 'object' && Array.isArray((obj as GoogleFormsResponseList).responses)

    if (isList(data)) {
      const listData = data as GoogleFormsResponseList
      const toTimestamp = (s?: string): number => {
        if (!s) return 0
        const t = Date.parse(s)
        return Number.isNaN(t) ? 0 : t
      }
      const sorted = (listData.responses || [])
        .slice()
        .sort(
          (a, b) =>
            toTimestamp(b.lastSubmittedTime || b.createTime) -
            toTimestamp(a.lastSubmittedTime || a.createTime)
        )
      const normalized = sorted.map((r) => normalizeResponse(r))
      return {
        success: true,
        output: {
          responses: normalized,
          raw: listData,
        } as unknown as Record<string, unknown>,
      }
    }
```

*   **`const isList = (obj: unknown): obj is GoogleFormsResponseList => ...`**: This is a TypeScript [type guard](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#type-guards). It checks if an `unknown` object `obj` conforms to the `GoogleFormsResponseList` type.
    *   `!!obj && typeof obj === 'object'`: Ensures `obj` is a non-null object.
    *   `Array.isArray((obj as GoogleFormsResponseList).responses)`: Checks if the object has a `responses` property that is an array. If this is true, TypeScript will then treat `obj` as `GoogleFormsResponseList` within the `if (isList(data))` block.
*   **`if (isList(data))`**: If the parsed `data` from the API is identified as a list of responses:
    *   **`const listData = data as GoogleFormsResponseList`**: Narrows the type of `data` to `GoogleFormsResponseList`.
    *   **`const toTimestamp = (s?: string): number => { ... }`**: A small helper function to safely convert a string (like a date/time string) to a Unix timestamp (milliseconds since epoch). Returns `0` if invalid or missing. This is used for sorting.
    *   **`const sorted = (listData.responses || []).slice().sort(...)`**:
        *   `(listData.responses || [])`: Ensures we have an array to work with, even if `responses` is `null` or `undefined`.
        *   `.slice()`: Creates a shallow copy of the array so that the original `listData.responses` is not modified by the sort operation.
        *   `.sort((a, b) => ...)`: Sorts the responses. The sorting logic uses `toTimestamp` to compare `lastSubmittedTime` (preferred) or `createTime` for each response, sorting in descending order (newest first).
    *   **`const normalized = sorted.map((r) => normalizeResponse(r))`**: Maps over the sorted list of raw responses, applying the `normalizeResponse` function to each one to get a list of cleaned-up response objects.
    *   **`return { success: true, output: { responses: normalized, raw: listData, } as unknown as Record<string, unknown>, }`**: Returns the final successful result for a list.
        *   `success: true`: Indicates successful operation.
        *   `output`: Contains an object with:
            *   `responses`: The array of normalized and sorted responses.
            *   `raw`: The original, raw `GoogleFormsResponseList` data, useful for debugging or if the caller needs access to un-normalized fields.
        *   `as unknown as Record<string, unknown>`: A double type assertion to tell TypeScript that the output object should conform to `Record<string, unknown>`, making it flexible for the receiving system.

---

```typescript
    const single = data as GoogleFormsResponse
    const normalizedSingle = normalizeResponse(single)

    return {
      success: true,
      output: {
        response: normalizedSingle,
        raw: single,
      } as unknown as Record<string, unknown>,
    }
  },
}
```

*   **`const single = data as GoogleFormsResponse`**: If the `if (isList(data))` condition was false, it means `data` is a single response. It's explicitly cast to `GoogleFormsResponse`.
*   **`const normalizedSingle = normalizeResponse(single)`**: The `normalizeResponse` function is called to clean up and structure this single response.
*   **`return { success: true, output: { response: normalizedSingle, raw: single, } as unknown as Record<string, unknown>, }`**: Returns the final successful result for a single response.
    *   `success: true`: Indicates successful operation.
    *   `output`: Contains an object with:
        *   `response`: The single normalized response.
        *   `raw`: The original, raw `GoogleFormsResponse` data.
    *   `as unknown as Record<string, unknown>`: Double type assertion for flexibility, similar to the list output.

---

### **Simplifying Complex Logic - The Normalization Process**

The most complex part of this code is within the `transformResponse` function, specifically the answer normalization. Here's a simplified explanation:

Google Forms API responses for answers are very nested and can vary based on the question type. For example, a "short answer" might look like:

```json
{
  "12345": {
    "textAnswers": {
      "answers": [
        {
          "value": "User's short answer"
        }
      ]
    }
  }
}
```

This tool aims to transform that into something much simpler, like:

```json
{
  "12345": "User's short answer"
}
```

The `normalizeAnswerContainer` and `normalizeAnswers` functions work together to achieve this:

1.  **`normalizeAnswerContainer` (Inner Simplification):** This function is responsible for taking a specific "answer container" (like `textAnswers` in the example above) and extracting the actual `value`. It handles cases where there might be one value (returns the value directly) or multiple values (returns an array of values).
2.  **`normalizeAnswers` (Outer Simplification):** This function iterates through all the questions in a response. For each question, it intelligently tries to find the correct "answer container" (e.g., identifies the `textAnswers` property for question `12345`). Once it finds it, it then calls `normalizeAnswerContainer` to extract the simple value(s) for that question.
3.  **`normalizeResponse` (Full Response Simplification):** This wraps up the entire process for a single response, combining the extracted `responseId`, `createTime`, `lastSubmittedTime` with the fully normalized `answers` from `normalizeAnswers`.

By breaking down the normalization into these three helper functions, the code tackles the complexity step-by-step, resulting in a much cleaner and more accessible output for the consumer of this tool.