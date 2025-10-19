This TypeScript file provides a set of utility functions primarily focused on interacting with and processing data from Atlassian Confluence APIs. Its main purpose is to help identify a specific Confluence instance and then clean and structure the content of Confluence pages for easier use.

Let's break down each function in detail.

---

### **1. `getConfluenceCloudId` Function**

This function's job is to figure out the unique "Cloud ID" for your Confluence instance. Why is this important? Because when you make API calls to Confluence Cloud, you often need this ID to specify which Confluence site you're talking to.

**Simplified Logic:**

1.  **Ask Atlassian:** It first talks to the main Atlassian API to get a list of all Atlassian products (like Confluence or Jira) that your `accessToken` has permission to access.
2.  **Find Your Confluence:** It then looks through this list to find a Confluence instance whose URL matches the `domain` you provided (e.g., `yourcompany.atlassian.net`).
3.  **Return the ID:** If it finds a specific match, it gives you that Confluence instance's unique ID.
4.  **Fallback:** If it can't find an exact match but still sees *any* Confluence instances, it just returns the ID of the first one it finds (as a reasonable default).
5.  **Error:** If it finds no accessible Confluence instances at all, it throws an error.

**Line-by-Line Explanation:**

```typescript
export async function getConfluenceCloudId(domain: string, accessToken: string): Promise<string> {
```

*   `export`: This keyword means the `getConfluenceCloudId` function can be used by other parts of your application (imported into other files).
*   `async`: This indicates that the function will perform asynchronous operations, primarily the `fetch` call, meaning it might take some time to complete and won't block the rest of your program while waiting.
*   `function getConfluenceCloudId(...)`: Defines the function named `getConfluenceCloudId`.
*   `(domain: string, accessToken: string)`: These are the inputs (parameters) the function expects:
    *   `domain`: A string representing the domain of your Confluence instance (e.g., `"mycompany.atlassian.net"`).
    *   `accessToken`: A string containing an OAuth access token, which grants permission to access Atlassian resources.
*   `: Promise<string>`: This specifies the function's return type. It will return a `Promise` that, when resolved (successful), will give you a `string` (the Cloud ID).

```typescript
  const response = await fetch('https://api.atlassian.com/oauth/token/accessible-resources', {
    method: 'GET',
    headers: {
      Authorization: `Bearer ${accessToken}`,
      Accept: 'application/json',
    },
  })
```

*   `const response = await fetch(...)`: This line makes an HTTP `GET` request to a specific Atlassian API endpoint.
    *   `await`: Pauses the function's execution until the `fetch` request completes and returns a `Response` object.
    *   `'https://api.atlassian.com/oauth/token/accessible-resources'`: This is the URL to the Atlassian API that lists all resources (like Jira sites, Confluence sites) an authenticated user has access to.
    *   `{ ... }`: This object provides configuration for the `fetch` request:
        *   `method: 'GET'`: Specifies that this is a GET request (asking for data).
        *   `headers: { ... }`: Defines the HTTP headers to send with the request:
            *   `Authorization: `Bearer ${accessToken}``: This is the critical authentication header. It includes the `accessToken` you provided, telling the Atlassian API who is making the request and that they are authorized. `Bearer` is a common type of token.
            *   `Accept: 'application/json'`: This header tells the server that the client (our function) prefers to receive the response data in JSON format.

```typescript
  const resources = await response.json()
```

*   `const resources = await response.json()`: After receiving the `response` from the `fetch` call, this line parses the body of that response as JSON.
    *   `await`: Pauses until the JSON parsing is complete.
    *   `response.json()`: A method of the `Response` object that reads the response stream to completion and parses it as JSON. The result (`resources`) is expected to be an array of objects, where each object represents an accessible Atlassian resource.

```typescript
  if (Array.isArray(resources) && resources.length > 0) {
```

*   `if (...)`: This `if` statement checks two conditions to ensure we have valid resources to process:
    *   `Array.isArray(resources)`: Checks if the `resources` variable is actually an array.
    *   `resources.length > 0`: Checks if the array contains at least one item.

```typescript
    const normalizedInput = `https://${domain}`.toLowerCase()
