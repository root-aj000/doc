```typescript
/**
 * Mock Data for Tool Tests
 *
 * This file contains mock data samples to be used in tool unit tests.
 */

// HTTP Request Mock Data
export const mockHttpResponses = {
  simple: {
    data: { message: 'Success', status: 'ok' },
    status: 200,
    headers: { 'content-type': 'application/json' },
  },
  error: {
    error: { message: 'Bad Request', code: 400 },
    status: 400,
  },
  notFound: {
    error: { message: 'Not Found', code: 404 },
    status: 404,
  },
  unauthorized: {
    error: { message: 'Unauthorized', code: 401 },
    status: 401,
  },
}

// Gmail Mock Data
export const mockGmailResponses = {
  // List messages response
  messageList: {
    messages: [
      { id: 'msg1', threadId: 'thread1' },
      { id: 'msg2', threadId: 'thread2' },
      { id: 'msg3', threadId: 'thread3' },
    ],
    nextPageToken: 'token123',
  },

  // Empty list response
  emptyList: {
    messages: [],
    resultSizeEstimate: 0,
  },

  // Single message response
  singleMessage: {
    id: 'msg1',
    threadId: 'thread1',
    labelIds: ['INBOX', 'UNREAD'],
    snippet: 'This is a snippet preview of the email...',
    payload: {
      headers: [
        { name: 'From', value: 'sender@example.com' },
        { name: 'To', value: 'recipient@example.com' },
        { name: 'Subject', value: 'Test Email Subject' },
        { name: 'Date', value: 'Mon, 15 Mar 2025 10:30:00 -0800' },
      ],
      mimeType: 'multipart/alternative',
      parts: [
        {
          mimeType: 'text/plain',
          body: {
            data: Buffer.from('This is the plain text content of the email').toString('base64'),
          },
        },
        {
          mimeType: 'text/html',
          body: {
            data: Buffer.from('<div>This is the HTML content of the email</div>').toString(
              'base64'
            ),
          },
        },
      ],
    },
  },
}

// Google Drive Mock Data
export const mockDriveResponses = {
  // List files response
  fileList: {
    files: [
      { id: 'file1', name: 'Document1.docx', mimeType: 'application/vnd.google-apps.document' },
      {
        id: 'file2',
        name: 'Spreadsheet.xlsx',
        mimeType: 'application/vnd.google-apps.spreadsheet',
      },
      {
        id: 'file3',
        name: 'Presentation.pptx',
        mimeType: 'application/vnd.google-apps.presentation',
      },
    ],
    nextPageToken: 'drive-page-token',
  },

  // Empty file list
  emptyFileList: {
    files: [],
  },

  // Single file metadata
  fileMetadata: {
    id: 'file1',
    name: 'Document1.docx',
    mimeType: 'application/vnd.google-apps.document',
    webViewLink: 'https://docs.google.com/document/d/123/edit',
    createdTime: '2025-03-15T12:00:00Z',
    modifiedTime: '2025-03-16T10:15:00Z',
    owners: [{ displayName: 'Test User', emailAddress: 'user@example.com' }],
    size: '12345',
  },
}

// Google Sheets Mock Data
export const mockSheetsResponses = {
  // Read range response
  rangeData: {
    range: 'Sheet1!A1:D5',
    majorDimension: 'ROWS',
    values: [
      ['Header1', 'Header2', 'Header3', 'Header4'],
      ['Row1Col1', 'Row1Col2', 'Row1Col3', 'Row1Col4'],
      ['Row2Col1', 'Row2Col2', 'Row2Col3', 'Row2Col4'],
      ['Row3Col1', 'Row3Col2', 'Row3Col3', 'Row3Col4'],
      ['Row4Col1', 'Row4Col2', 'Row4Col3', 'Row4Col4'],
    ],
  },

  // Empty range
  emptyRange: {
    range: 'Sheet1!A1:D5',
    majorDimension: 'ROWS',
    values: [],
  },

  // Update range response
  updateResponse: {
    spreadsheetId: 'spreadsheet123',
    updatedRange: 'Sheet1!A1:D5',
    updatedRows: 5,
    updatedColumns: 4,
    updatedCells: 20,
  },
}

// Pinecone Mock Data
export const mockPineconeResponses = {
  // Vector embedding
  embedding: {
    embedding: Array(1536)
      .fill(0)
      .map(() => Math.random() * 2 - 1),
    metadata: { text: 'Sample text for embedding', id: 'embed-123' },
  },

  // Search results
  searchResults: {
    matches: [
      { id: 'doc1', score: 0.92, metadata: { text: 'Matching text 1' } },
      { id: 'doc2', score: 0.85, metadata: { text: 'Matching text 2' } },
      { id: 'doc3', score: 0.78, metadata: { text: 'Matching text 3' } },
    ],
  },

  // Upsert response
  upsertResponse: {
    upsertedCount: 5,
  },
}

// GitHub Mock Data
export const mockGitHubResponses = {
  // Repository info
  repoInfo: {
    id: 12345,
    name: 'test-repo',
    full_name: 'user/test-repo',
    description: 'A test repository',
    html_url: 'https://github.com/user/test-repo',
    owner: {
      login: 'user',
      id: 54321,
      avatar_url: 'https://avatars.githubusercontent.com/u/54321',
    },
    private: false,
    fork: false,
    created_at: '2025-01-01T00:00:00Z',
    updated_at: '2025-03-15T10:00:00Z',
    pushed_at: '2025-03-15T09:00:00Z',
    default_branch: 'main',
    open_issues_count: 5,
    watchers_count: 10,
    forks_count: 3,
    stargazers_count: 15,
    language: 'TypeScript',
  },

  // PR creation response
  prResponse: {
    id: 12345,
    number: 42,
    title: 'Test PR Title',
    body: 'Test PR description',
    html_url: 'https://github.com/user/test-repo/pull/42',
    state: 'open',
    user: {
      login: 'user',
      id: 54321,
    },
    created_at: '2025-03-15T10:00:00Z',
    updated_at: '2025-03-15T10:05:00Z',
  },
}

// Serper Search Mock Data
export const mockSerperResponses = {
  // Search results
  searchResults: {
    searchParameters: {
      q: 'test query',
      gl: 'us',
      hl: 'en',
    },
    organic: [
      {
        title: 'Test Result 1',
        link: 'https://example.com/1',
        snippet: 'This is a snippet for the first test result.',
        position: 1,
      },
      {
        title: 'Test Result 2',
        link: 'https://example.com/2',
        snippet: 'This is a snippet for the second test result.',
        position: 2,
      },
      {
        title: 'Test Result 3',
        link: 'https://example.com/3',
        snippet: 'This is a snippet for the third test result.',
        position: 3,
      },
    ],
    knowledgeGraph: {
      title: 'Test Knowledge Graph',
      type: 'Test Type',
      description: 'This is a test knowledge graph result',
    },
  },
}

// Slack Mock Data
export const mockSlackResponses = {
  // Message post response
  messageResponse: {
    ok: true,
    channel: 'C1234567890',
    ts: '1627385301.000700',
    message: {
      text: 'This is a test message',
      user: 'U1234567890',
      ts: '1627385301.000700',
      team: 'T1234567890',
    },
  },

  // Error response
  errorResponse: {
    ok: false,
    error: 'channel_not_found',
  },
}

// Tavily Mock Data
export const mockTavilyResponses = {
  // Search results
  searchResults: {
    results: [
      {
        title: 'Test Article 1',
        url: 'https://example.com/article1',
        content: 'This is the content of test article 1.',
        score: 0.95,
      },
      {
        title: 'Test Article 2',
        url: 'https://example.com/article2',
        content: 'This is the content of test article 2.',
        score: 0.87,
      },
      {
        title: 'Test Article 3',
        url: 'https://example.com/article3',
        content: 'This is the content of test article 3.',
        score: 0.72,
      },
    ],
    query: 'test query',
    search_id: 'search-123',
  },
}

// Supabase Mock Data
export const mockSupabaseResponses = {
  // Query response
  queryResponse: {
    data: [
      { id: 1, name: 'Item 1', description: 'Description 1' },
      { id: 2, name: 'Item 2', description: 'Description 2' },
      { id: 3, name: 'Item 3', description: 'Description 3' },
    ],
    error: null,
  },

  // Insert response
  insertResponse: {
    data: [{ id: 4, name: 'Item 4', description: 'Description 4' }],
    error: null,
  },

  // Update response
  updateResponse: {
    data: [{ id: 1, name: 'Updated Item 1', description: 'Updated Description 1' }],
    error: null,
  },

  // Error response
  errorResponse: {
    data: null,
    error: {
      message: 'Database error',
      details: 'Error details',
      hint: 'Error hint',
      code: 'DB_ERROR',
    },
  },
}
```

