This TypeScript file is a crucial part of an application that interacts with an external service, likely called "Firecrawl," which provides functionalities like scraping web pages, performing web searches, and initiating web crawls.

### Purpose of This File

The primary purpose of this file is to **define the data structures (interfaces and types) for making requests to and receiving responses from the Firecrawl service.**

Think of it as a detailed blueprint or a contract:
1.  **Input Parameters**: It specifies exactly what information you need to provide when asking Firecrawl to perform an action (e.g., what parameters are needed to scrape a URL).
2.  **Output Responses**: It outlines the precise structure of the data you can expect back from Firecrawl for each operation (scrape, search, crawl), including the content, metadata, and any status information.
3.  **Type Safety**: By defining these interfaces, TypeScript can rigorously check your code, ensuring that you're sending the correct data to Firecrawl and correctly handling the data it sends back. This prevents many common programming errors before they even happen.

In essence, this file acts as the "source of truth" for how your application communicates with the Firecrawl API, ensuring consistency and reliability.

---

### Important TypeScript Concepts Explained

Before diving into the line-by-line explanation, let's clarify some core TypeScript concepts used throughout this file:

*   **`interface`**: An `interface` in TypeScript is a way to define the *shape* or *contract* of an object. It specifies the names and types of properties an object must have. It doesn't contain any implementation logic; it's purely for type checking.
*   **`export`**: The `export` keyword makes a declaration (like an interface or type) available for use in other TypeScript files. Without `export`, it would only be accessible within this file.
*   **`import type`**: This statement is used to import *only type definitions* from another file. It's a performance optimization that tells the TypeScript compiler that the imported item is purely for type checking and won't exist at runtime, allowing it to be completely removed from the compiled JavaScript output.
*   **`?` (Optional Property)**: When you see a question mark after a property name (e.g., `propertyName?: Type`), it means that this property is *optional*. An object conforming to the interface *might* have this property, but it's not strictly required.
*   **`extends`**: In interfaces, `extends` allows an interface to inherit properties from another interface. The extending interface will have all the properties of the base interface, plus any new properties defined within itself. This promotes code reuse and creates a hierarchy.
*   **`Array<Type>` or `Type[]`**: Both notations are used to define an array where all elements are of a specific `Type`. For example, `string[]` means "an array of strings."
*   **`|` (Union Type)**: The pipe symbol creates a "union type." A variable of a union type can hold a value that is *any one* of the types specified. For example, `A | B` means "either type `A` or type `B`." This is useful when a function or API might return different types of responses depending on the situation.

---

### Detailed Line-by-Line Explanation

Let's go through the code line by line, explaining its purpose and any nested structures.

```typescript
import type { ToolResponse } from '@/tools/types'
```

*   **`import type { ToolResponse } from '@/tools/types'`**: This line imports the `ToolResponse` interface from a file located at `src/tools/types.ts` (the `@/` typically maps to `src/`). This `ToolResponse` likely defines common properties that all tool responses share, such as a status code, success/failure flag, or a generic message. Using `import type` ensures that only the type information is imported, making the compiled JavaScript smaller.

---

#### Request Parameter Interfaces

These interfaces define the structure of the data you send to the Firecrawl service to initiate an action.

```typescript
export interface ScrapeParams {
  apiKey: string
  url: string
  scrapeOptions?: {
    onlyMainContent?: boolean
    formats?: string[]
  }
}
```

*   **`export interface ScrapeParams`**: This defines an interface named `ScrapeParams` that describes the required parameters for a web scraping operation.
*   **`apiKey: string`**: A required property. This `string` holds your API key for authentication with the Firecrawl service.
*   **`url: string`**: A required property. This `string` is the web address (URL) of the page you want to scrape.
*   **`scrapeOptions?: { ... }`**: An *optional* property (`?`). If provided, it must be an object containing further customization for the scraping process.
    *   **`onlyMainContent?: boolean`**: An optional `boolean`. If set to `true`, the scraper will attempt to extract only the primary, article-like content of the page, ignoring navigation, footers, sidebars, etc.
    *   **`formats?: string[]`**: An optional array of `string`s. This likely specifies the desired output formats for the scraped content (e.g., `['markdown', 'html']`).

