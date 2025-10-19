This TypeScript file defines the configuration for a "Function" block within a workflow automation or AI orchestration platform. Think of it as a blueprint that tells the platform: "Here's how to display, interact with, and execute a custom code snippet as part of a larger automated process."

This block allows users to inject custom JavaScript or Python logic into their workflows, making it highly versatile for tasks that require specific programmatic operations.

---

### Key Concepts & Imports

Before diving into the `FunctionBlock` itself, let's understand the `import` statements at the top:

*   `import { CodeIcon } from '@/components/icons'`: This brings in a pre-defined icon to visually represent the "Function" block in the platform's UI.
*   `import { CodeLanguage, getLanguageDisplayName } from '@/lib/execution/languages'`:
    *   `CodeLanguage`: An enumeration (like a list of named constants) that defines the supported programming languages (e.g., `JavaScript`, `Python`).
    *   `getLanguageDisplayName`: A utility function that takes a `CodeLanguage` value and returns a human-readable name (e.g., `CodeLanguage.JavaScript` becomes "JavaScript").
*   `import type { BlockConfig } from '@/blocks/types'`: This is a TypeScript type definition. It tells us that `FunctionBlock` will conform to the `BlockConfig` structure, which outlines all the properties a workflow block can have (like its name, description, inputs, outputs, etc.).
*   `import type { CodeExecutionOutput } from '@/tools/function/types'`: This type defines the expected shape of the data that the "Function" block will output after it successfully executes its code.

---

### The `FunctionBlock` Constant: A Detailed Blueprint

The core of this file is the `export const FunctionBlock: BlockConfig<CodeExecutionOutput> = { ... }` declaration. This object is the comprehensive configuration for our "Function" block.

Let's break down each property:

*   **`type: 'function'`**
    *   **Purpose:** A unique identifier string for this type of block. The platform uses this to recognize and differentiate it from other block types (e.g., 'text-input', 'api-call').
    *   **Explanation:** When the system sees `type: 'function'`, it knows to load this specific configuration.

*   **`name: 'Function'`**
    *   **Purpose:** The user-friendly name displayed for the block in the workflow editor.
    *   **Explanation:** Users will see "Function" as the title of this block.

*   **`description: 'Run custom logic'`**
    *   **Purpose:** A short, concise summary of what the block does, often shown when hovering over the block in a palette.
    *   **Explanation:** A quick tooltip will tell users this block is for "Run custom logic".

*   **`longDescription: 'This is a core workflow block. Execute custom JavaScript or Python code within your workflow. Use E2B for remote execution with imports or enable Fast Mode (bolt) to run JavaScript locally for lowest latency.'`**
    *   **Purpose:** A more detailed explanation, usually visible in a block's information panel or documentation.
    *   **Explanation:** This clarifies that it's a fundamental block for executing code and highlights two execution modes: "Remote Execution" (for Python or JavaScript with external imports, potentially slower) and "Fast Mode" (for local JavaScript, faster).

*   **`bestPractices: \`...\``**
    *   **Purpose:** Guidelines for users on when and how to effectively use this block, particularly in the context of AI-assisted workflow generation.
    *   **Explanation:** These are instructions, likely for an AI agent constructing workflows, on when to enable remote execution, choose Python vs. JavaScript, and how to reference workflow variables (`<blockName.output>`) within the code. It explicitly warns against using XML/HTML tags in variable references.

*   **`docsLink: 'https://docs.sim.ai/blocks/function'`**
    *   **Purpose:** A URL pointing to more comprehensive documentation for this block.
    *   **Explanation:** Users can click this link to get more in-depth information.

*   **`category: 'blocks'`**
    *   **Purpose:** Helps categorize blocks within the platform's UI, making them easier to find.
    *   **Explanation:** This block will appear under the "blocks" category.

*   **`bgColor: '#FF402F'`**
    *   **Purpose:** The background color for the block's visual representation in the editor.
    *   **Explanation:** The block will have a distinct red-orange color in the UI.

*   **`icon: CodeIcon`**
    *   **Purpose:** The visual icon associated with this block.
    *   **Explanation:** The `CodeIcon` imported earlier will be displayed on the block.

