This TypeScript file defines the configuration for an integration "block" related to **Hunter.io**, an email finder and verification service. In the context of a workflow automation or low-code platform, this `HunterBlock` acts as a reusable component that users can drag and drop into their workflows to interact with the Hunter.io API.

It specifies everything needed for the platform to:
1.  **Display a UI** for configuring Hunter.io operations (e.g., Domain Search, Email Finder).
2.  **Handle user inputs** and present relevant options dynamically.
3.  **Execute the correct Hunter.io API call** based on the configuration.
4.  **Define the expected inputs and outputs** of those API calls.

---

## **Purpose of this File**

This file serves as a **blueprint or schema for integrating Hunter.io functionalities** into a larger application or workflow orchestration system. It achieves this by:

*   **Defining Metadata:** Providing a name, description, icon, and other descriptive information for the Hunter.io block.
*   **Configuring User Interface (UI):** Specifying the various input fields (dropdowns, text inputs) that a user will interact with to choose an Hunter.io operation and provide its parameters. Crucially, it uses conditional logic to show only the relevant input fields for the selected operation.
*   **Specifying Authentication:** Declaring that an API Key is required for Hunter.io interactions.
*   **Mapping UI to API Calls:** Containing logic (`tools.config.tool`) to translate the user's UI selections into the specific Hunter.io API endpoints that need to be invoked.
*   **Defining Data Contracts:** Clearly outlining the expected `inputs` (parameters for the API) and `outputs` (results from the API) for the Hunter.io operations, allowing the platform to correctly pass data to and receive data from this block.

In essence, it encapsulates all the necessary information for a user to configure and run Hunter.io operations within the encompassing platform, abstracting away the underlying API complexities.

---

## **Simplified Complex Logic**

The most "complex" logic in this file revolves around **dynamic UI generation** and **mapping UI choices to API actions**. Let's break it down:

1.  **Dynamic Input Fields (`subBlocks` with `condition`):**
    *   **The Challenge:** Hunter.io offers several different operations (e.g., "Domain Search", "Email Finder", "Email Verifier"). Each operation requires a *different set* of input parameters. How do you present only the relevant fields to the user without overwhelming them?
    *   **The Solution:** The `subBlocks` array contains *all* possible input fields across *all* Hunter.io operations. However, most of these input fields include a `condition` property. This `condition` states that the field should only be visible if a specific other field (in this case, the main `operation` dropdown) has a particular value.
    *   **Example:** When the user selects "Domain Search" from the "Operation" dropdown (`id: 'operation'`, `value: 'hunter_domain_search'`), only the `subBlocks` whose `condition` matches `field: 'operation', value: 'hunter_domain_search'` (like 'Domain', 'Number of Results', 'Email Type', etc.) will become visible. Other fields, like those for 'Email Finder', remain hidden until that operation is selected. This creates a clean, contextual UI.

