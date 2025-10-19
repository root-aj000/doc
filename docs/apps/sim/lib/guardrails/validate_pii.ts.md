```typescript
import { spawn } from 'child_process'
import fs from 'fs'
import path from 'path'
import { createLogger } from '@/lib/logs/console/logger'

// Creates a logger instance specifically for this module, tagging logs with "PIIValidator" for easy identification.
const logger = createLogger('PIIValidator')
// Defines a default timeout for PII validation, set to 30 seconds. This prevents the process from running indefinitely.
const DEFAULT_TIMEOUT = 30000 // 30 seconds

// Defines the input structure for PII validation.  This interface outlines the necessary data to perform PII checks.
export interface PIIValidationInput {
  // The text string to be validated for PII.
  text: string
  // An array of entity types to detect (e.g., "PERSON", "EMAIL_ADDRESS").  These should correspond to the types supported by the PII detection tool (e.g., Presidio).
  entityTypes: string[] // e.g., ["PERSON", "EMAIL_ADDRESS", "CREDIT_CARD"]
  // The mode of operation: "block" fails validation if PII is found, while "mask" replaces PII with a masking character and passes validation.
  mode: 'block' | 'mask' // block = fail if PII found, mask = return masked text
  // The language of the text. Defaults to "en" (English). This is important for accurate PII detection.
  language?: string // default: "en"
  // A unique request identifier. Useful for tracing requests through logs.
  requestId: string
}

// Defines the structure of a detected PII entity.  This interface describes the properties of each identified piece of PII.
export interface DetectedPIIEntity {
  // The type of PII detected (e.g., "PERSON", "EMAIL_ADDRESS").
  type: string
  // The starting index of the PII in the input text.
  start: number
  // The ending index of the PII in the input text.
  end: number
  // A confidence score representing the certainty of the PII detection.
  score: number
  // The actual text that was identified as PII.
  text: string
}

// Defines the structure of the PII validation result.  This interface represents the outcome of the validation process.
export interface PIIValidationResult {
  // A boolean indicating whether the validation passed. True if no PII was found (in "block" mode) or if the text was successfully masked (in "mask" mode).
  passed: boolean
  // An optional error message if the validation failed.
  error?: string
  // An array of detected PII entities.
  detectedEntities: DetectedPIIEntity[]
  // An optional masked text string, only present when using "mask" mode.
  maskedText?: string
}

/**
 * Validate text for PII using Microsoft Presidio
 *
 * Supports two modes:
 * - block: Fails validation if any PII is detected
 * - mask: Passes validation and returns masked text with PII replaced
 */
// This is the main function for validating text for PII. It orchestrates the PII detection process.
export async function validatePII(input: PIIValidationInput): Promise<PIIValidationResult> {
  // Destructures the input object for easier access to its properties.
  const { text, entityTypes, mode, language = 'en', requestId } = input

  // Logs the start of the PII validation process, including relevant input parameters.  The request ID is included for traceability.
  logger.info(`[${requestId}] Starting PII validation`, {
    textLength: text.length,
    entityTypes,
    mode,
    language,
  })

  try {
    // Calls the `executePythonPIIDetection` function to perform the actual PII detection using a Python script.
    const result = await executePythonPIIDetection(text, entityTypes, mode, language, requestId)

    // Logs the completion of the PII validation process, including whether it passed, the number of detected entities, and whether masked text was generated.
    logger.info(`[${requestId}] PII validation completed`, {
      passed: result.passed,
      detectedCount: result.detectedEntities.length,
      hasMaskedText: !!result.maskedText,
    })

    // Returns the result of the PII validation.
    return result
  } catch (error: any) {
    // Catches any errors that occur during the PII validation process.
    logger.error(`[${requestId}] PII validation failed`, {
      error: error.message,
    })

    // Returns a failed PIIValidationResult with an error message.
    return {
      passed: false,
      error: `PII validation failed: ${error.message}`,
      detectedEntities: [],
    }
  }
}

/**
 * Execute Python PII detection script
 */
// This function executes the Python script responsible for PII detection. It handles process spawning, input/output, and error handling.
async function executePythonPIIDetection(
  text: string,
  entityTypes: string[],
  mode: string,
  language: string,
  requestId: string
): Promise<PIIValidationResult> {
  // Returns a new Promise to handle the asynchronous execution of the Python script.
  return new Promise((resolve, reject) => {
    // Constructs the path to the guardrails directory.  `process.cwd()` gets the current working directory (project root in Next.js).
    const guardrailsDir = path.join(process.cwd(), 'lib/guardrails')
    // Constructs the full path to the Python script.
    const scriptPath = path.join(guardrailsDir, 'validate_pii.py')
    // Constructs the path to the Python virtual environment's executable.
    const venvPython = path.join(guardrailsDir, 'venv/bin/python3')

    // Determines the Python executable to use.  It prioritizes the virtual environment's Python if it exists, falling back to the system's `python3` if not.
    const pythonCmd = fs.existsSync(venvPython) ? venvPython : 'python3'

    // Spawns the Python process, executing the PII detection script.
    const python = spawn(pythonCmd, [scriptPath])

    // Variables to store the standard output and standard error from the Python script.
    let stdout = ''
    let stderr = ''

    // Sets a timeout for the Python process. If the process takes longer than DEFAULT_TIMEOUT, it's killed, and the promise is rejected.
    const timeout = setTimeout(() => {
      python.kill()
      reject(new Error('PII validation timeout'))
    }, DEFAULT_TIMEOUT)

    // Converts the input data to a JSON string.
    const inputData = JSON.stringify({
      text,
      entityTypes,
      mode,
      language,
    })
    // Writes the JSON data to the Python script's standard input (stdin).
    python.stdin.write(inputData)
    // Signals the end of the input stream to the Python script.
    python.stdin.end()

    // Listens for data on the standard output stream (stdout) of the Python process and appends it to the `stdout` variable.
    python.stdout.on('data', (data) => {
      stdout += data.toString()
    })

    // Listens for data on the standard error stream (stderr) of the Python process and appends it to the `stderr` variable.
    python.stderr.on('data', (data) => {
      stderr += data.toString()
    })

    // Listens for the 'close' event, which is emitted when the Python process exits.
    python.on('close', (code) => {
      // Clears the timeout timer, preventing it from executing after the process has finished.
      clearTimeout(timeout)

      // Checks the exit code of the Python process.  A non-zero code indicates an error.
      if (code !== 0) {
        // Logs the error, including the exit code and standard error output.
        logger.error(`[${requestId}] Python PII detection failed`, {
          code,
          stderr,
        })
        // Resolves the promise with a failed PIIValidationResult, including the error message.
        resolve({
          passed: false,
          error: stderr || 'PII detection failed',
          detectedEntities: [],
        })
        return
      }

      // Parses the result from the standard output.
      try {
        // Defines the prefix that marks the beginning of the JSON result in the Python script's output.
        const prefix = '__SIM_RESULT__='
        // Splits the standard output into lines.
        const lines = stdout.split('\n')
        // Finds the line that starts with the result prefix.
        const marker = lines.find((l) => l.startsWith(prefix))

        // Checks if the result marker was found.
        if (marker) {
          // Extracts the JSON part of the output by removing the prefix.
          const jsonPart = marker.slice(prefix.length)
          // Parses the JSON string into a JavaScript object.
          const result = JSON.parse(jsonPart)
          // Resolves the promise with the parsed PIIValidationResult.
          resolve(result)
        } else {
          // Logs an error if the result marker was not found in the output.
          logger.error(`[${requestId}] No result marker found`, {
            stdout,
            stderr,
            stdoutLines: lines,
          })
          // Resolves the promise with a failed PIIValidationResult, indicating that the result marker was missing.
          resolve({
            passed: false,
            error: `No result marker found in output. stdout: ${stdout.substring(0, 200)}, stderr: ${stderr.substring(0, 200)}`,
            detectedEntities: [],
          })
        }
      } catch (error: any) {
        // Catches any errors that occur during the parsing of the Python script's output.
        logger.error(`[${requestId}] Failed to parse Python result`, {
          error: error.message,
          stdout,
          stderr,
        })
        // Resolves the promise with a failed PIIValidationResult, including the parsing error message.
        resolve({
          passed: false,
          error: `Failed to parse result: ${error.message}. stdout: ${stdout.substring(0, 200)}`,
          detectedEntities: [],
        })
      }
    })

    // Listens for the 'error' event, which is emitted if the Python process fails to spawn or execute.
    python.on('error', (error) => {
      // Clears the timeout timer.
      clearTimeout(timeout)
      // Logs the error, including the error message.
      logger.error(`[${requestId}] Failed to spawn Python process`, {
        error: error.message,
      })
      // Rejects the promise with an error message indicating that the Python process failed to execute.  Provides guidance on ensuring Python and Presidio are installed.
      reject(
        new Error(
          `Failed to execute Python: ${error.message}. Make sure Python 3 and Presidio are installed.`
        )
      )
    })
  })
}

/**
 * List of all supported PII entity types
 * Based on Microsoft Presidio's supported entities
 */
// Defines a constant object containing a list of supported PII entity types.  This object serves as a source of truth for available entity types.
// Each key represents the entity type code (e.g., "CREDIT_CARD"), and the value is a human-readable description.
export const SUPPORTED_PII_ENTITIES = {
  // Common/Global
  CREDIT_CARD: 'Credit card number',
  CRYPTO: 'Cryptocurrency wallet address',
  DATE_TIME: 'Date or time',
  EMAIL_ADDRESS: 'Email address',
  IBAN_CODE: 'International Bank Account Number',
  IP_ADDRESS: 'IP address',
  NRP: 'Nationality, religious or political group',
  LOCATION: 'Location',
  PERSON: 'Person name',
  PHONE_NUMBER: 'Phone number',
  MEDICAL_LICENSE: 'Medical license number',
  URL: 'URL',

  // USA
  US_BANK_NUMBER: 'US bank account number',
  US_DRIVER_LICENSE: 'US driver license',
  US_ITIN: 'US Individual Taxpayer Identification Number',
  US_PASSPORT: 'US passport number',
  US_SSN: 'US Social Security Number',

  // UK
  UK_NHS: 'UK NHS number',
  UK_NINO: 'UK National Insurance Number',

  // Other countries
  ES_NIF: 'Spanish NIF number',
  ES_NIE: 'Spanish NIE number',
  IT_FISCAL_CODE: 'Italian fiscal code',
  IT_DRIVER_LICENSE: 'Italian driver license',
  IT_VAT_CODE: 'Italian VAT code',
  IT_PASSPORT: 'Italian passport',
  IT_IDENTITY_CARD: 'Italian identity card',
  PL_PESEL: 'Polish PESEL number',
  SG_NRIC_FIN: 'Singapore NRIC/FIN',
  SG_UEN: 'Singapore Unique Entity Number',
  AU_ABN: 'Australian Business Number',
  AU_ACN: 'Australian Company Number',
  AU_TFN: 'Australian Tax File Number',
  AU_MEDICARE: 'Australian Medicare number',
  IN_PAN: 'Indian Permanent Account Number',
  IN_AADHAAR: 'Indian Aadhaar number',
  IN_VEHICLE_REGISTRATION: 'Indian vehicle registration',
  IN_VOTER: 'Indian voter ID',
  IN_PASSPORT: 'Indian passport',
  FI_PERSONAL_IDENTITY_CODE: 'Finnish Personal Identity Code',
  KR_RRN: 'Korean Resident Registration Number',
  TH_TNIN: 'Thai National ID Number',
} as const

// Defines a type `PIIEntityType` that represents the keys of the `SUPPORTED_PII_ENTITIES` object.  This type ensures that only valid PII entity types are used.
export type PIIEntityType = keyof typeof SUPPORTED_PII_ENTITIES
```
**Purpose of this file:**