```

*   `const normalizedInput = ...`: Creates a standardized URL string from the `domain` provided.
    *   ``https://${domain}``: Prepends `"https://"` to the `domain`.
    *   `.toLowerCase()`: Converts the entire URL to lowercase. This helps ensure that the comparison with resource URLs from the API is case-insensitive and robust.

```typescript
    const matchedResource = resources.find((r) => r.url.toLowerCase() === normalizedInput)
```

*   `const matchedResource = ...`: This line searches for a specific resource within the `resources` array.
    *   `resources.find(...)`: The `find` array method iterates through each item (`r`) in the `resources` array.
    *   `(r) => r.url.toLowerCase() === normalizedInput`: This is the condition that `find` uses. For each `r` (resource object), it takes its `url` property, converts it to lowercase, and checks if it exactly matches our `normalizedInput`. If a match is found, `find` returns that resource object; otherwise, it returns `undefined`.

```typescript
    if (matchedResource) {
      return matchedResource.id
    }
```

*   `if (matchedResource)`: Checks if a specific resource was found by the `find` method (i.e., `matchedResource` is not `undefined`).
*   `return matchedResource.id`: If a match is found, the function immediately returns the `id` property of that resource. This `id` is the Confluence Cloud ID we're looking for.

```typescript
  } // Closes the first 'if (Array.isArray(resources) && resources.length > 0)'
```

```typescript
  if (Array.isArray(resources) && resources.length > 0) {
    return resources[0].id
  }
```

*   `if (...)`: This is a fallback. If the previous `if` block didn't find a *specific* domain match but we *still have* an array of resources with items in it, this block will execute.
*   `return resources[0].id`: It returns the `id` of the very first resource in the `resources` array. This is a pragmatic default in cases where a direct domain match wasn't found but there's only one (or a primary) Confluence instance associated with the token.

```typescript
  throw new Error('No Confluence resources found')
}
```

*   `throw new Error(...)`: If neither of the above `if` blocks resulted in a return (meaning `resources` was empty, not an array, or no match/fallback was found), this line throws an `Error`. This signals that the function couldn't determine a Confluence Cloud ID.

---

### **2. `decodeHtmlEntities` Function**

This helper function cleans up strings by converting common HTML entities (like `&amp;`, `&lt;`) back into their original characters (`&`, `<`). This is useful when you've received content that might have been escaped for HTML display.

**Simplified Logic:**

It repeatedly goes through the text, replacing common HTML entity codes (like `&nbsp;`, `&lt;`, `&quot;`, `&#39;`) with their actual characters. It then specifically handles `&amp;` (ampersand). It loops until no more entities can be replaced, ensuring even nested entities (like `&amp;lt;`) are fully decoded.

**Line-by-Line Explanation:**

```typescript
function decodeHtmlEntities(text: string): string {
```

*   `function decodeHtmlEntities(...)`: Defines the function.
*   `(text: string)`: It takes one input, `text`, which is a string that might contain HTML entities.
*   `: string`: It will return a string.

```typescript
  let decoded = text
  let previous: string
```

*   `let decoded = text`: Initializes a mutable variable `decoded` with the original input `text`. This variable will be modified throughout the decoding process.
*   `let previous: string`: Declares a variable `previous`. This will be used to store the state of `decoded` before each set of replacements, allowing us to detect if any changes were made.

```typescript
  do {
    previous = decoded
    decoded = decoded
      .replace(/&nbsp;/g, ' ')
      .replace(/&lt;/g, '<')
      .replace(/&gt;/g, '>')
      .replace(/&quot;/g, '"')
      .replace(/&#39;/g, "'")
    decoded = decoded.replace(/&amp;/g, '&')
  } while (decoded !== previous)
```

*   `do { ... } while (decoded !== previous)`: This is a `do-while` loop.
    *   The code inside the `do { ... }` block will execute *at least once*.
    *   The loop will *continue* to execute as long as the condition `decoded !== previous` is true (meaning a change occurred in the last iteration). This is crucial for handling nested entities like `&amp;lt;` (which becomes `&lt;` then `<`).