2.  **Mapping UI Selections to API Tools (`tools.config.tool` function):**
    *   **The Challenge:** The user selects a human-readable "Operation" (e.g., "Domain Search") in the UI. How does the system know which specific Hunter.io API endpoint or internal tool function corresponds to that choice?
    *   **The Solution:** The `tools.config.tool` function acts as a dispatcher. It receives all the user-provided parameters (`params`) from the UI. It then inspects the `operation` parameter (which comes from the "Operation" dropdown's selected value) and uses a `switch` statement to return the exact string identifier (e.g., `'hunter_domain_search'`) that the underlying API integration expects to execute that specific Hunter.io function.
    *   **Additional Logic:** This function also handles data type conversions, such as converting a string input for `limit` (from a UI text field) into a `Number`, as APIs often expect specific numeric types.

---

## **Explanation of Each Line of Code**

```typescript
import { HunterIOIcon } from '@/components/icons'
```
*   **`import { HunterIOIcon } from '@/components/icons'`**: This line imports a React component named `HunterIOIcon`. This component is likely a visual representation (an SVG or image) that will be used as the icon for the Hunter.io block in the platform's UI. The `@/components/icons` path suggests an alias pointing to a common components directory.

```typescript
import { AuthMode, type BlockConfig } from '@/blocks/types'
import type { HunterResponse } from '@/tools/hunter/types'
```
*   **`import { AuthMode, type BlockConfig } from '@/blocks/types'`**:
    *   `AuthMode`: Imports an `enum` (enumeration) called `AuthMode`. This enum defines different ways a block can authenticate (e.g., `ApiKey`, `OAuth`, `None`).
    *   `type BlockConfig`: Imports a type definition named `BlockConfig`. This is the fundamental interface or type that `HunterBlock` must conform to. It defines the structure and expected properties of any block configuration within the platform. The `type` keyword is used for type-only imports, which are entirely removed during compilation, keeping the JavaScript bundle smaller.
    *   `from '@/blocks/types'`: Indicates that these types/enums are part of the core block definition system of the platform.
*   **`import type { HunterResponse } from '@/tools/hunter/types'`**:
    *   `type HunterResponse`: Imports a type definition specifically for the expected response structure when interacting with the Hunter.io API. This type will be used to strongly type the `outputs` of this block, ensuring type safety and autocompletion for results.
    *   `from '@/tools/hunter/types'`: Suggests this type definition is located within the Hunter.io tool's specific type definitions.

---

```typescript
export const HunterBlock: BlockConfig<HunterResponse> = {
  // ... configuration details
};
```
*   **`export const HunterBlock: BlockConfig<HunterResponse> = {`**: This line declares and exports a constant variable named `HunterBlock`.
    *   `export`: Makes `HunterBlock` available for other modules in the application to import and use.
    *   `const HunterBlock`: Declares a constant variable.
    *   `: BlockConfig<HunterResponse>`: This is a TypeScript type annotation. It states that the `HunterBlock` object *must* conform to the `BlockConfig` interface, and critically, the `HunterResponse` type is passed as a generic argument. This means that the `outputs` property within this `BlockConfig` will specifically refer to properties defined in `HunterResponse`.

---

### **Top-Level Block Configuration**

```typescript
  type: 'hunter',
  name: 'Hunter io',
  description: 'Find and verify professional email addresses',
  authMode: AuthMode.ApiKey,
  longDescription:
    'Integrate Hunter into the workflow. Can search domains, find email addresses, verify email addresses, discover companies, find companies, and count email addresses.',
  docsLink: 'https://docs.sim.ai/tools/hunter',
  category: 'tools',
  bgColor: '#E0E0E0',
  icon: HunterIOIcon,
```
*   **`type: 'hunter'`**: A unique string identifier for this specific block type within the platform (e.g., for internal routing or storage).
*   **`name: 'Hunter io'`**: The user-friendly name displayed for the block in the UI.
*   **`description: 'Find and verify professional email addresses'`**: A short, concise summary of what the block does, displayed in the UI.
*   **`authMode: AuthMode.ApiKey`**: Specifies the authentication method required for this block. Here, it indicates that a Hunter.io API key is needed. `AuthMode.ApiKey` comes from the imported `AuthMode` enum.
*   **`longDescription: '...'`**: A more detailed explanation of the block's capabilities, potentially shown in a modal or sidebar in the UI.
*   **`docsLink: 'https://docs.sim.ai/tools/hunter'`**: A URL pointing to external documentation or help for this block.
*   **`category: 'tools'`**: Categorizes the block within the platform (e.g., "Tools", "Data Sources", "AI Models"), aiding discoverability.
*   **`bgColor: '#E0E0E0'`**: The background color (in hexadecimal) to be used when rendering the block in the UI, helping with visual distinction.
*   **`icon: HunterIOIcon`**: The React component (imported earlier) that provides the visual icon for the block.

---

### **`subBlocks`: UI Input Configuration**

This array defines all the configurable input fields that the user will see in the block's settings panel.

```typescript
  subBlocks: [
    {
      id: 'operation',
      title: 'Operation',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'Domain Search', id: 'hunter_domain_search' },
        { label: 'Email Finder', id: 'hunter_email_finder' },
        { label: 'Email Verifier', id: 'hunter_email_verifier' },
        { label: 'Discover Companies', id: 'hunter_discover' },
        { label: 'Find Company', id: 'hunter_companies_find' },
        { label: 'Email Count', id: 'hunter_email_count' },
      ],
      value: () => 'hunter_domain_search',
    },
```
*   This is the primary dropdown that lets the user choose *which* Hunter.io function they want to use.
    *   **`id: 'operation'`**: Unique identifier for this input field.
    *   **`title: 'Operation'`**: The label displayed to the user.
    *   **`type: 'dropdown'`**: Specifies that this is a dropdown UI element.
    *   **`layout: 'full'`**: Indicates it should occupy the full width of its container.
    *   **`options: [...]`**: An array of objects defining the choices in the dropdown.
        *   `label`: The text displayed to the user.
        *   `id`: The internal value associated with that choice, which will be stored as `params.operation`.
    *   **`value: () => 'hunter_domain_search'`**: A function that returns the default selected value for this dropdown when the block is first added.

The subsequent `subBlocks` define input fields for different operations. They often share the same `id` (e.g., `'domain'`) but are differentiated by their `condition`.

```typescript
    // Domain Search operation inputs
    {
      id: 'domain',
      title: 'Domain',
      type: 'short-input',
      layout: 'full',
      required: true,
      placeholder: 'Enter domain name (e.g., stripe.com)',
      condition: { field: 'operation', value: 'hunter_domain_search' },
    },
    { /* ... other Domain Search inputs: limit, type, seniority, department ... */ },
```
*   These blocks define inputs specific to the "Domain Search" operation.
    *   **`id: 'domain'`**: Identifier for the domain input.
    *   **`title: 'Domain'`**: Label.
    *   **`type: 'short-input'`**: A single-line text input field.
    *   **`required: true`**: User must provide a value.
    *   **`placeholder: '...' `**: Text displayed inside the input when it's empty.
    *   **`condition: { field: 'operation', value: 'hunter_domain_search' }`**: **Key dynamic UI logic.** This input field will *only* be visible if the `operation` dropdown (`id: 'operation'`) has the value `hunter_domain_search` selected.

This pattern (a set of inputs with a `condition` matching an `operation` value) repeats for:
*   **Email Finder operation inputs:** (`first_name`, `last_name`, `company`, `domain`)
*   **Email Verifier operation inputs:** (`email`)
*   **Discover operation inputs:** (`query`, `domain`)
*   **Find Company operation inputs:** (`domain`)
*   **Email Count operation inputs:** (`domain`, `company`, `type`)

```typescript
    // API Key (common)
    {
      id: 'apiKey',
      title: 'API Key',
      type: 'short-input',
      layout: 'full',
      required: true,
      placeholder: 'Enter your Hunter.io API key',
      password: true,
    },
  ],
```
*   **`id: 'apiKey'`**: Identifier for the API key input.
*   **`title: 'API Key'`**: Label.
*   **`password: true`**: Specifies that this input field should mask the user's input (like `type="password"` in HTML) for security. This field does *not* have a `condition`, meaning it's always visible regardless of the selected operation, as it's required for all Hunter.io interactions.

---

### **`tools`: API Interaction Logic**

This section defines how the block interacts with the actual Hunter.io API.

```typescript
  tools: {
    access: [
      'hunter_discover',
      'hunter_domain_search',
      'hunter_email_finder',
      'hunter_email_verifier',
      'hunter_companies_find',
      'hunter_email_count',
    ],
    config: {
      tool: (params) => {
        // Convert numeric parameters
        if (params.limit) {
          params.limit = Number(params.limit)
        }

        switch (params.operation) {
          case 'hunter_discover':
            return 'hunter_discover'
          case 'hunter_domain_search':
            return 'hunter_domain_search'
          case 'hunter_email_finder':
            return 'hunter_email_finder'
          case 'hunter_email_verifier':
            return 'hunter_email_verifier'
          case 'hunter_companies_find':
            return 'hunter_companies_find'
          case 'hunter_email_count':
            return 'hunter_email_count'
          default:
            return 'hunter_domain_search'
        }
      },
    },
  },
```
*   **`tools: { ... }`**: An object containing configurations related to the underlying tools/APIs.
    *   **`access: [...]`**: An array of strings. This explicitly lists all the Hunter.io "tool" identifiers that this block is authorized or configured to use. This acts as a security or capability manifest.
    *   **`config: { ... }`**: Further configuration for how tools are called.
        *   **`tool: (params) => { ... }`**: This is a crucial function. It takes an object `params` (which contains all the values from the `subBlocks` inputs) and decides *which specific Hunter.io function* to invoke.
            *   **`if (params.limit) { params.limit = Number(params.limit) }`**: This line demonstrates **input transformation**. The `limit` input from the UI (`short-input`) would initially be a string. This converts it to a `number`, which is likely what the Hunter.io API expects for a "limit" parameter. This is important for data type correctness before making the API call.
            *   **`switch (params.operation) { ... }`**: This `switch` statement dynamically determines which Hunter.io tool to call based on the `operation` value selected by the user in the "Operation" dropdown (`id: 'operation'`).
                *   Each `case` matches an `id` from the `operation` dropdown's `options`.
                *   `return '...'`: The function returns a string identifier that the platform uses to execute the corresponding Hunter.io API function.
            *   **`default: return 'hunter_domain_search'`**: A fallback. If for some reason `params.operation` doesn't match any of the defined cases, it defaults to `hunter_domain_search`.

---

### **`inputs`: API Input Schema**

```typescript
  inputs: {
    operation: { type: 'string', description: 'Operation to perform' },
    apiKey: { type: 'string', description: 'Hunter.io API key' },
    // Domain Search & Email Count
    domain: { type: 'string', description: 'Company domain name' },
    limit: { type: 'number', description: 'Result limit' },
    offset: { type: 'number', description: 'Result offset' },
    type: { type: 'string', description: 'Email type filter' },
    seniority: { type: 'string', description: 'Seniority level filter' },
    department: { type: 'string', description: 'Department filter' },
    // Email Finder
    first_name: { type: 'string', description: 'First name' },
    last_name: { type: 'string', description: 'Last name' },
    company: { type: 'string', description: 'Company name' },
    // Email Verifier & Enrichment
    email: { type: 'string', description: 'Email address' },
    // Discover
    query: { type: 'string', description: 'Search query' },
    headcount: { type: 'string', description: 'Company headcount filter' },
    company_type: { type: 'string', description: 'Company type filter' },
    technology: { type: 'string', description: 'Technology filter' },
  },
```
*   **`inputs: { ... }`**: This object defines the expected structure and types of the parameters that will be passed *to* the Hunter.io API when this block is executed. This is a formal **data contract** for the API, not directly for the UI fields (though UI fields populate these).
    *   Each property (`operation`, `apiKey`, `domain`, etc.) corresponds to a parameter expected by the Hunter.io API.
    *   **`type: 'string'` / `type: 'number'`**: Specifies the TypeScript data type of the parameter. Notice `limit` is `number` here, aligning with the `Number(params.limit)` conversion in `tools.config.tool`.
    *   **`description: '...' `**: A brief explanation of what each input parameter represents.
    *   This section includes *all possible parameters* that *any* Hunter.io operation might use, providing a comprehensive schema.

---

### **`outputs`: API Output Schema**

```typescript
  outputs: {
    results: { type: 'json', description: 'Search results' },
    emails: { type: 'json', description: 'Email addresses found' },
    email: { type: 'string', description: 'Found email address' },
    score: { type: 'number', description: 'Confidence score' },
    result: { type: 'string', description: 'Verification result' },
    status: { type: 'string', description: 'Status message' },
    total: { type: 'number', description: 'Total results count' },
    personal_emails: { type: 'number', description: 'Personal emails count' },
    generic_emails: { type: 'number', description: 'Generic emails count' },
  },
}
```
*   **`outputs: { ... }`**: This object defines the expected structure and types of the data that will be returned *by* the Hunter.io API (and thus, by this block) after an operation is completed. This is the **data contract** for the block's results.
    *   Each property (`results`, `emails`, `email`, etc.) represents a potential field in the response from Hunter.io.
    *   **`type: 'json'` / `type: 'string'` / `type: 'number'`**: Specifies the data type of the output. `'json'` likely means a complex object or array.
    *   **`description: '...' `**: A brief explanation of what each output field contains.
    *   This section provides a comprehensive schema for all possible data that might be returned, allowing subsequent blocks in a workflow to correctly interpret and utilize the Hunter.io block's results.

---