This TypeScript file implements a PII (Personally Identifiable Information) validator. It leverages a Python script (presumably using a library like Microsoft Presidio) to detect and optionally mask PII within a given text. The file defines interfaces for input, output, and detected PII entities, along with functions to orchestrate the validation process, execute the Python script, and handle results. The file also defines constants that represent the list of support PII entities.

**Simplifying Complex Logic:**

1.  **Clear Interface Definitions:** The `PIIValidationInput`, `DetectedPIIEntity`, and `PIIValidationResult` interfaces provide a clear contract for data flow, improving code readability and maintainability.
2.  **Function Decomposition:** The primary logic is separated into two main functions: `validatePII` and `executePythonPIIDetection`. This separation promotes modularity and makes the code easier to understand and test.
3.  **Error Handling:** The `try...catch` block in `validatePII` handles potential errors during the PII validation process, providing a consistent error response.
4.  **Logging:**  The logger enhances debugging and provides insight into the PII validation process.
5.  **Virtual Environment Handling:** The code checks for a Python virtual environment and uses it if available, which ensures a consistent and isolated environment for the PII detection script.
6.  **Asynchronous Operations with Promises:** Using Promises simplifies the asynchronous execution of the Python script and its result handling.

**Line-by-Line Explanation:**

1.  `import { spawn } from 'child_process'`: Imports the `spawn` function from the `child_process` module.  `spawn` is used to execute external commands (in this case, the Python script) as a child process.
2.  `import fs from 'fs'`: Imports the `fs` module for file system operations, specifically to check if the virtual environment exists.
3.  `import path from 'path'`: Imports the `path` module for working with file paths.
4.  `import { createLogger } from '@/lib/logs/console/logger'`: Imports a function `createLogger` for creating logger instances.  The `@/` alias likely refers to the project's root directory.
5.  `const logger = createLogger('PIIValidator')`: Creates a logger instance with the name 'PIIValidator'.  This logger will be used to log information, warnings, and errors related to PII validation.
6.  `const DEFAULT_TIMEOUT = 30000 // 30 seconds`: Defines a constant for the default timeout value in milliseconds (30 seconds).
7.  `export interface PIIValidationInput { ... }`: Defines the `PIIValidationInput` interface, specifying the structure of the input object for the PII validation function.
    *   `text: string`: The text to be validated.
    *   `entityTypes: string[]`: An array of PII entity types to detect.
    *   `mode: 'block' | 'mask'`: The validation mode: 'block' to fail if PII is found, 'mask' to mask the PII.
    *   `language?: string`: The language of the text (optional, defaults to 'en').
    *   `requestId: string`:  A unique identifier for the request.