### Purpose of this File

This TypeScript file serves as a central repository for mock data used in unit tests, specifically for tools that interact with various APIs and services.  Instead of making actual API calls during testing, which can be slow, unreliable, and costly, these mock responses simulate the data that would be returned by those services. This allows for fast, predictable, and isolated testing of tool logic.

### General Structure and Explanation

The file is organized into several sections, each dedicated to a specific external service or API. Each section defines a constant variable (using `export const`) that holds an object containing different mock response scenarios.

Let's break down the structure and explain each part in detail:

1.  **File-Level JSDoc Comment:**

    ```typescript
    /**
     * Mock Data for Tool Tests
     *
     * This file contains mock data samples to be used in tool unit tests.
     */
    ```

    This is a JSDoc-style comment that provides a brief description of the file's purpose.  It's good practice to include such comments for documentation and clarity.

2.  **Section-Specific Mock Data (e.g., `mockHttpResponses`, `mockGmailResponses`):**

    Each section represents a set of mock responses for a particular service.  For example:

    ```typescript
    // HTTP Request Mock Data
    export const mockHttpResponses = { ... };
    ```

    *   `// HTTP Request Mock Data`: A comment that clearly indicates the type of data this section provides.
    *   `export const mockHttpResponses`:  Declares a constant variable named `mockHttpResponses`.  The `export` keyword makes this variable available for use in other files (i.e., test files).  `const` means the variable cannot be reassigned after its initial value is set.
    *   `{ ... }`: An object literal.  This object contains key-value pairs, where each key represents a different scenario (e.g., `simple`, `error`, `notFound`).