---

```typescript
export interface SearchParams {
  apiKey: string
  query: string
}
```

*   **`export interface SearchParams`**: This defines the interface for parameters needed to perform a web search.
*   **`apiKey: string`**: Required API key for authentication.
*   **`query: string`**: Required. This `string` holds the actual search query you want to perform (e.g., "latest TypeScript features").

---

```typescript
export interface FirecrawlCrawlParams {
  apiKey: string
  url: string
  limit?: number
  onlyMainContent?: boolean
}
```

*   **`export interface FirecrawlCrawlParams`**: This defines the interface for parameters needed to initiate a web crawling operation (exploring multiple linked pages).
*   **`apiKey: string`**: Required API key for authentication.
*   **`url: string`**: Required. The starting URL from which the web crawl should begin.
*   **`limit?: number`**: An *optional* `number`. If provided, it specifies the maximum number of pages to crawl during this operation.
*   **`onlyMainContent?: boolean`**: An *optional* `boolean`. Similar to `ScrapeParams`, if `true`, it suggests that only the main content should be extracted from each page found during the crawl.

---

#### Response Interfaces

These interfaces define the structure of the data you receive back from the Firecrawl service after an operation.

```typescript
export interface ScrapeResponse extends ToolResponse {
  output: {
    markdown: string
    html?: string
    metadata: {
      title: string
      description: string
      language: string
      keywords: string
      robots: string
      ogTitle: string
      ogDescription: string
      ogUrl: string
      ogImage: string
      ogLocaleAlternate: string[]
      ogSiteName: string
      sourceURL: string
      statusCode: number
    }
  }
}
```

*   **`export interface ScrapeResponse extends ToolResponse`**: This defines the interface for the response received after a successful scraping operation. It `extends ToolResponse`, meaning it inherits all properties from `ToolResponse` (e.g., a general `status` or `message` property) in addition to its own specific properties.
*   **`output: { ... }`**: A required property. This object contains the core scraped content and its associated metadata.
    *   **`markdown: string`**: The primary scraped content, formatted as a Markdown string. This is required.
    *   **`html?: string`**: An *optional* `string`. If provided, it contains the raw HTML content of the scraped page.
    *   **`metadata: { ... }`**: A required object containing various pieces of information about the scraped web page itself.
        *   **`title: string`**: The HTML `<title>` of the page.
        *   **`description: string`**: The content of the HTML `<meta name="description">` tag.
        *   **`language: string`**: The detected language of the page (e.g., "en", "fr").
        *   **`keywords: string`**: The content of the HTML `<meta name="keywords">` tag.
        *   **`robots: string`**: The content of the HTML `<meta name="robots">` tag, which instructs search engine crawlers.
        *   **`ogTitle: string`**: The Open Graph title (`og:title`), used for social media sharing.
        *   **`ogDescription: string`**: The Open Graph description (`og:description`).
        *   **`ogUrl: string`**: The Open Graph URL (`og:url`).
        *   **`ogImage: string`**: The Open Graph image URL (`og:image`).
        *   **`ogLocaleAlternate: string[]`**: An array of alternate Open Graph locales (`og:locale:alternate`).
        *   **`ogSiteName: string`**: The Open Graph site name (`og:site_name`).
        *   **`sourceURL: string`**: The exact URL that was scraped.
        *   **`statusCode: number`**: The HTTP status code received when fetching the `sourceURL` (e.g., 200 for success, 404 for not found).

---

```typescript
export interface SearchResponse extends ToolResponse {
  output: {
    data: Array<{
      title: string
      description: string
      url: string
      markdown?: string
      html?: string
      rawHtml?: string
      links?: string[]
      screenshot?: string
      metadata: {
        title: string
        description: string
        sourceURL: string
        statusCode: number
        error?: string
      }
    }>
    warning?: string
  }
}
```

