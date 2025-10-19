```typescript
import type { ArxivGetPaperParams, ArxivGetPaperResponse } from '@/tools/arxiv/types'
import { parseArxivXML } from '@/tools/arxiv/utils'
import type { ToolConfig } from '@/tools/types'

export const getPaperTool: ToolConfig<ArxivGetPaperParams, ArxivGetPaperResponse> = {
  id: 'arxiv_get_paper',
  name: 'ArXiv Get Paper',
  description: 'Get detailed information about a specific ArXiv paper by its ID.',
  version: '1.0.0',

  params: {
    paperId: {
      type: 'string',
      required: true,
      visibility: 'user-or-llm',
      description: 'ArXiv paper ID (e.g., "1706.03762")',
    },
  },

  request: {
    url: (params: ArxivGetPaperParams) => {
      // Clean paper ID - remove arxiv.org URLs if present
      let paperId = params.paperId
      if (paperId.includes('arxiv.org/abs/')) {
        paperId = paperId.split('arxiv.org/abs/')[1]
      }

      const baseUrl = 'https://export.arxiv.org/api/query'
      const searchParams = new URLSearchParams()
      searchParams.append('id_list', paperId)

      return `${baseUrl}?${searchParams.toString()}`
    },
    method: 'GET',
    headers: () => ({
      'Content-Type': 'application/xml',
    }),
  },

  transformResponse: async (response: Response) => {
    const xmlText = await response.text()
    const papers = parseArxivXML(xmlText)

    return {
      success: true,
      output: {
        paper: papers[0] || null,
      },
    }
  },

  outputs: {
    paper: {
      type: 'json',
      description: 'Detailed information about the requested ArXiv paper',
      items: {
        type: 'object',
        properties: {
          id: { type: 'string' },
          title: { type: 'string' },
          summary: { type: 'string' },
          authors: { type: 'string' },
          published: { type: 'string' },
          updated: { type: 'string' },
          link: { type: 'string' },
          pdfLink: { type: 'string' },
          categories: { type: 'string' },
          primaryCategory: { type: 'string' },
          comment: { type: 'string' },
          journalRef: { type: 'string' },
          doi: { type: 'string' },
        },
      },
    },
  },
}
```

## Explanation:

This TypeScript code defines a tool, `getPaperTool`, for retrieving information about a specific paper from the ArXiv API. It leverages the `ToolConfig` type for structure and integrates with ArXiv-specific utilities for data parsing. Let's break down the code section by section:

**1. Imports:**

```typescript
import type { ArxivGetPaperParams, ArxivGetPaperResponse } from '@/tools/arxiv/types'
import { parseArxivXML } from '@/tools/arxiv/utils'
import type { ToolConfig } from '@/tools/types'
```

- `import type { ArxivGetPaperParams, ArxivGetPaperResponse } from '@/tools/arxiv/types'`: This line imports the type definitions `ArxivGetPaperParams` and `ArxivGetPaperResponse` from a file located at `'@/tools/arxiv/types'`.  These types likely define the structure of the input parameters required by the tool (e.g., the ArXiv paper ID) and the structure of the response data the tool is expected to return, respectively. Using `type` ensures that these are only used for type checking and not included in the compiled JavaScript, improving performance.

- `import { parseArxivXML } from '@/tools/arxiv/utils'`: This line imports the function `parseArxivXML` from `'@/tools/arxiv/utils'`. This function is responsible for parsing the XML response received from the ArXiv API and converting it into a more usable JavaScript object (likely an array of paper objects).

- `import type { ToolConfig } from '@/tools/types'`: This line imports the `ToolConfig` type from `'@/tools/types'`. This type likely defines the structure for configuring a tool within a larger system.  It is a generic type, meaning it can be parameterized with the specific input and output types for the tool (in this case, `ArxivGetPaperParams` and `ArxivGetPaperResponse`).

**2. Tool Configuration:**

