```typescript
import { readFile } from 'fs/promises'
import * as cheerio from 'cheerio'
import type { FileParseResult, FileParser } from '@/lib/file-parsers/types'
import { sanitizeTextForUTF8 } from '@/lib/file-parsers/utils'
import { createLogger } from '@/lib/logs/console/logger'

const logger = createLogger('HtmlParser')

export class HtmlParser implements FileParser {
  async parseFile(filePath: string): Promise<FileParseResult> {
    try {
      if (!filePath) {
        throw new Error('No file path provided')
      }

      const buffer = await readFile(filePath)
      return this.parseBuffer(buffer)
    } catch (error) {
      logger.error('HTML file error:', error)
      throw new Error(`Failed to parse HTML file: ${(error as Error).message}`)
    }
  }

  async parseBuffer(buffer: Buffer): Promise<FileParseResult> {
    try {
      logger.info('Parsing HTML buffer, size:', buffer.length)

      const htmlContent = buffer.toString('utf-8')
      const $ = cheerio.load(htmlContent)

      // Extract meta information before removing tags
      const title = $('title').text().trim()
      const metaDescription = $('meta[name="description"]').attr('content') || ''

      $('script, style, noscript, meta, link, iframe, object, embed, svg').remove()

      $.root()
        .contents()
        .filter(function () {
          return this.type === 'comment'
        })
        .remove()

      const content = this.extractStructuredText($)

      const sanitizedContent = sanitizeTextForUTF8(content)

      const characterCount = sanitizedContent.length
      const wordCount = sanitizedContent.split(/\s+/).filter((word) => word.length > 0).length
      const estimatedTokenCount = Math.ceil(characterCount / 4)

      const headings = this.extractHeadings($)

      const links = this.extractLinks($)

      return {
        content: sanitizedContent,
        metadata: {
          title,
          metaDescription,
          characterCount,
          wordCount,
          tokenCount: estimatedTokenCount,
          headings,
          links: links.slice(0, 50),
          hasImages: $('img').length > 0,
          imageCount: $('img').length,
          hasTable: $('table').length > 0,
          tableCount: $('table').length,
          hasList: $('ul, ol').length > 0,
          listCount: $('ul, ol').length,
        },
      }
    } catch (error) {
      logger.error('HTML buffer parsing error:', error)
      throw new Error(`Failed to parse HTML buffer: ${(error as Error).message}`)
    }
  }

  /**
   * Extract structured text content preserving document hierarchy
   */
  private extractStructuredText($: cheerio.CheerioAPI): string {
    const contentParts: string[] = []

    const rootElement = $('body').length > 0 ? $('body') : $.root()

    this.processElement($, rootElement, contentParts, 0)

    return contentParts.join('\n').trim()
  }

  /**
   * Recursively process elements to extract text with structure
   */
  private processElement(
    $: cheerio.CheerioAPI,
    element: cheerio.Cheerio<any>,
    contentParts: string[],
    depth: number
  ): void {
    element.contents().each((_, node) => {
      if (node.type === 'text') {
        const text = $(node).text().trim()
        if (text) {
          contentParts.push(text)
        }
      } else if (node.type === 'tag') {
        const $node = $(node)
        const tagName = node.tagName?.toLowerCase()

        switch (tagName) {
          case 'h1':
          case 'h2':
          case 'h3':
          case 'h4':
          case 'h5':
          case 'h6': {
            const headingText = $node.text().trim()
            if (headingText) {
              contentParts.push(`\n${headingText}\n`)
            }
            break
          }

          case 'p': {
            const paragraphText = $node.text().trim()
            if (paragraphText) {
              contentParts.push(`${paragraphText}\n`)
            }
            break
          }

          case 'br':
            contentParts.push('\n')
            break

          case 'hr':
            contentParts.push('\n---\n')
            break

          case 'li': {
            const listItemText = $node.text().trim()
            if (listItemText) {
              const indent = '  '.repeat(Math.min(depth, 3))
              contentParts.push(`${indent}• ${listItemText}`)
            }
            break
          }

          case 'ul':
          case 'ol':
            contentParts.push('\n')
            this.processElement($, $node, contentParts, depth + 1)
            contentParts.push('\n')
            break

          case 'table':
            this.processTable($, $node, contentParts)
            break

          case 'blockquote': {
            const quoteText = $node.text().trim()
            if (quoteText) {
              contentParts.push(`\n> ${quoteText}\n`)
            }
            break
          }

          case 'pre':
          case 'code': {
            const codeText = $node.text().trim()
            if (codeText) {
              contentParts.push(`\n\`\`\`\n${codeText}\n\`\`\`\n`)
            }
            break
          }

          case 'div':
          case 'section':
          case 'article':
          case 'main':
          case 'aside':
          case 'nav':
          case 'header':
          case 'footer':
            this.processElement($, $node, contentParts, depth)
            break

          case 'a': {
            const linkText = $node.text().trim()
            const href = $node.attr('href')
            if (linkText) {
              if (href?.startsWith('http')) {
                contentParts.push(`${linkText} (${href})`)
              } else {
                contentParts.push(linkText)
              }
            }
            break
          }

          case 'img': {
            const alt = $node.attr('alt')
            if (alt) {
              contentParts.push(`[Image: ${alt}]`)
            }
            break
          }

          default:
            this.processElement($, $node, contentParts, depth)
        }
      }
    })
  }

  /**
   * Process table elements to extract structured data
   */
  private processTable(
    $: cheerio.CheerioAPI,
    table: cheerio.Cheerio<any>,
    contentParts: string[]
  ): void {
    contentParts.push('\n[Table]')

    table.find('tr').each((_, row) => {
      const $row = $(row)
      const cells: string[] = []

      $row.find('td, th').each((_, cell) => {
        const cellText = $(cell).text().trim()
        cells.push(cellText || '')
      })

      if (cells.length > 0) {
        contentParts.push(`| ${cells.join(' | ')} |`)
      }
    })

    contentParts.push('[/Table]\n')
  }

  /**
   * Extract heading structure for metadata
   */
  private extractHeadings($: cheerio.CheerioAPI): Array<{ level: number; text: string }> {
    const headings: Array<{ level: number; text: string }> = []

    $('h1, h2, h3, h4, h5, h6').each((_, element) => {
      const $element = $(element)
      const tagName = element.tagName?.toLowerCase()
      const level = Number.parseInt(tagName?.charAt(1) || '1', 10)
      const text = $element.text().trim()

      if (text) {
        headings.push({ level, text })
      }
    })

    return headings
  }

  /**
   * Extract links from the document
   */
  private extractLinks($: cheerio.CheerioAPI): Array<{ text: string; href: string }> {
    const links: Array<{ text: string; href: string }> = []

    $('a[href]').each((_, element) => {
      const $element = $(element)
      const href = $element.attr('href')
      const text = $element.text().trim()

      if (href && text && href.startsWith('http')) {
        links.push({ text, href })
      }
    })

    return links
  }
}
```

## Detailed Explanation of the HTML Parser

This TypeScript code defines a class `HtmlParser` responsible for parsing HTML files or buffers. It extracts relevant content and metadata from the HTML, making it suitable for further processing, such as indexing or analysis. The parser leverages the `cheerio` library for efficient HTML manipulation and traversal.

### 1. Imports

*   `import { readFile } from 'fs/promises'`: Imports the `readFile` function from the `fs/promises` module, which allows asynchronous reading of files.
*   `import * as cheerio from 'cheerio'`: Imports the `cheerio` library, which provides a jQuery-like interface for parsing and manipulating HTML.
*   `import type { FileParseResult, FileParser } from '@/lib/file-parsers/types'`: Imports the types `FileParseResult` and `FileParser` from a local module. These types define the structure of the parser's output and the interface it implements.
*   `import { sanitizeTextForUTF8 } from '@/lib/file-parsers/utils'`: Imports the `sanitizeTextForUTF8` function, which ensures the extracted text is properly encoded in UTF-8.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports the `createLogger` function to create a logger instance for logging information and errors.

### 2. Logger Initialization

*   `const logger = createLogger('HtmlParser')`: Creates a logger instance named 'HtmlParser'. This logger is used throughout the class to record important events and potential errors.

### 3. HtmlParser Class

The `HtmlParser` class implements the `FileParser` interface, meaning it provides a method for parsing files and returning a structured result.

#### 3.1. `parseFile(filePath: string): Promise<FileParseResult>`

This method is the main entry point for parsing an HTML file.

*   It takes the `filePath` as input, which is the path to the HTML file.
*   **Error Handling:** It checks if `filePath` is provided.  If not, it throws an error.
*   **File Reading:** It uses `readFile` to asynchronously read the file content into a `Buffer`.
*   **Buffer Parsing:** It calls `this.parseBuffer(buffer)` to process the buffer and extract the relevant information.
*   **Error Handling:** It includes a `try...catch` block to handle potential errors during file reading or parsing. If an error occurs, it logs the error using the logger and re-throws a new error with a more descriptive message.

#### 3.2. `parseBuffer(buffer: Buffer): Promise<FileParseResult>`

This method parses the HTML content provided as a `Buffer`.

*   It takes the `buffer` containing the HTML content.
*   **Logging:** Logs the buffer size using the logger.
*   **HTML Loading:** It converts the `Buffer` to a UTF-8 string using `buffer.toString('utf-8')` and then loads the HTML content into `cheerio` using `cheerio.load(htmlContent)`. The `$` variable is now a Cheerio object, providing a jQuery-like interface for traversing and manipulating the HTML.
*   **Metadata Extraction:**
    *   `const title = $('title').text().trim()`: Extracts the content of the `<title>` tag, trims any leading/trailing whitespace, and stores it in the `title` variable.
    *   `const metaDescription = $('meta[name="description"]').attr('content') || ''`: Extracts the value of the `content` attribute from the `<meta>` tag with `name="description"`. If the attribute is not found, it defaults to an empty string.
*   **Tag Removal:** It removes unwanted tags (`script`, `style`, `noscript`, `meta`, `link`, `iframe`, `object`, `embed`, `svg`) from the HTML to simplify the content and prevent interference with the text extraction.
*   **Comment Removal:** It removes all HTML comments.
*   **Content Extraction:**  Calls the `this.extractStructuredText($)` method to extract the main content of the HTML, preserving the document structure.
*   **Sanitization:** It calls `sanitizeTextForUTF8` to ensure that the extracted content is properly encoded in UTF-8.
*   **Counting:**
    *   `const characterCount = sanitizedContent.length`: Calculates the number of characters in the sanitized content.
    *   `const wordCount = sanitizedContent.split(/\s+/).filter((word) => word.length > 0).length`: Calculates the number of words by splitting the content by whitespace and filtering out empty strings.
    *   `const estimatedTokenCount = Math.ceil(characterCount / 4)`:  Estimates the number of tokens, which is often used for language models, by dividing the character count by 4 (a common approximation).
*   **Headings and Links Extraction**: Calls `this.extractHeadings($)` and `this.extractLinks($)` to extract headings and links from the HTML.
*   **Result Construction:** It constructs the `FileParseResult` object, which contains:
    *   `content`: The sanitized text content.
    *   `metadata`: An object containing:
        *   `title`: The extracted title.
        *   `metaDescription`: The extracted meta description.
        *   `characterCount`: The character count.
        *   `wordCount`: The word count.
        *   `tokenCount`: The estimated token count.
        *   `headings`: The extracted headings.
        *   `links`: The extracted links (limited to the first 50).
        *   `hasImages`: A boolean indicating whether the HTML contains images.
        *   `imageCount`: The number of images in the HTML.
        *   `hasTable`: A boolean indicating whether the HTML contains tables.
        *   `tableCount`: The number of tables in the HTML.
        *   `hasList`: A boolean indicating whether the HTML contains lists (unordered or ordered).
        *   `listCount`: The number of lists in the HTML.
*   **Error Handling:** Similar to `parseFile`, it includes a `try...catch` block to handle errors during parsing and re-throws a new error with a more descriptive message.

#### 3.3. `extractStructuredText($: cheerio.CheerioAPI): string`

This method is the core of the HTML parsing logic. It aims to extract the text content while preserving some structural information from the HTML.

*   It initializes an empty array `contentParts` to store the extracted text segments.
*   It determines the root element for processing. If the HTML contains a `<body>` tag, it uses that as the root. Otherwise, it uses the root element of the entire document (`$.root()`).
*   It calls the `this.processElement()` method, which recursively traverses the HTML tree and extracts the content.
*   It joins the elements of the `contentParts` array with newline characters (`\n`) and trims the result to remove any leading/trailing whitespace.

#### 3.4. `processElement($: cheerio.CheerioAPI, element: cheerio.Cheerio<any>, contentParts: string[], depth: number): void`

This recursive method processes each element in the HTML tree and extracts its text content based on the tag name.

*   It iterates over the children of the current `element` using `element.contents().each()`.
*   For each child `node`:
    *   **Text Nodes:** If the node is a text node (`node.type === 'text'`), it extracts the text, trims it, and adds it to the `contentParts` array.
    *   **Tag Nodes:** If the node is a tag node (`node.type === 'tag'`), it determines the tag name and performs different actions based on the tag:
        *   **Headings (h1-h6):** Extracts the heading text, trims it, and adds it to `contentParts` with surrounding newline characters for better formatting.
        *   **Paragraphs (p):** Extracts the paragraph text, trims it, and adds it to `contentParts` followed by a newline character.
        *   **Line Breaks (br):** Adds a newline character to `contentParts`.
        *   **Horizontal Rules (hr):** Adds a separator line (`\n---\n`) to `contentParts`.
        *   **List Items (li):** Extracts the list item text, trims it, and adds it to `contentParts` with indentation based on the `depth` of the list.
        *   **Lists (ul, ol):** Adds newline characters and recursively calls `this.processElement()` to process the list's children, increasing the `depth` to control indentation.
        *   **Tables (table):** Calls `this.processTable()` to process the table and extract its data.
        *   **Blockquotes (blockquote):** Extracts the quoted text, trims it, and adds it to `contentParts` with `>` prefix.
        *   **Code (pre, code):** Extracts the code text, trims it, and adds it to `contentParts` wrapped in triple backticks (`````).
        *   **Div, Section, Article, Main, Aside, Nav, Header, Footer:** Recursively calls `this.processElement()` to process the children of these container elements.
        *   **Links (a):** Extracts the link text and the `href` attribute. If the `href` starts with "http", it adds the link text and URL to `contentParts` in the format "text (URL)". Otherwise, it adds only the link text.
        *   **Images (img):** Extracts the `alt` attribute (alternative text) and adds it to `contentParts` in the format "[Image: alt]".
        *   **Default:** For other tags, it recursively calls `this.processElement()` to process their children.  This ensures that the entire HTML tree is traversed.