*   **`export interface SearchResponse extends ToolResponse`**: This defines the interface for the response received after a web search operation. It also `extends ToolResponse`.
*   **`output: { ... }`**: A required property, containing the results of the search.
    *   **`data: Array<{ ... }> `**: A required array. Each element in this array is an object representing a single search result.
        *   **`title: string`**: The title of the search result.
        *   **`description: string`**: A short snippet or description of the search result.
        *   **`url: string`**: The URL of the search result.
        *   **`markdown?: string`**: An *optional* `string`. If provided, it's the scraped content of this particular search result's page in Markdown.
        *   **`html?: string`**: An *optional* `string`. If provided, it's the scraped content of this particular search result's page in HTML.
        *   **`rawHtml?: string`**: An *optional* `string`. This might be the raw, unprocessed HTML of the result page.
        *   **`links?: string[]`**: An *optional* array of `string`s, containing hyperlinks found on the result page.
        *   **`screenshot?: string`**: An *optional* `string`, likely a URL to a screenshot image of the search result page.
        *   **`metadata: { ... }`**: A required object containing specific metadata for *this individual search result*.
            *   **`title: string`**: Title from the page's metadata.
            *   **`description: string`**: Description from the page's metadata.
            *   **`sourceURL: string`**: The exact URL of this search result.
            *   **`statusCode: number`**: The HTTP status code received when fetching this `sourceURL`.
            *   **`error?: string`**: An *optional* `string`. If present, it indicates an error specific to fetching this particular search result.
    *   **`warning?: string`**: An *optional* `string`. If present, it contains a general warning message related to the overall search operation.

---

```typescript
export interface FirecrawlCrawlResponse extends ToolResponse {
  output: {
    jobId?: string
    pages: Array<{
      markdown: string
      html?: string
      metadata: {
        title: string
        description: string
        language: string
        sourceURL: string
        statusCode: number
      }
    }>
    total: number
    creditsUsed: number
  }
}
```

*   **`export interface FirecrawlCrawlResponse extends ToolResponse`**: This defines the interface for the response received after a web crawling operation. It also `extends ToolResponse`.
*   **`output: { ... }`**: A required property, containing the results of the crawl.
    *   **`jobId?: string`**: An *optional* `string`. If the crawl operation is asynchronous, this might be an ID you can use to check the status or retrieve results later.
    *   **`pages: Array<{ ... }> `**: A required array. Each element in this array is an object representing a single page that was successfully crawled.
        *   **`markdown: string`**: The main content of the crawled page, formatted as Markdown.
        *   **`html?: string`**: An *optional* `string`. If provided, it contains the raw HTML content of the crawled page.
        *   **`metadata: { ... }`**: A required object containing metadata for *this individual crawled page*.
            *   **`title: string`**: The HTML `<title>` of the page.
            *   **`description: string`**: The HTML `<meta name="description">` of the page.
            *   **`language: string`**: The detected language of the page.
            *   **`sourceURL: string`**: The exact URL of this crawled page.
            *   **`statusCode: number`**: The HTTP status code received when fetching this `sourceURL`.
    *   **`total: number`**: A required `number`, indicating the total number of pages processed or found during the crawl.
    *   **`creditsUsed: number`**: A required `number`, indicating the amount of API credits consumed by this specific crawl operation.

---

```typescript
export type FirecrawlResponse = ScrapeResponse | SearchResponse | FirecrawlCrawlResponse
```

*   **`export type FirecrawlResponse = ScrapeResponse | SearchResponse | FirecrawlCrawlResponse`**: This line defines a **union type** called `FirecrawlResponse`. This means that any variable or function parameter declared as `FirecrawlResponse` can hold a value that is *either* a `ScrapeResponse`, *or* a `SearchResponse`, *or* a `FirecrawlCrawlResponse`. This is incredibly useful when a single API endpoint might return different types of data based on the specific action requested. Your code can then use type guards (`if ('pages' in response.output)`) to determine which specific response type it has received and process it accordingly.