3.  **Individual Mock Response Objects (e.g., `simple`, `error` within `mockHttpResponses`):**

    Within each section, there are objects representing individual mock responses.  These objects are designed to mimic the structure and content of the actual API responses. For example:

    ```typescript
      simple: {
        data: { message: 'Success', status: 'ok' },
        status: 200,
        headers: { 'content-type': 'application/json' },
      },
    ```

    *   `simple`:  The key representing a specific scenario (in this case, a successful HTTP request).
    *   `{ ... }`:  An object literal that defines the mock response. The properties in the object vary depending on what would typically be returned by the API.

### Detailed Explanation of Each Section:

Now, let's go through each section in more detail:

**1. `mockHttpResponses`:**

This section provides mock responses for generic HTTP requests.  It includes scenarios for:

*   `simple`: A successful response with a 200 status code, a JSON payload, and a `content-type` header.
*   `error`:  A bad request (400 status code) with an error message and code in the body.
*   `notFound`: A "Not Found" error (404 status code)
*   `unauthorized`: An unauthorized error (401 status code)

**2. `mockGmailResponses`:**

This section simulates responses from the Gmail API.  It includes mocks for:

*   `messageList`:  A list of Gmail messages with their IDs and thread IDs, along with a `nextPageToken` for pagination.
*   `emptyList`:  An empty list of messages, indicating no messages were found.
*   `singleMessage`:  A detailed response for a single Gmail message, including its ID, thread ID, labels, a snippet, headers (From, To, Subject, Date), MIME type, and parts (plain text and HTML content).  The `data` fields within the body are base64 encoded to represent the content in a way that's consistent with how the Gmail API often returns it.

**3. `mockDriveResponses`:**

This section provides mock responses for the Google Drive API.

*   `fileList`:  A list of files with their IDs, names, and MIME types, simulating the response when listing files in a Google Drive folder.
*   `emptyFileList`: Represents an empty list of files.
*   `fileMetadata`:  Represents the metadata for a single file, including its ID, name, MIME type, a web view link, creation and modification timestamps, owner information, and file size.

**4. `mockSheetsResponses`:**

This section simulates Google Sheets API responses.

*   `rangeData`:  Represents the data retrieved from a specific range in a Google Sheet, including the range, major dimension (rows or columns), and the values (a 2D array).
*   `emptyRange`: Represents a range with no data (an empty 2D array).
*   `updateResponse`:  Simulates the response after updating a range in a Google Sheet, including the spreadsheet ID, updated range, and the number of updated rows, columns, and cells.

**5. `mockPineconeResponses`:**

This section provides mock data for interacting with a Pinecone vector database.