```typescript
export const getPaperTool: ToolConfig<ArxivGetPaperParams, ArxivGetPaperResponse> = {
  id: 'arxiv_get_paper',
  name: 'ArXiv Get Paper',
  description: 'Get detailed information about a specific ArXiv paper by its ID.',
  version: '1.0.0',

  params: {
    paperId: {
      type: 'string',
      required: true,
      visibility: 'user-or-llm',
      description: 'ArXiv paper ID (e.g., "1706.03762")',
    },
  },

  request: {
    url: (params: ArxivGetPaperParams) => {
      // Clean paper ID - remove arxiv.org URLs if present
      let paperId = params.paperId
      if (paperId.includes('arxiv.org/abs/')) {
        paperId = paperId.split('arxiv.org/abs/')[1]
      }

      const baseUrl = 'https://export.arxiv.org/api/query'
      const searchParams = new URLSearchParams()
      searchParams.append('id_list', paperId)

      return `${baseUrl}?${searchParams.toString()}`
    },
    method: 'GET',
    headers: () => ({
      'Content-Type': 'application/xml',
    }),
  },

  transformResponse: async (response: Response) => {
    const xmlText = await response.text()
    const papers = parseArxivXML(xmlText)

    return {
      success: true,
      output: {
        paper: papers[0] || null,
      },
    }
  },

  outputs: {
    paper: {
      type: 'json',
      description: 'Detailed information about the requested ArXiv paper',
      items: {
        type: 'object',
        properties: {
          id: { type: 'string' },
          title: { type: 'string' },
          summary: { type: 'string' },
          authors: { type: 'string' },
          published: { type: 'string' },
          updated: { type: 'string' },
          link: { type: 'string' },
          pdfLink: { type: 'string' },
          categories: { type: 'string' },
          primaryCategory: { type: 'string' },
          comment: { type: 'string' },
          journalRef: { type: 'string' },
          doi: { type: 'string' },
        },
      },
    },
  },
}
```

This section defines the `getPaperTool` constant, which is of type `ToolConfig<ArxivGetPaperParams, ArxivGetPaperResponse>`.  This means that `getPaperTool` is an object that conforms to the structure defined by the `ToolConfig` type, and it's specifically configured to handle `ArxivGetPaperParams` as input and `ArxivGetPaperResponse` as output.

Let's break this down further:

- `id: 'arxiv_get_paper'`: A unique identifier for the tool.
- `name: 'ArXiv Get Paper'`: A human-readable name for the tool.
- `description: 'Get detailed information about a specific ArXiv paper by its ID.'`: A brief explanation of what the tool does.
- `version: '1.0.0'`: The version number of the tool.

**3. `params` Section:**

```typescript
  params: {
    paperId: {
      type: 'string',
      required: true,
      visibility: 'user-or-llm',
      description: 'ArXiv paper ID (e.g., "1706.03762")',
    },
  },
```

This section defines the input parameters that the tool accepts. In this case, it only accepts one parameter:

- `paperId`:
    - `type: 'string'`: Specifies that the `paperId` must be a string.
    - `required: true`: Indicates that the `paperId` is a mandatory parameter.  The tool cannot function without it.
    - `visibility: 'user-or-llm'`:  This likely controls who or what can see/modify this parameter.  `'user-or-llm'` probably means both a human user and a large language model can provide this parameter.
    - `description: 'ArXiv paper ID (e.g., "1706.03762")'`: Provides a helpful description of the parameter, including an example.

**4. `request` Section:**

```typescript
 request: {
    url: (params: ArxivGetPaperParams) => {
      // Clean paper ID - remove arxiv.org URLs if present
      let paperId = params.paperId
      if (paperId.includes('arxiv.org/abs/')) {
        paperId = paperId.split('arxiv.org/abs/')[1]
      }

      const baseUrl = 'https://export.arxiv.org/api/query'
      const searchParams = new URLSearchParams()
      searchParams.append('id_list', paperId)

      return `${baseUrl}?${searchParams.toString()}`
    },
    method: 'GET',
    headers: () => ({
      'Content-Type': 'application/xml',
    }),
  },
```

This section defines how the tool makes a request to the ArXiv API.

- `url`: This is a function that constructs the URL for the API request.
    - It takes the `params` (of type `ArxivGetPaperParams`) as input.
    - It first cleans the `paperId` by removing the `arxiv.org/abs/` prefix if it exists. This ensures that the API receives only the paper ID.
    - It then constructs the base URL for the ArXiv API query.
    - It uses `URLSearchParams` to create a query string with the `id_list` parameter set to the cleaned `paperId`.
    - Finally, it returns the complete URL, including the query string.