#### 3.5. `processTable($: cheerio.CheerioAPI, table: cheerio.Cheerio<any>, contentParts: string[]): void`

This method extracts data from a table element.

*   Adds a `[Table]` marker to `contentParts`.
*   Iterates over each row (`tr`) in the table.
*   For each row, it iterates over each cell (`td` or `th`).
*   It extracts the text from each cell, trims it, and adds it to a `cells` array.
*   After processing all cells in a row, it joins the cells with `" | "` and adds the resulting string (enclosed in `"| "` on both sides) to `contentParts`.
*   Adds a `[/Table]` marker to `contentParts`.

#### 3.6. `extractHeadings($: cheerio.CheerioAPI): Array<{ level: number; text: string }>`

This method extracts the heading structure (h1-h6) from the HTML.

*   It initializes an empty array `headings` to store the extracted headings.
*   It selects all heading tags (h1-h6) using `$('h1, h2, h3, h4, h5, h6')`.
*   For each heading tag:
    *   It extracts the tag name and determines the heading level (e.g., h1 is level 1, h2 is level 2, etc.).
    *   It extracts the heading text and trims it.
    *   It creates an object with the `level` and `text` and adds it to the `headings` array.
*   It returns the `headings` array.

#### 3.7. `extractLinks($: cheerio.CheerioAPI): Array<{ text: string; href: string }>`