*   `embedding`:  Simulates a vector embedding, which is a numerical representation of text.  The `embedding` is an array of 1536 random numbers between -1 and 1. It includes `metadata` such as the original text and an ID. The length of 1536 is standard for the `text-embedding-ada-002` OpenAI model
*   `searchResults`:  Represents the results of a Pinecone search, including a list of matching documents with their IDs, scores (similarity scores), and metadata.
*   `upsertResponse`:  Simulates the response after upserting (inserting or updating) vectors in Pinecone, indicating the number of vectors upserted.

**6. `mockGitHubResponses`:**

This section mocks responses from the GitHub API.

*   `repoInfo`:  Represents information about a GitHub repository, including its ID, name, full name, description, URL, owner details, privacy settings, creation and update timestamps, default branch, and various counts (issues, watchers, forks, stargazers).
*   `prResponse`:  Simulates the response after creating a pull request (PR) in GitHub, including the PR's ID, number, title, body, URL, state, creator information, and creation/update timestamps.

**7. `mockSerperResponses`:**

This section contains mock data for the Serper.dev search API.

*   `searchResults`: Simulates a comprehensive search results page from Serper.  It includes the query parameters used (`searchParameters`), a list of organic search results (`organic`) with titles, links, snippets, and positions, and a knowledge graph result (`knowledgeGraph`).

**8. `mockSlackResponses`:**

This section mocks responses from the Slack API.

*   `messageResponse`:  Simulates a successful response after posting a message to a Slack channel, including the channel ID, timestamp of the message, and the message content itself.
*   `errorResponse`:  Represents an error response from the Slack API, specifically a `channel_not_found` error.

**9. `mockTavilyResponses`:**

This section provides mock data for the Tavily search API.

*   `searchResults`:  Simulates search results from Tavily, including a list of articles with their titles, URLs, content, and relevance scores. It also includes the original search query and a search ID.

**10. `mockSupabaseResponses`:**

This section contains mock data for the Supabase API.

*   `queryResponse`:  Simulates a successful response to a database query, including an array of data (objects representing database rows) and a `null` error.
*   `insertResponse`:  Represents a successful response after inserting data into a Supabase table.
*   `updateResponse`: Represents a successful response after updating data in a Supabase table.
*   `errorResponse`:  Simulates an error response from Supabase, including `null` data and an error object with a message, details, hint, and code.

### How to Use This File in Unit Tests

1.  **Import the desired mock data:**

    In your test file, import the specific mock response you need.  For example:

    ```typescript
    import { mockGmailResponses } from './path/to/this/file';
    ```

2.  **Use the mock data in your tests:**

    Use the imported mock data to simulate API responses in your tests.  This often involves mocking the API client or HTTP request function to return the mock data instead of making a real API call.  For example, if you're using `jest`:

    ```typescript
    jest.mock('node-fetch', () => ({
        __esModule: true,
        default: jest.fn(() => Promise.resolve({
            json: () => Promise.resolve(mockGmailResponses.singleMessage),
            status: 200,
        })),
    }));

    //OR using mockImplementation
    import fetch from 'node-fetch';
    (fetch as jest.MockedFunction<typeof fetch>).mockImplementation(() =>
        Promise.resolve({
            json: () => Promise.resolve(mockGmailResponses.singleMessage),
            status: 200,
        } as any)
    );
    ```

    Then you run your tests and assert the code behaves correctly, using the mock data as the basis for assertions.

### Key Benefits of Using Mock Data

*   **Isolation:** Tests don't depend on external services being available.
*   **Speed:**  Mock data is much faster than making actual API calls.
*   **Predictability:**  Mock data provides consistent results, making tests reliable.
*   **Cost Savings:**  Avoids unnecessary API usage and potential costs associated with external services.
*   **Error Condition Testing:**  Easily simulate error scenarios and edge cases.

### Best Practices

*   **Keep mock data realistic:**  Strive to make the mock data as close as possible to the actual API responses. This makes your tests more accurate.
*   **Update mock data as APIs change:**  When the APIs you're using change, update your mock data accordingly to avoid test failures.
*   **Provide a variety of scenarios:**  Cover different cases, including success, errors, empty results, and edge cases.
*   **Organize your mock data:**  Use clear naming conventions and a logical structure to make it easy to find and use the mock data you need.
*   **Consider using a mocking library:**  Libraries like `jest.mock`, `sinon`, or `nock` can simplify the process of mocking API calls in your tests.

This detailed explanation provides a comprehensive understanding of the purpose, structure, and usage of this mock data file, enabling you to effectively use it in your tool unit tests.