- `method: 'GET'`: Specifies that the request method is a GET request.
- `headers`: This is a function that returns an object containing the request headers.
    - In this case, it sets the `Content-Type` header to `application/xml`, indicating that the expected response is in XML format.

**5. `transformResponse` Section:**

```typescript
transformResponse: async (response: Response) => {
    const xmlText = await response.text()
    const papers = parseArxivXML(xmlText)

    return {
      success: true,
      output: {
        paper: papers[0] || null,
      },
    }
  },
```

This section defines how the tool processes the response received from the ArXiv API.

- It's an `async` function, meaning it can handle asynchronous operations.
- It takes the `response` (a `Response` object from the fetch API) as input.
- `const xmlText = await response.text()`:  This line extracts the response body as a string, assuming it's in XML format. The `await` keyword ensures that the code waits for the response text to be fully retrieved before proceeding.
- `const papers = parseArxivXML(xmlText)`: This line calls the `parseArxivXML` function (imported earlier) to parse the XML text into a JavaScript object, likely an array of paper objects.
- It then returns an object with the following properties:
    - `success: true`: Indicates that the request and response processing were successful.
    - `output`: An object containing the parsed paper data.
        - `paper: papers[0] || null`:  This assigns the first element of the `papers` array to the `paper` property. If the `papers` array is empty (meaning no paper was found with the given ID), it assigns `null` to the `paper` property.

**6. `outputs` Section:**

```typescript
  outputs: {
    paper: {
      type: 'json',
      description: 'Detailed information about the requested ArXiv paper',
      items: {
        type: 'object',
        properties: {
          id: { type: 'string' },
          title: { type: 'string' },
          summary: { type: 'string' },
          authors: { type: 'string' },
          published: { type: 'string' },
          updated: { type: 'string' },
          link: { type: 'string' },
          pdfLink: { type: 'string' },
          categories: { type: 'string' },
          primaryCategory: { type: 'string' },
          comment: { type: 'string' },
          journalRef: { type: 'string' },
          doi: { type: 'string' },
        },
      },
    },
  },
```

This section defines the structure and type of the output data produced by the tool. It describes what the shape of the `paper` object should be.

- `paper`:
    - `type: 'json'`: Specifies that the `paper` object should be represented as JSON. Although it's JSON, the type system within TypeScript allows it to enforce the structure as described by the `items` section.
    - `description: 'Detailed information about the requested ArXiv paper'`: A description of the output.
    - `items`:
        - `type: 'object'`:  Confirms that we expect an object.
        - `properties`: An object that defines the properties of the `paper` object and their respective types.  This provides a schema for the paper object.
            - `id: { type: 'string' }`: The ArXiv paper ID (string).
            - `title: { type: 'string' }`: The title of the paper (string).
            - `summary: { type: 'string' }`: The abstract or summary of the paper (string).
            - `authors: { type: 'string' }`: The authors of the paper (string).
            - `published: { type: 'string' }`: The publication date of the paper (string).
            - `updated: { type: 'string' }`: The last updated date of the paper (string).
            - `link: { type: 'string' }`: A link to the paper on the ArXiv website (string).
            - `pdfLink: { type: 'string' }`: A link to the PDF version of the paper (string).
            - `categories: { type: 'string' }`: The categories the paper belongs to (string).
            - `primaryCategory: { type: 'string' }`: The primary category of the paper (string).
            - `comment: { type: 'string' }`:  Any comments associated with the paper (string).
            - `journalRef: { type: 'string' }`: The journal reference for the paper (string).
            - `doi: { type: 'string' }`: The Digital Object Identifier (DOI) of the paper (string).

**In Summary:**

The `getPaperTool` is a well-structured TypeScript tool that allows you to retrieve information about a specific ArXiv paper using its ID. It handles the API request, parses the XML response, and provides a structured output containing the paper's details.  The use of TypeScript types ensures type safety and helps prevent errors.  The separation of concerns (request, response transformation, and output definition) makes the code more maintainable and easier to understand.