*   **`subBlocks: [...]`**
    *   **Purpose:** This array defines the configurable user interface (UI) elements that appear *inside* the "Function" block when a user interacts with it. These elements allow users to customize how the function behaves.
    *   **Explanation:** This is where we define the switches, dropdowns, and code editors that make up the "Function" block's configuration panel.

    Let's break down each sub-block:

    1.  **Remote Execution Switch:**
        ```typescript
        {
          id: 'remoteExecution',
          type: 'switch',
          layout: 'full',
          title: 'Remote Code Execution',
          description: 'Python/Javascript code run in a sandbox environment. Slower execution times.',
        }
        ```
        *   **`id: 'remoteExecution'`**: A unique identifier for this UI element.
        *   **`type: 'switch'`**: Specifies that this is a toggle switch (on/off).
        *   **`layout: 'full'`**: Dictates that this UI element should span the full width of its container.
        *   **`title: 'Remote Code Execution'`**: The label displayed next to the switch.
        *   **`description: 'Python/Javascript code run in a sandbox environment. Slower execution times.'`**: A tooltip or additional text explaining what this switch does. It highlights that remote execution is sandboxed (for security) and might be slower.

    2.  **Language Dropdown:**
        ```typescript
        {
          id: 'language',
          type: 'dropdown',
          layout: 'full',
          options: [
            { label: getLanguageDisplayName(CodeLanguage.JavaScript), id: CodeLanguage.JavaScript },
            { label: getLanguageDisplayName(CodeLanguage.Python), id: CodeLanguage.Python },
          ],
          placeholder: 'Select language',
          value: () => CodeLanguage.JavaScript,
          condition: {
            field: 'remoteExecution',
            value: true,
          },
        }
        ```
        *   **`id: 'language'`**: Unique identifier.
        *   **`type: 'dropdown'`**: Specifies that this is a selectable list.
        *   **`layout: 'full'`**: Full width.
        *   **`options: [...]`**: The list of choices for the dropdown.
            *   We use `getLanguageDisplayName(CodeLanguage.JavaScript)` to get "JavaScript" as the display label, and `CodeLanguage.JavaScript` as the actual value (`id`) to be stored. The same applies to Python.
        *   **`placeholder: 'Select language'`**: Text shown when no option is selected.
        *   **`value: () => CodeLanguage.JavaScript`**: A function that returns the default selected value, which is JavaScript.
        *   **`condition: { field: 'remoteExecution', value: true }`**: **This is crucial.** This dropdown will *only* appear and be active if the `remoteExecution` switch (defined just above) is turned *on* (`true`). This makes sense: you only need to choose a language if you're using remote execution; otherwise, it defaults to local JavaScript.

    3.  **Code Editor (`code`):**
        ```typescript
        {
          id: 'code',
          type: 'code',
          layout: 'full',
          wandConfig: {
            // ... (AI generation configuration)
          },
        }
        ```
        *   **`id: 'code'`**: Unique identifier.
        *   **`type: 'code'`**: Specifies that this is a code editor UI component.
        *   **`layout: 'full'`**: Full width.
        *   **`wandConfig: { ... }`**: This is a powerful configuration for enabling **AI-assisted code generation** directly within the code editor. This is a complex part, let's break it down:

            *   **`enabled: true`**: Activates the AI code generation feature for this editor.
            *   **`maintainHistory: true`**: The AI model should consider previous turns in the conversation when generating new code, allowing for iterative refinement.
            *   **`prompt: \`...\``**: This is the most important part. It's a detailed set of instructions given to the AI model (often a Large Language Model or LLM) on *how* to generate code for this block. It acts as a system prompt, guiding the AI's output.

                Let's simplify and explain the key parts of this prompt:

                *   **AI's Role**: "You are an expert JavaScript programmer." - Sets the persona for the AI.
                *   **Output Format**: "Generate ONLY the raw body of a JavaScript function... Never wrap in markdown formatting." - The AI should produce *just* the code, no extra text, no ` ```javascript ` or ` ``` ` fences.
                *   **Execution Context**: "The code should be executable within an '`async function(params, environmentVariables) {...}`' context." - This tells the AI what variables will be available to its generated code (`params` for inputs, `environmentVariables` for secrets/config).
                *   **Variable Access Rules (Simplified):**
                    *   **Input Parameters (`params`):** Access directly using **`<paramName>`** (e.g., `<userId>`), *not* `params.paramName`. This is a custom syntax the platform uses for easy referencing of workflow inputs or outputs from previous blocks.
                    *   **Environment Variables (`environmentVariables`):** Access directly using **`{{ENV_VAR_NAME}}`** (e.g., `{{SERVICE_API_KEY}}`), *not* `environmentVariables.VAR_NAME`. This is another custom syntax for securely injecting secrets.
                *   **`Current code context: {context}`**: A placeholder that the system will replace with any existing code in the editor, helping the AI understand what has already been written.
                *   **IMPORTANT FORMATTING RULES**: This section is crucial for ensuring the AI's output is compatible with the platform's execution environment.
                    1.  **Environment Variables**: Emphasizes using `{{VARIABLE_NAME}}` directly, *without quotes*. The system replaces this placeholder *before* execution.
                    2.  **Input Parameters/Workflow Variables**: Emphasizes using `<variable_name>` directly, *without quotes*. Similar to env vars, this is a placeholder.
                    3.  **Function Body ONLY**: No `function()` declarations or surrounding curly braces; just the code that goes *inside* the function.
                    4.  **Imports**: Only standard Node.js built-in modules (like `crypto`, `fs`) are allowed. External libraries are *not* supported, indicating a specific execution environment.
                    5.  **Output**: Reminds the AI to `return` a value if the function is meant to produce output.
                    6.  **Clarity**: Encourages well-written, readable code.
                    7.  **No Explanations**: Reiterates the "raw code only" rule, prohibiting markdown, comments explaining rules, or any other conversational text.
                *   **`Example Scenario` and `Generated Code`**: A perfect illustration for the AI (and a user reading this config) of a user prompt and the *exact* desired code output, demonstrating all the rules in practice. It clearly shows how to use `<block.content>` (a generic placeholder for an input parameter) and `{{SERVICE_API_KEY}}`.

            *   **`placeholder: 'Describe the function you want to create...'`**: Text shown in the empty code editor to guide the user.
            *   **`generationType: 'javascript-function-body'`**: A hint to the AI or system about the type of code expected from the generation.