*   `previous = decoded`: At the start of each loop iteration, the current state of `decoded` is saved into `previous`.
*   `decoded = decoded .replace(/&nbsp;/g, ' ') ... .replace(/&#39;/g, "'")`: These lines perform a series of `replace` operations using regular expressions (`/pattern/g`).
    *   `.replace(/&nbsp;/g, ' ')`: Replaces all occurrences (`g` flag for global) of `&nbsp;` (non-breaking space) with a regular space.
    *   `.replace(/&lt;/g, '<')`: Replaces `&lt;` (less than) with `<`.
    *   `.replace(/&gt;/g, '>')`: Replaces `&gt;` (greater than) with `>`.
    *   `.replace(/&quot;/g, '"')`: Replaces `&quot;` (double quote) with `"`.
    *   `.replace(/&#39;/g, "'")`: Replaces `&#39;` (apostrophe/single quote) with `'`.
*   `decoded = decoded.replace(/&amp;/g, '&')`: This is a separate replacement for the `&amp;` entity. It's placed after the others because `&amp;` often precedes other entities (e.g., `&amp;lt;`). By doing it last in the inner replacements, an `&amp;lt;` will first become `&lt;` (in a previous pass), and then `&lt;` will become `<` in a subsequent pass, ensuring correct decoding.

```typescript
  return decoded
}
```

*   `return decoded`: Once the `do-while` loop finishes (because no more changes were made), the fully decoded string is returned.

---

### **3. `stripHtmlTags` Function**

This helper function removes all HTML tags from a string, leaving only the plain text content.

**Simplified Logic:**

It repeatedly scans the input text, finds anything that looks like an HTML tag (e.g., `<p>`, `<div>`, `<br/>`), and removes it. It also cleans up any leftover `<` or `>` characters that might not have been part of a complete tag. It loops until all possible tags and angle brackets are gone.

**Line-by-Line Explanation:**

```typescript
function stripHtmlTags(html: string): string {
```

*   `function stripHtmlTags(...)`: Defines the function.
*   `(html: string)`: It takes one input, `html`, which is a string that might contain HTML tags.
*   `: string`: It will return a string.

```typescript
  let text = html
  let previous: string
```

*   `let text = html`: Initializes a mutable variable `text` with the original input HTML string. This will be modified.
*   `let previous: string`: Declares `previous` to hold the state of `text` from the last iteration, used to check for changes.

```typescript
  do {
    previous = text
    text = text.replace(/<[^>]*>/g, '')
    text = text.replace(/[<>]/g, '')
  } while (text !== previous)
```

*   `do { ... } while (text !== previous)`: This `do-while` loop repeatedly performs tag removal until no more changes are made to the `text` string. This is important for robustness, especially with potentially malformed HTML or nested structures.
*   `previous = text`: At the start of each loop iteration, the current state of `text` is saved into `previous`.
*   `text = text.replace(/<[^>]*>/g, '')`: This is the primary HTML tag removal step.
    *   `/<[^>]*>/g`: This regular expression matches HTML tags:
        *   `<`: Matches a literal opening angle bracket.
        *   `[^>]*`: Matches any character *except* (indicated by `^`) a closing angle bracket (`>`), zero or more times (`*`). This captures the content *inside* the tag.
        *   `>`: Matches a literal closing angle bracket.
        *   `/g`: The global flag ensures *all* matching tags in the string are replaced.
    *   The matched tags are replaced with an empty string, effectively removing them.