This method extracts links from the HTML.

*   It initializes an empty array `links` to store the extracted links.
*   It selects all `<a>` tags with an `href` attribute using `$('a[href]')`.
*   For each link tag:
    *   It extracts the `href` attribute.
    *   It extracts the link text and trims it.
    *   If both `href` and `text` are present, and `href` starts with "http", it creates an object with the `text` and `href` and adds it to the `links` array.
*   It returns the `links` array.

### Purpose of the File

The purpose of this file is to provide a robust and flexible HTML parsing solution. It extracts relevant content and metadata from HTML documents, making it useful for various applications such as:

*   **Content Indexing:**  Preparing HTML content for search engines.
*   **Data Extraction:**  Pulling structured data from web pages.
*   **Text Analysis:**  Analyzing the text content of HTML documents.
*   **Content Summarization:** Creating summaries of HTML documents.

### Simplification of Complex Logic

*   **Cheerio Library:** Using `cheerio` simplifies HTML parsing and manipulation, providing a jQuery-like interface for traversing the DOM.
*   **Recursive Processing:**  The `processElement` function handles nested HTML structures using recursion, making the code more manageable and easier to understand.
*   **Specific Tag Handling:** The `switch` statement in `processElement` allows for specific handling of different HTML tags, making it easy to customize the parsing logic for each tag type.
*   **Modular Design:** Breaking down the parsing process into smaller, well-defined functions (e.g., `extractHeadings`, `extractLinks`, `processTable`) improves code readability and maintainability.
*   **Error Handling:**  The consistent use of `try...catch` blocks ensures that errors are handled gracefully and provide informative error messages.
*   **Clear Variable Names:** Descriptive variable names improve code clarity and make it easier to understand the purpose of each variable.
*   **Comments:** Comments explain the purpose of code sections and individual lines, further enhancing readability.

In summary, this `HtmlParser` class provides a comprehensive solution for parsing HTML documents, extracting relevant content, and providing valuable metadata. Its modular design, recursive processing, and error handling make it a robust and maintainable component for various applications.