*   **`tools: { access: ['function_execute'] }`**
    *   **Purpose:** Specifies any external tools or capabilities that this block requires access to.
    *   **Explanation:** This block needs access to a tool named `function_execute`, which is likely the actual engine that takes the code, parameters, and environment variables, and runs the custom function.

*   **`inputs: { ... }`**
    *   **Purpose:** Defines the data inputs that this block expects to receive from the workflow. These are the parameters that the user's code will operate on.
    *   **Explanation:**
        *   **`code: { type: 'string', description: 'JavaScript or Python code to execute' }`**: The actual code string itself, which is the primary input.
        *   **`remoteExecution: { type: 'boolean', description: 'Use E2B remote execution' }`**: A boolean flag corresponding to the `remoteExecution` switch.
        *   **`language: { type: 'string', description: 'Language (javascript or python)' }`**: The selected language from the dropdown.
        *   **`timeout: { type: 'number', description: 'Execution timeout' }`**: An optional number specifying how long the code execution can run before being stopped.

*   **`outputs: { ... }`**
    *   **Purpose:** Defines the data outputs that this block will produce after its execution, making them available to subsequent blocks in the workflow.
    *   **Explanation:**
        *   **`result: { type: 'json', description: 'Return value from the executed JavaScript function' }`**: The main output, typically the value returned by the user's function.
        *   **`stdout: { type: 'string', description: 'Console log output and debug messages from function execution' }`**: Any text printed to the console (e.g., `console.log()` in JavaScript, `print()` in Python) during the function's execution. This is useful for debugging.

---

### Simplified Complex Logic: How it All Connects

1.  **Block Definition:** The `FunctionBlock` constant acts as a comprehensive schema.
2.  **UI Configuration (`subBlocks`):**
    *   Users see a `Remote Code Execution` switch. If they turn it **on**, a `Select language` dropdown appears. This `condition` link is key to keeping the UI relevant.
    *   Below these, there's a code editor.
3.  **AI Assistance (`wandConfig`):**
    *   When a user types a natural language prompt in the code editor, the AI uses the detailed `prompt` to generate the code.
    *   The `prompt` is meticulously crafted to ensure the generated code uses specific placeholders (`<paramName>` and `{{ENV_VAR}}`) that the platform understands. This custom syntax simplifies how users (and the AI) reference data without needing to know the underlying `params` or `environmentVariables` objects.
    *   The AI is instructed to only generate the *body* of the function, avoiding boilerplate.
4.  **Execution (`inputs`, `outputs`, `tools`):**
    *   When the workflow runs, the `Function` block collects its `inputs` (the generated/edited `code`, `remoteExecution` flag, `language`, and `timeout`).
    *   It then calls the `function_execute` tool, passing these inputs.
    *   The `function_execute` tool replaces the custom placeholders (`<paramName>`, `{{ENV_VAR}}`) with actual values from the workflow context and environment.
    *   The code is then executed (either locally for JavaScript or in a remote sandbox).
    *   Finally, the results (`result` and `stdout`) are captured and made available as `outputs` to subsequent blocks in the workflow.

In essence, this file defines a powerful, AI-enhanced "Function" block that lets users easily embed custom code into their automated processes, with careful attention to UI, execution environment, and AI interaction.