*   `text = text.replace(/[<>]/g, '')`: This is a secondary cleanup step. After removing full tags, some stray `<` or `>` characters might remain (e.g., if the HTML was malformed, or if content literally contained `<` or `>` that wasn't part of a tag, or if a previous regex pass left them). This line removes any remaining lone angle brackets.

```typescript
  return text.trim()
}
```

*   `return text.trim()`: Once the loop finishes, the `trim()` method is called on the final `text` string to remove any leading or trailing whitespace (like leftover newlines or spaces from removed tags). The cleaned plain text is then returned.

---

### **4. `transformPageData` Function**

This function takes raw, possibly inconsistent, Confluence page data (likely from an API response) and transforms it into a clean, standardized, and easy-to-use format.

**Simplified Logic:**

1.  **Find Content:** It cleverly tries multiple common locations within the input `data` object to find the actual page content. If it can't find anything, it uses a default message.
2.  **Clean Content:** It then applies the `stripHtmlTags` and `decodeHtmlEntities` functions to thoroughly clean this content.
3.  **Normalize Whitespace:** Finally, it consolidates multiple spaces and newlines into single spaces and removes leading/trailing whitespace.
4.  **Structure Output:** It returns a structured object containing a success flag, a timestamp, the page ID, the cleaned content, and the page title.

**Line-by-Line Explanation:**

```typescript
export function transformPageData(data: any) {
```

*   `export`: Makes this function available externally.
*   `function transformPageData(...)`: Defines the function.
*   `(data: any)`: It takes one input, `data`, which is an object representing the raw Confluence page data. `any` is used here, indicating that the exact structure of `data` might vary, and the function will attempt to handle different formats.

```typescript
  const content =
    data.body?.view?.value ||
    data.body?.storage?.value ||
    data.body?.atlas_doc_format?.value ||
    data.content ||
    data.description ||
    `Content for page ${data.title || 'Unknown'}`
```

*   `const content = ...`: This is a key part that intelligently extracts the page's main content. It uses the logical OR operator (`||`) to try several possible paths within the `data` object. The first expression that evaluates to a "truthy" value (not `null`, `undefined`, `0`, `false`, or an empty string) will be assigned to `content`.
    *   `data.body?.view?.value`: Tries to get content from `data.body.view.value`. The `?.` (optional chaining) safely accesses nested properties; if any part of the path (`body` or `view`) is `null` or `undefined`, it simply returns `undefined` instead of throwing an error.
    *   `data.body?.storage?.value`: If the `view` path fails, it tries `data.body.storage.value`. Confluence stores content in different formats (view, storage, Atlas Document Format).
    *   `data.body?.atlas_doc_format?.value`: If `storage` fails, it tries `data.body.atlas_doc_format.value`.
    *   `data.content`: If none of the `body` paths yield content, it tries a top-level `data.content` property.
    *   `data.description`: If `data.content` is also not found, it tries `data.description`.
    *   ``Content for page ${data.title || 'Unknown'}``: This is the ultimate fallback. If all previous attempts fail, it constructs a default content string using the page's `title` (or "Unknown" if `data.title` is also missing).

```typescript
  let cleanContent = stripHtmlTags(content)
```

*   `let cleanContent = ...`: Calls the `stripHtmlTags` function (explained above) to remove all HTML tags from the extracted `content`. The result is assigned to `cleanContent`.

```typescript
  cleanContent = decodeHtmlEntities(cleanContent)
```

*   `cleanContent = ...`: Takes the `cleanContent` (now free of HTML tags) and passes it to the `decodeHtmlEntities` function (explained above) to convert any remaining HTML entities back to their original characters.

```typescript
  cleanContent = cleanContent.replace(/\s+/g, ' ').trim()
```

*   `cleanContent = ...`: This line performs a final cleanup on whitespace:
    *   `.replace(/\s+/g, ' ')`: Uses a regular expression to replace any sequence of one or more whitespace characters (`\s+`) with a single space (` `). This effectively collapses multiple spaces, tabs, newlines, etc., into single spaces.
    *   `.trim()`: Removes any leading or trailing whitespace from the resulting string.

```typescript
  return {
    success: true,
    output: {
      ts: new Date().toISOString(),
      pageId: data.id || '',
      content: cleanContent,
      title: data.title || '',
    },
  }
}
```

*   `return { ... }`: The function returns a structured object:
    *   `success: true`: A boolean flag indicating that the transformation was successfully completed.
    *   `output: { ... }`: A nested object containing the transformed page data:
        *   `ts: new Date().toISOString()`: A timestamp (`ts`) generated at the time of processing, formatted as an ISO 8601 string (e.g., `"2023-10-27T10:30:00.000Z"`).
        *   `pageId: data.id || ''`: Extracts the `id` from the original `data` object. If `data.id` is not found, it defaults to an empty string.
        *   `content: cleanContent`: The fully cleaned and processed page content.
        *   `title: data.title || ''`: Extracts the `title` from the original `data` object. If `data.title` is not found, it defaults to an empty string.

---

This file provides a robust set of tools for authenticating with Atlassian and then cleaning up the often-messy HTML content returned by APIs into a more usable plain text format.