8.  `export interface DetectedPIIEntity { ... }`: Defines the `DetectedPIIEntity` interface, specifying the structure of a detected PII entity.
    *   `type: string`: The type of PII detected.
    *   `start: number`: The starting index of the PII in the text.
    *   `end: number`: The ending index of the PII in the text.
    *   `score: number`: The confidence score of the detection.
    *   `text: string`: The actual text that was detected as PII.
9.  `export interface PIIValidationResult { ... }`: Defines the `PIIValidationResult` interface, specifying the structure of the result object.
    *   `passed: boolean`:  Indicates whether the validation passed.
    *   `error?: string`: An optional error message if the validation failed.
    *   `detectedEntities: DetectedPIIEntity[]`: An array of detected PII entities.
    *   `maskedText?: string`: The masked text (only present in 'mask' mode).
10. `/** ... */ export async function validatePII(input: PIIValidationInput): Promise<PIIValidationResult> { ... }`: Defines the `validatePII` function, which is the main entry point for PII validation. It takes a `PIIValidationInput` object as input and returns a `Promise` that resolves to a `PIIValidationResult` object.
11. `const { text, entityTypes, mode, language = 'en', requestId } = input`: Destructures the input object to extract its properties.  A default value of 'en' is provided for the `language` property if it's not provided in the input.
12. `logger.info(\`[${requestId}] Starting PII validation\`, { ... })`: Logs an informational message indicating the start of PII validation, including the request ID and input parameters.
13. `try { ... } catch (error: any) { ... }`:  A `try...catch` block is used to handle potential errors during the validation process.
14. `const result = await executePythonPIIDetection(text, entityTypes, mode, language, requestId)`: Calls the `executePythonPIIDetection` function to execute the Python script and awaits the result.
15. `logger.info(\`[${requestId}] PII validation completed\`, { ... })`: Logs an informational message indicating the completion of PII validation, including the request ID and the result of the validation.
16. `return result`: Returns the result of the PII validation.
17. `logger.error(\`[${requestId}] PII validation failed\`, { error: error.message })`: Logs an error message if an error occurs during the validation process, including the request ID and the error message.
18. `return { passed: false, error: \`PII validation failed: ${error.message}\`, detectedEntities: [] }`: Returns a failed `PIIValidationResult` object with an error message and an empty array of detected entities.
19. `async function executePythonPIIDetection( ... ): Promise<PIIValidationResult> { ... }`: Defines the `executePythonPIIDetection` function, which executes the Python script.
20. `return new Promise((resolve, reject) => { ... })`: Returns a new `Promise` to handle the asynchronous execution of the Python script.
21. `const guardrailsDir = path.join(process.cwd(), 'lib/guardrails')`: Constructs the path to the directory containing the Python script and virtual environment, relative to the project root.
22. `const scriptPath = path.join(guardrailsDir, 'validate_pii.py')`: Constructs the full path to the Python script.
23. `const venvPython = path.join(guardrailsDir, 'venv/bin/python3')`: Constructs the path to the Python executable within the virtual environment.
24. `const pythonCmd = fs.existsSync(venvPython) ? venvPython : 'python3'`: Determines the Python executable to use, prioritizing the virtual environment's Python if it exists.
25. `const python = spawn(pythonCmd, [scriptPath])`: Spawns the Python process, executing the script.
26. `let stdout = ''; let stderr = ''`: Initializes variables to store the standard output and standard error from the Python script.
27. `const timeout = setTimeout(() => { ... }, DEFAULT_TIMEOUT)`: Sets a timeout for the Python process. If the process takes longer than `DEFAULT_TIMEOUT`, it's killed.
28. `const inputData = JSON.stringify({ ... })`: Converts the input data to a JSON string.
29. `python.stdin.write(inputData); python.stdin.end()`: Writes the JSON data to the Python script's standard input (stdin) and signals the end of the input stream.
30. `python.stdout.on('data', (data) => { stdout += data.toString() })`: Listens for data on the standard output stream (stdout) and appends it to the `stdout` variable.
31. `python.stderr.on('data', (data) => { stderr += data.toString() })`: Listens for data on the standard error stream (stderr) and appends it to the `stderr` variable.
32. `python.on('close', (code) => { ... })`: Listens for the 'close' event, which is emitted when the Python process exits.
33. `clearTimeout(timeout)`: Clears the timeout timer.
34. `if (code !== 0) { ... }`: Checks the exit code of the Python process. A non-zero code indicates an error.
35. `logger.error(\`[${requestId}] Python PII detection failed\`, { code, stderr })`: Logs an error message if the Python process failed.
36. `resolve({ passed: false, error: stderr || 'PII detection failed', detectedEntities: [] })`: Resolves the promise with a failed `PIIValidationResult` if the Python process failed.
37. `try { ... } catch (error: any) { ... }`: A `try...catch` block is used to handle potential errors during the parsing of the Python script's output.
38. `const prefix = '__SIM_RESULT__='`: Defines the prefix that marks the beginning of the JSON result in the Python script's output.
39. `const lines = stdout.split('\n')`: Splits the standard output into lines.
40. `const marker = lines.find((l) => l.startsWith(prefix))`: Finds the line that starts with the result prefix.
41. `if (marker) { ... } else { ... }`: Checks if the result marker was found.
42. `const jsonPart = marker.slice(prefix.length)`: Extracts the JSON part of the output by removing the prefix.
43. `const result = JSON.parse(jsonPart)`: Parses the JSON string into a JavaScript object.
44. `resolve(result)`: Resolves the promise with the parsed `PIIValidationResult`.
45. `logger.error(\`[${requestId}] No result marker found\`, { stdout, stderr, stdoutLines: lines })`: Logs an error message if the result marker was not found.
46. `resolve({ passed: false, error: \`No result marker found in output. stdout: ${stdout.substring(0, 200)}, stderr: ${stderr.substring(0, 200)}\`, detectedEntities: [] })`: Resolves the promise with a failed `PIIValidationResult` if the result marker was not found.
47. `logger.error(\`[${requestId}] Failed to parse Python result\`, { error: error.message, stdout, stderr })`: Logs an error message if an error occurs during parsing.
48. `resolve({ passed: false, error: \`Failed to parse result: ${error.message}. stdout: ${stdout.substring(0, 200)}\`, detectedEntities: [] })`: Resolves the promise with a failed `PIIValidationResult` if parsing fails.
49. `python.on('error', (error) => { ... })`: Listens for the 'error' event, which is emitted if the Python process fails to spawn or execute.
50. `clearTimeout(timeout)`: Clears the timeout timer.
51. `logger.error(\`[${requestId}] Failed to spawn Python process\`, { error: error.message })`: Logs an error message if the Python process fails to spawn.
52. `reject(new Error(\`Failed to execute Python: ${error.message}. Make sure Python 3 and Presidio are installed.\`))`: Rejects the promise with an error message indicating that the Python process failed to execute.
53. `/** ... */ export const SUPPORTED_PII_ENTITIES = { ... } as const`: Defines a constant object `SUPPORTED_PII_ENTITIES` that lists all supported PII entity types and their descriptions. The `as const` assertion creates a type that is a readonly deep copy of the value assigned to `SUPPORTED_PII_ENTITIES`, which enhances type safety.
54. `export type PIIEntityType = keyof typeof SUPPORTED_PII_ENTITIES`: Defines a type alias `PIIEntityType` that represents the keys of the `SUPPORTED_PII_ENTITIES` object. This ensures that only valid PII entity types are used.

In summary, this file provides a robust and well-structured PII validation solution that uses a separate Python process for detection and provides comprehensive error handling and logging.
