```typescript
/**
 * Query language parser for logs search
 *
 * Supports syntax like:
 * level:error workflow:"my-workflow" trigger:api cost:>0.005 date:today
 */

// Defines the structure of a parsed filter.  Each filter represents a condition
// to apply to the logs, such as `level:error` or `cost:>0.005`.
export interface ParsedFilter {
  // The field being filtered (e.g., "level", "workflow", "cost").
  field: string;
  // The operator used for comparison (e.g., "=", ">", "<", ">=", "<=", "!=").
  operator: '=' | '>' | '<' | '>=' | '<=' | '!=';
  // The value to compare the field against.  Can be a string, number, or boolean.
  value: string | number | boolean;
  // The original value as it appeared in the query string, before parsing. This is
  // useful for preserving the original formatting (e.g. quotes around strings).
  originalValue: string;
}

// Defines the structure of a parsed query.  A query consists of a set of filters
// and a text search string.
export interface ParsedQuery {
  // An array of parsed filters extracted from the query string.
  filters: ParsedFilter[];
  // Any text from the query string that doesn't match the filter format.  This is
  // treated as a general text search term.
  textSearch: string; // Any remaining text not in field:value format
}

// Defines the valid filter fields and their expected data types. This object acts as a
// schema for the filter parameters.
const FILTER_FIELDS = {
  level: 'string',
  status: 'string', // alias for level
  workflow: 'string',
  trigger: 'string',
  execution: 'string',
  id: 'string',
  cost: 'number',
  duration: 'number',
  date: 'date',
  folder: 'string',
} as const;

// Creates a type `FilterField` which is a union of all keys in `FILTER_FIELDS`.  This
// ensures type safety when accessing filter fields.
type FilterField = keyof typeof FILTER_FIELDS;

/**
 * Parse a search query string into structured filters and text search
 */
export function parseQuery(query: string): ParsedQuery {
  // Initialize an empty array to store the parsed filters.
  const filters: ParsedFilter[] = [];
  // Initialize an empty array to store any text tokens that are not filters.
  const tokens: string[] = [];

  // Regular expression to match filter patterns in the query string.
  //  - `(\w+)`: Matches one or more word characters (letters, numbers, and underscore) for the field name.
  //  - `:`: Matches the colon separator between the field and value.
  //  - `((?:[><!]=?|=)?(?:"[^"]*"|[^\s]+))`: Matches the value with an optional operator.
  //    - `(?:[><!]=?|=)?`:  Non-capturing group for the operator. Matches optional `>=`, `<=`, `!=`, `>`, `<`, or `=`.
  //    - `(?:"[^"]*"|[^\s]+)`: Matches either:
  //      - `"[^"]*"`: A quoted string (anything between double quotes).
  //      - `[^\s]+`: One or more characters that are not whitespace.
  const filterRegex = /(\w+):((?:[><!]=?|=)?(?:"[^"]*"|[^\s]+))/g;

  // Keeps track of the last index processed in the query string.
  let lastIndex = 0;
  // Variable to store the result of each regular expression match.
  let match;

  // Loop through the query string, finding all filter matches.
  while ((match = filterRegex.exec(query)) !== null) {
    // Extract the full match, field name, and value (with operator) from the regex match.
    const [fullMatch, field, valueWithOperator] = match;

    // Extract any text before the current filter match.
    const beforeText = query.slice(lastIndex, match.index).trim();
    // If there's text before the filter, add it to the tokens array.
    if (beforeText) {
      tokens.push(beforeText);
    }

    // Parse the filter using the `parseFilter` function.
    const parsedFilter = parseFilter(field, valueWithOperator);
    // If the filter was successfully parsed, add it to the filters array.
    if (parsedFilter) {
      filters.push(parsedFilter);
    } else {
      // If the filter could not be parsed, treat the entire match as a text token.
      tokens.push(fullMatch);
    }

    // Update `lastIndex` to the end of the current match.
    lastIndex = match.index + fullMatch.length;
  }

  // Extract any remaining text after the last filter match.
  const remainingText = query.slice(lastIndex).trim();
  // If there's remaining text, add it to the tokens array.
  if (remainingText) {
    tokens.push(remainingText);
  }

  // Return the parsed query object containing the filters and the combined text search.
  return {
    filters,
    // Join the tokens array into a single string, separated by spaces, and trim any leading/trailing whitespace.
    textSearch: tokens.join(' ').trim(),
  };
}

/**
 * Parse a single field:value filter
 */
function parseFilter(field: string, valueWithOperator: string): ParsedFilter | null {
  // Check if the field is a valid filter field (defined in `FILTER_FIELDS`).  If not, return null.
  if (!(field in FILTER_FIELDS)) {
    return null;
  }

  // Cast the field to the `FilterField` type to ensure type safety.
  const filterField = field as FilterField;
  // Get the expected data type for the field from the `FILTER_FIELDS` object.
  const fieldType = FILTER_FIELDS[filterField];

  // Initialize the operator to "=", which is the default if no operator is specified.
  let operator: ParsedFilter['operator'] = '=';
  // Initialize the value to the raw value with the operator.
  let value = valueWithOperator;

  // Check for different operators at the beginning of the value string and update the
  // `operator` and `value` accordingly.
  if (value.startsWith('>=')) {
    operator = '>=';
    value = value.slice(2);
  } else if (value.startsWith('<=')) {
    operator = '<=';
    value = value.slice(2);
  } else if (value.startsWith('!=')) {
    operator = '!=';
    value = value.slice(2);
  } else if (value.startsWith('>')) {
    operator = '>';
    value = value.slice(1);
  } else if (value.startsWith('<')) {
    operator = '<';
    value = value.slice(1);
  } else if (value.startsWith('=')) {
    operator = '=';
    value = value.slice(1);
  }

  // Store the original value before any further processing.
  const originalValue = value;
  // If the value is enclosed in double quotes, remove them.
  if (value.startsWith('"') && value.endsWith('"')) {
    value = value.slice(1, -1);
  }

  // Initialize the parsed value.
  let parsedValue: string | number | boolean = value;

  // Convert the value to the correct data type based on the `fieldType`.
  if (fieldType === 'number') {
    // Special handling for duration fields that have units.
    if (field === 'duration' && value.endsWith('ms')) {
      // Parse duration in milliseconds
      parsedValue = Number.parseFloat(value.slice(0, -2));
    } else if (field === 'duration' && value.endsWith('s')) {
      // Parse duration in seconds and convert to milliseconds.
      parsedValue = Number.parseFloat(value.slice(0, -1)) * 1000; // Convert to ms
    } else {
      // Parse the value as a floating-point number.
      parsedValue = Number.parseFloat(value);
    }

    // If the parsed value is NaN (Not a Number), return null.
    if (Number.isNaN(parsedValue)) {
      return null;
    }
  }

  // Return the parsed filter object.
  return {
    field: filterField,
    operator,
    value: parsedValue,
    originalValue,
  };
}

/**
 * Convert parsed query back to URL parameters for the logs API
 */
export function queryToApiParams(parsedQuery: ParsedQuery): Record<string, string> {
  // Initialize an empty object to store the API parameters.
  const params: Record<string, string> = {};

  // If there is a text search, add it to the parameters.
  if (parsedQuery.textSearch) {
    params.search = parsedQuery.textSearch;
  }

  // Iterate over each parsed filter.
  for (const filter of parsedQuery.filters) {
    // Use a switch statement to handle different filter fields.
    switch (filter.field) {
      // Handle `level` and `status` filters (which are aliases for the same parameter).
      case 'level':
      case 'status':
        // If the operator is "=", add the level value to the parameters.
        if (filter.operator === '=') {
          params.level = filter.value as string;
        }
        break;

      // Handle `trigger` filters.
      case 'trigger':
        // If the operator is "=", add the trigger value to the parameters.  If there are
        // multiple triggers, they are comma-separated.
        if (filter.operator === '=') {
          const existing = params.triggers ? params.triggers.split(',') : [];
          existing.push(filter.value as string);
          params.triggers = existing.join(',');
        }
        break;

      // Handle `workflow` filters.
      case 'workflow':
        // If the operator is "=", add the workflow name to the parameters.
        if (filter.operator === '=') {
          params.workflowName = filter.value as string;
        }
        break;

      case 'folder':
        if (filter.operator === '=') {
          params.folderName = filter.value as string;
        }
        break;

      // Handle `execution` filters.
      case 'execution':
        // If the operator is "=", add the execution value to the search parameters. If `textSearch` exists
        // then append the filter value to the text search string.
        if (filter.operator === '=' && parsedQuery.textSearch) {
          params.search = `${parsedQuery.textSearch} ${filter.value}`.trim();
        } else if (filter.operator === '=') {
          params.search = filter.value as string;
        }
        break;

      // Handle `workflowId` filters.
      case 'workflowId':
        // If the operator is "=", add the workflow ID to the parameters.
        if (filter.operator === '=') {
          params.workflowIds = String(filter.value);
        }
        break;

      // Handle `executionId` filters.
      case 'executionId':
        // If the operator is "=", add the execution ID to the parameters.
        if (filter.operator === '=') {
          params.executionId = String(filter.value);
        }
        break;

      // Handle `date` filters.
      case 'date':
        // If the operator is "=" and the value is "today", set the start date to today.
        if (filter.operator === '=' && filter.value === 'today') {
          const today = new Date();
          today.setHours(0, 0, 0, 0); // Set time to midnight
          params.startDate = today.toISOString();
        }
        // If the operator is "=" and the value is "yesterday", set the start date to yesterday and end date to the end of yesterday.
        else if (filter.operator === '=' && filter.value === 'yesterday') {
          const yesterday = new Date();
          yesterday.setDate(yesterday.getDate() - 1); // Subtract one day
          yesterday.setHours(0, 0, 0, 0); // Set time to midnight
          params.startDate = yesterday.toISOString();

          const endOfYesterday = new Date(yesterday);
          endOfYesterday.setHours(23, 59, 59, 999); // Set time to the end of the day
          params.endDate = endOfYesterday.toISOString();
        }
        break;

      // Handle `cost` filters.
      case 'cost':
        // Create a parameter based on the operator and value.  The value is not important
        // so set to "true".
        params[`cost_${filter.operator}_${filter.value}`] = 'true';
        break;

      // Handle `duration` filters.
      case 'duration':
        // Create a parameter based on the operator and value.  The value is not important
        // so set to "true".
        params[`duration_${filter.operator}_${filter.value}`] = 'true';
        break;
    }
  }

  // Return the object containing the API parameters.
  return params;
}
```

**Purpose of this file:**

This file provides a query language parser specifically designed for log searches. It allows users to write search queries with a structured syntax (e.g., `level:error workflow:"my-workflow"`) and transforms these queries into a format suitable for an API.  The core functionality includes:

1.  **Parsing**: Converting a user-provided query string into structured data (filters and text search).
2.  **Filtering**: Extracting and interpreting specific filter criteria from the query (e.g., `level:error`, `cost:>0.005`).
3.  **Conversion**: Transforming the parsed query into URL parameters that can be used to call a logs API.

**Simplification of Complex Logic:**

*   **Regular Expressions**:  The `filterRegex` is the heart of the parsing logic. It uses a regular expression to efficiently identify and extract filter components from the query string.  The regex is carefully constructed to handle various operators (`=`, `>`, `<`, `>=`, `<=`, `!=`) and quoted string values.
*   **Type Definitions**:  The `ParsedFilter` and `ParsedQuery` interfaces provide clear data structures, making the code easier to understand and maintain. `FILTER_FIELDS` creates a single source of truth for acceptable filter values and provides type safety.
*   **Helper Functions**: The `parseFilter` function encapsulates the logic for parsing individual filters, separating it from the main `parseQuery` function.  This improves readability and makes the code more modular.
*   **Switch Statement**:  The `queryToApiParams` function uses a `switch` statement to handle different filter fields, making the logic more organized and easier to extend.

**Explanation of Each Line of Code:**

1.  **`/** ... */` (JSDoc Comments):**  Provides documentation for the file, interfaces, and functions.  Crucial for understanding the purpose and usage of each element.
2.  **`export interface ParsedFilter { ... }`:**  Defines the `ParsedFilter` interface, which represents a single filter extracted from the query string.  It includes the `field` being filtered, the `operator` used for comparison, the `value` being compared against, and the original raw value.
3.  **`export interface ParsedQuery { ... }`:**  Defines the `ParsedQuery` interface, which represents the overall parsed query.  It contains an array of `ParsedFilter` objects and a `textSearch` string for any unstructured search terms.
4.  **`const FILTER_FIELDS = { ... } as const;`:**  Defines a constant object, `FILTER_FIELDS`, which acts as a schema.  It lists all the valid filter fields (e.g., `level`, `workflow`, `cost`) and their corresponding data types (`string`, `number`, etc.).  The `as const` assertion ensures that the object is treated as read-only and that its keys are treated as literal types, making this strongly typed.  If you try to use a filter that is not in this list it will cause a Typescript error.
5.  **`type FilterField = keyof typeof FILTER_FIELDS;`:**  Creates a type alias, `FilterField`, which is a union of all the keys in the `FILTER_FIELDS` object.  This is useful for type-checking and ensuring that only valid filter fields are used.
6.  **`export function parseQuery(query: string): ParsedQuery { ... }`:** Defines the main function, `parseQuery`, which takes a query string as input and returns a `ParsedQuery` object.  This is the primary entry point for parsing a query.
7.  **`const filters: ParsedFilter[] = [];`:**  Initializes an empty array, `filters`, to store the parsed filter objects.
8.  **`const tokens: string[] = [];`:**  Initializes an empty array, `tokens`, to store any text segments from the query that are not part of a structured filter. This captures the "textSearch" portion of the query.
9.  **`const filterRegex = /(\w+):((?:[><!]=?|=)?(?:"[^"]*"|[^\s]+))/g;`:** Defines a regular expression, `filterRegex`, used to match filter patterns in the query string.  The regex is designed to capture the field name, operator (if present), and value of each filter.  The `g` flag ensures that all matches in the string are found.
10. **`let lastIndex = 0;`:** Initializes `lastIndex` variable. This variable is used to keep track of the current position being parsed within the input `query` string.
11. **`let match;`:** Initializes `match` variable. This variable will hold the result of each attempt to match the regular expression pattern (`filterRegex`) against the `query` string.
12. **`while ((match = filterRegex.exec(query)) !== null) { ... }`:**  Starts a loop that iterates over all filter matches in the query string. The loop continues as long as the `filterRegex.exec(query)` finds a match (i.e., returns a non-null value). In each iteration, the `match` variable is assigned the match object returned by `filterRegex.exec(query)`, which contains information about the matched substring, including the matched groups.
13. **`const [fullMatch, field, valueWithOperator] = match;`:** Destructures the `match` array into individual variables: `fullMatch` (the entire matched string), `field` (the field name), and `valueWithOperator` (the value with any associated operator).
14. **`const beforeText = query.slice(lastIndex, match.index).trim();`:** Extracts the text from the query string that appears before the current filter match.  It trims any leading/trailing whitespace from the extracted text.
15. **`if (beforeText) { tokens.push(beforeText); }`:** Checks if there is any text before current match (`beforeText`). If yes, adds this `beforeText` into the `tokens` array, because it is outside of the filters and represents a part of the `textSearch`.
16. **`const parsedFilter = parseFilter(field, valueWithOperator);`:**  Calls the `parseFilter` function to parse the current filter (field and value).
17. **`if (parsedFilter) { filters.push(parsedFilter); } else { tokens.push(fullMatch); }`:** If the `parsedFilter` returns a filter successfully then push this filter to `filters` array otherwise, it will push full match to the `tokens` array.
18. **`lastIndex = match.index + fullMatch.length;`:** Updates the `lastIndex` to point to the end of the current match.  This ensures that the next iteration of the loop starts after the current filter.
19. **`const remainingText = query.slice(lastIndex).trim();`:**  Extracts any remaining text from the query string after the last filter. Trims whitespace.
20. **`if (remainingText) { tokens.push(remainingText); }`:**  Adds `remainingText` to `tokens` array for the text search.
21. **`return { filters, textSearch: tokens.join(' ').trim(), };`:**  Returns the `ParsedQuery` object with the `filters` and `textSearch` properties set. The `textSearch` is constructed by joining the elements of `tokens` with a space.
22. **`function parseFilter(field: string, valueWithOperator: string): ParsedFilter | null { ... }`:** Defines a function, `parseFilter`, which takes a field name and value (with operator) as input and returns a `ParsedFilter` object or null if parsing fails.
23. **`if (!(field in FILTER_FIELDS)) { return null; }`:**  Checks if the field is a valid filter field.  If not, returns `null` to indicate parsing failure.
24. **`const filterField = field as FilterField;`:**  Casts the `field` to the `FilterField` type.
25. **`const fieldType = FILTER_FIELDS[filterField];`:**  Gets the data type associated with the field from the `FILTER_FIELDS` object.
26. **`let operator: ParsedFilter['operator'] = '=';`:**  Initializes the `operator` variable to "=", the default operator.
27. **`let value = valueWithOperator;`:**  Initializes the `value` variable to the original value with the operator.
28. **`if (value.startsWith('>=')) { ... }`:** Series of `if/else if` conditions to check if `value` begins with different operators. Depending on the operator, the corresponding value will be assigned to the `operator` variable and the value will be sliced so it won't include the operator.
29. **`const originalValue = value;`:**  Stores the original value for inclusion in the `ParsedFilter` object.
30. **`if (value.startsWith('"') && value.endsWith('"')) { value = value.slice(1, -1); }`:** Removes surrounding quotes.
31. **`let parsedValue: string | number | boolean = value;`:**  Initializes the `parsedValue` variable.
32. **`if (fieldType === 'number') { ... }`:**  If the `fieldType` is number it will use `parseFloat` to parse the value to a number. If the parsing fails (e.g. invalid number) it will return `null`.
33. **`return { field: filterField, operator, value: parsedValue, originalValue, };`:** Returns the `ParsedFilter` object with the parsed data.
34. **`export function queryToApiParams(parsedQuery: ParsedQuery): Record<string, string> { ... }`:** Defines a function, `queryToApiParams`, that takes a `ParsedQuery` object and converts it into a record of URL parameters suitable for an API request.
35. **`const params: Record<string, string> = {};`:**  Initializes an empty object, `params`, to store the URL parameters.
36. **`if (parsedQuery.textSearch) { params.search = parsedQuery.textSearch; }`:**  If there is a `textSearch` value in the parsed query, add it to the `params` object under the key "search".
37. **`for (const filter of parsedQuery.filters) { ... }`:** Starts a loop that iterates over each filter in the `parsedQuery.filters` array.
38. **`switch (filter.field) { ... }`:**  A `switch` statement that handles different filter fields and generates the corresponding URL parameters. Each `case` represents a different filter field (e.g., `level`, `workflow`, `cost`). The code within each `case` block generates the URL parameters based on the filter's field, operator, and value.
39. **`return params;`:**  Returns the `params` object containing the generated URL parameters.

In essence, this code provides a flexible and robust way to parse log search queries, making it easier for users to find the information they need within their logs. The use of regular expressions, type definitions, and helper functions contributes to the code's clarity, maintainability, and extensibility.
