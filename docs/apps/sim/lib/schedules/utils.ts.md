Okay, let's break down this TypeScript code file, explaining its purpose, simplifying the logic where possible, and providing a line-by-line explanation.

**Purpose of this File**

This file, `ScheduleUtils.ts`, provides a collection of utility functions related to scheduling tasks. It handles:

1.  **Cron Expression Validation:**  Validates the syntax of cron expressions.
2.  **Cron Expression Generation:**  Creates cron expressions based on different schedule types (minutes, hourly, daily, weekly, monthly).
3.  **Next Run Time Calculation:** Determines the next execution time for a scheduled task, considering timezone and potential DST (Daylight Saving Time) transitions. It uses the `croner` library for robust scheduling.
4.  **Human-Readable Schedule Descriptions:** Converts cron expressions into easy-to-understand text descriptions using the `cronstrue` library.
5.  **Timezone Handling:** It deals with timezones correctly, by creating `Date` object with timezones when necessary.
6.  **Data Extraction:** Safely extracts configurations from starter blocks

In essence, this file encapsulates all the logic required to define, validate, interpret, and calculate schedule times for various task automation scenarios.

**Overall Structure and Key Concepts**

The file is organized as a module with several exported functions and interfaces.  Here's a high-level overview:

*   **Imports:**  Imports necessary libraries: `croner` for cron scheduling, `cronstrue` for cron expression parsing, a logger, and a utility function for date/time formatting.
*   **`validateCronExpression` Function:** Checks if a given cron expression is valid.
*   **`getSubBlockValue` Function:** Extracts a value from a nested data structure (likely representing a UI block configuration).
*   **`parseTimeString` Function:** Parses a time string (e.g., "10:30") into hours and minutes.
*   **`getScheduleTimeValues` Function:** Extracts all the schedule-related configuration values from a given "starter block" (again, likely a UI configuration object).
*   **`createDateWithTimezone` Function:** Creates `Date` object with specified timezone.
*   **`generateCronExpression` Function:**  Generates a cron expression string based on the selected schedule type and associated values.
*   **`calculateNextRunTime` Function:** The core function that calculates the next execution time, taking into account the schedule type, configuration values, a potential last run time, and timezone considerations.
*   **`parseCronToHumanReadable` Function:** Converts a cron expression into a human-readable description.
*   **`getScheduleInfo` Function:** Formats schedule information for display (next run time, last run time, etc.).

**Line-by-Line Explanation**

```typescript
import { Cron } from 'croner'
import cronstrue from 'cronstrue'
import { createLogger } from '@/lib/logs/console/logger'
import { formatDateTime } from '@/lib/utils'

const logger = createLogger('ScheduleUtils')

/**
 * Validates a cron expression and returns validation results
 * @param cronExpression - The cron expression to validate
 * @param timezone - Optional IANA timezone string (e.g., 'America/Los_Angeles'). Defaults to 'UTC'
 * @returns Validation result with isValid flag, error message, and next run date
 */
export function validateCronExpression(
  cronExpression: string,
  timezone?: string
): {
  isValid: boolean
  error?: string
  nextRun?: Date
} {
  if (!cronExpression?.trim()) {
    return {
      isValid: false,
      error: 'Cron expression cannot be empty',
    }
  }

  try {
    // Validate with timezone if provided for accurate next run calculation
    const cron = new Cron(cronExpression, timezone ? { timezone } : undefined)
    const nextRun = cron.nextRun()

    if (!nextRun) {
      return {
        isValid: false,
        error: 'Cron expression produces no future occurrences',
      }
    }

    return {
      isValid: true,
      nextRun,
    }
  } catch (error) {
    return {
      isValid: false,
      error: error instanceof Error ? error.message : 'Invalid cron expression syntax',
    }
  }
}
```

*   `import { Cron } from 'croner'`: Imports the `Cron` class from the `croner` library, which is used for creating and managing cron schedules.
*   `import cronstrue from 'cronstrue'`: Imports the `cronstrue` library, used for converting cron expressions into human-readable strings.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a logger function from a local module (likely for application logging).  The `@/` alias suggests this is within a larger project using path aliases.
*   `import { formatDateTime } from '@/lib/utils'`: Imports a function for formatting dates and times from a local utility module.
*   `const logger = createLogger('ScheduleUtils')`: Creates a logger instance with the name 'ScheduleUtils'.
*   `export function validateCronExpression(...)`: Defines an exported function `validateCronExpression` that takes a cron expression and an optional timezone as input. It returns an object indicating whether the cron expression is valid, any error message, and the next run date.
*   `if (!cronExpression?.trim())`: Checks if the cron expression is empty or contains only whitespace.  The `?.` is the optional chaining operator, preventing errors if `cronExpression` is `null` or `undefined`. `trim()` removes leading/trailing whitespace.
*   `return { isValid: false, error: 'Cron expression cannot be empty' }`: If the cron expression is empty, it returns an object indicating that it is invalid and provides an error message.
*   `try { ... } catch (error) { ... }`: A `try...catch` block is used to handle potential errors during cron expression validation.
*   `const cron = new Cron(cronExpression, timezone ? { timezone } : undefined)`: Creates a new `Cron` object from the `croner` library.  The `timezone ? { timezone } : undefined` part conditionally passes the timezone option to the `Cron` constructor if a timezone is provided.
*   `const nextRun = cron.nextRun()`: Gets the next run date from the `Cron` object.
*   `if (!nextRun) { ... }`: Checks if `nextRun` is null or undefined. A null/undefined nextRun value means that the Cron expression produced no future occurrences.
*   `return { isValid: false, error: 'Cron expression produces no future occurrences' }`:  Returns an error if the cron expression does not produce any future occurrences.
*   `return { isValid: true, nextRun }`: If the cron expression is valid and produces a next run date, it returns an object indicating that it is valid and provides the next run date.
*   `return { isValid: false, error: error instanceof Error ? error.message : 'Invalid cron expression syntax' }`: If an error occurs during cron expression validation, it returns an object indicating that it is invalid and provides an error message. It checks if the caught error is an `Error` instance and extracts its message, otherwise, it provides a generic error message.

```typescript
export interface SubBlockValue {
  value: string
}

export interface BlockState {
  type: string
  subBlocks: Record<string, SubBlockValue | any>
  [key: string]: any
}

export const DAY_MAP: Record<string, number> = {
  MON: 1,
  TUE: 2,
  WED: 3,
  THU: 4,
  FRI: 5,
  SAT: 6,
  SUN: 0,
}

/**
 * Safely extract a value from a block's subBlocks
 */
export function getSubBlockValue(block: BlockState, id: string): string {
  const subBlock = block.subBlocks[id] as SubBlockValue | undefined
  return subBlock?.value || ''
}
```

*   `export interface SubBlockValue { value: string }`: Defines an interface `SubBlockValue` with a single property `value` of type string. This likely represents a simple key-value pair within a configuration block.
*   `export interface BlockState { ... }`: Defines an interface `BlockState`. This is a more complex type, representing a configuration block with:
    *   `type`: A string representing the block type.
    *   `subBlocks`: A record (object) where keys are strings and values are either `SubBlockValue` or `any`. This is where the configuration data is stored.
    *   `[key: string]: any`: An index signature, allowing the `BlockState` object to have arbitrary additional properties.
*   `export const DAY_MAP: Record<string, number> = { ... }`: Defines a constant `DAY_MAP`, which is a record mapping day abbreviations (e.g., "MON") to their corresponding numerical representation (0-6, where 0 is Sunday).  This is used for cron expression generation.
*   `export function getSubBlockValue(block: BlockState, id: string): string { ... }`: Defines an exported function `getSubBlockValue` that takes a `BlockState` and an `id` as input. It retrieves the `value` property from the `subBlocks` object within the `BlockState`, handling potential `null` or `undefined` values gracefully.
*   `const subBlock = block.subBlocks[id] as SubBlockValue | undefined`: Attempts to retrieve the sub-block with the specified ID from the `block.subBlocks`.  The `as SubBlockValue | undefined` is a type assertion, telling TypeScript that the value is either a `SubBlockValue` or `undefined`.
*   `return subBlock?.value || ''`: Returns the `value` property of the `subBlock` if it exists, otherwise, it returns an empty string. The `?.` is the optional chaining operator, preventing errors if `subBlock` is `null` or `undefined`.

```typescript
/**
 * Parse and extract hours and minutes from a time string
 * @param timeString - Time string in format "HH:MM"
 * @returns Array with [hours, minutes] as numbers, or [9, 0] as default
 */
export function parseTimeString(timeString: string | undefined | null): [number, number] {
  if (!timeString || !timeString.includes(':')) {
    return [9, 0] // Default to 9:00 AM
  }

  const [hours, minutes] = timeString.split(':').map(Number)
  return [Number.isNaN(hours) ? 9 : hours, Number.isNaN(minutes) ? 0 : minutes]
}
```

*   `export function parseTimeString(timeString: string | undefined | null): [number, number] { ... }`: Defines an exported function `parseTimeString` that takes a time string (in "HH:MM" format) as input and returns an array containing the hours and minutes as numbers. If the input is invalid, it defaults to 9:00 AM (\[9, 0]).
*   `if (!timeString || !timeString.includes(':')) { ... }`: Checks if the `timeString` is null, undefined, or doesn't contain a colon (:).
*   `return [9, 0] // Default to 9:00 AM`: If the time string is invalid, it returns the default value of \[9, 0].
*   `const [hours, minutes] = timeString.split(':').map(Number)`: Splits the `timeString` by the colon (:) character and converts the resulting substrings to numbers using the `Number` constructor. The `hours` and `minutes` variables are then assigned the corresponding values.
*   `return [Number.isNaN(hours) ? 9 : hours, Number.isNaN(minutes) ? 0 : minutes]`: Returns an array containing the extracted hours and minutes. If either `hours` or `minutes` is not a valid number (NaN), it defaults to 9 for hours and 0 for minutes.

```typescript
/**
 * Get time values from starter block for scheduling
 * @param starterBlock - The starter block containing schedule configuration
 * @returns Object with parsed time values
 */
export function getScheduleTimeValues(starterBlock: BlockState): {
  scheduleTime: string
  scheduleStartAt?: string
  minutesInterval: number
  hourlyMinute: number
  dailyTime: [number, number]
  weeklyDay: number
  weeklyTime: [number, number]
  monthlyDay: number
  monthlyTime: [number, number]
  cronExpression: string | null
  timezone: string
} {
  // Extract schedule time (common field that can override others)
  const scheduleTime = getSubBlockValue(starterBlock, 'scheduleTime')

  // Extract schedule start date
  const scheduleStartAt = getSubBlockValue(starterBlock, 'scheduleStartAt')

  // Extract timezone (default to UTC)
  const timezone = getSubBlockValue(starterBlock, 'timezone') || 'UTC'

  // Get minutes interval (default to 15)
  const minutesIntervalStr = getSubBlockValue(starterBlock, 'minutesInterval')
  const minutesInterval = Number.parseInt(minutesIntervalStr) || 15

  // Get hourly minute (default to 0)
  const hourlyMinuteStr = getSubBlockValue(starterBlock, 'hourlyMinute')
  const hourlyMinute = Number.parseInt(hourlyMinuteStr) || 0

  // Get daily time
  const dailyTime = parseTimeString(getSubBlockValue(starterBlock, 'dailyTime'))

  // Get weekly config
  const weeklyDayStr = getSubBlockValue(starterBlock, 'weeklyDay') || 'MON'
  const weeklyDay = DAY_MAP[weeklyDayStr] || 1
  const weeklyTime = parseTimeString(getSubBlockValue(starterBlock, 'weeklyDayTime'))

  // Get monthly config
  const monthlyDayStr = getSubBlockValue(starterBlock, 'monthlyDay')
  const monthlyDay = Number.parseInt(monthlyDayStr) || 1
  const monthlyTime = parseTimeString(getSubBlockValue(starterBlock, 'monthlyTime'))

  const cronExpression = getSubBlockValue(starterBlock, 'cronExpression') || null

  // Validate cron expression if provided
  if (cronExpression) {
    const validation = validateCronExpression(cronExpression)
    if (!validation.isValid) {
      throw new Error(`Invalid cron expression: ${validation.error}`)
    }
  }

  return {
    scheduleTime,
    scheduleStartAt,
    timezone,
    minutesInterval,
    hourlyMinute,
    dailyTime,
    weeklyDay,
    weeklyTime,
    monthlyDay,
    monthlyTime,
    cronExpression,
  }
}
```

*   `export function getScheduleTimeValues(starterBlock: BlockState): { ... }`: Defines an exported function `getScheduleTimeValues` that takes a `BlockState` object (representing a "starter block" configuration) as input and returns an object containing the extracted and parsed schedule-related values.
*   `const scheduleTime = getSubBlockValue(starterBlock, 'scheduleTime')`: Extracts the value associated with the 'scheduleTime' key from the `starterBlock` using the `getSubBlockValue` function.
*   `const scheduleStartAt = getSubBlockValue(starterBlock, 'scheduleStartAt')`: Extracts the value associated with the 'scheduleStartAt' key from the `starterBlock`.
*   `const timezone = getSubBlockValue(starterBlock, 'timezone') || 'UTC'`: Extracts the value associated with the 'timezone' key from the `starterBlock`. If no timezone is found, defaults to 'UTC'.
*   `const minutesIntervalStr = getSubBlockValue(starterBlock, 'minutesInterval')`: Extracts the value associated with the 'minutesInterval' key from the `starterBlock`.
*   `const minutesInterval = Number.parseInt(minutesIntervalStr) || 15`: Converts the extracted 'minutesIntervalStr' to a number. If the conversion fails or the value is not provided, it defaults to 15.
*   `const hourlyMinuteStr = getSubBlockValue(starterBlock, 'hourlyMinute')`: Extracts the value associated with the 'hourlyMinute' key from the `starterBlock`.
*   `const hourlyMinute = Number.parseInt(hourlyMinuteStr) || 0`: Converts the extracted 'hourlyMinuteStr' to a number. If the conversion fails or the value is not provided, it defaults to 0.
*   `const dailyTime = parseTimeString(getSubBlockValue(starterBlock, 'dailyTime'))`: Extracts the value associated with the 'dailyTime' key and parses it using the `parseTimeString` function.
*   `const weeklyDayStr = getSubBlockValue(starterBlock, 'weeklyDay') || 'MON'`: Extracts the value associated with the 'weeklyDay' key from the `starterBlock`. If no value is found, defaults to 'MON'.
*   `const weeklyDay = DAY_MAP[weeklyDayStr] || 1`: Uses the `DAY_MAP` to convert the `weeklyDayStr` to its numerical representation. If the conversion fails (e.g., the `weeklyDayStr` is not a valid day abbreviation), it defaults to 1 (Monday).
*   `const weeklyTime = parseTimeString(getSubBlockValue(starterBlock, 'weeklyDayTime'))`: Extracts the value associated with the 'weeklyDayTime' key and parses it using the `parseTimeString` function.
*   `const monthlyDayStr = getSubBlockValue(starterBlock, 'monthlyDay')`: Extracts the value associated with the 'monthlyDay' key from the `starterBlock`.
*   `const monthlyDay = Number.parseInt(monthlyDayStr) || 1`: Converts the extracted 'monthlyDayStr' to a number. If the conversion fails or the value is not provided, it defaults to 1.
*   `const monthlyTime = parseTimeString(getSubBlockValue(starterBlock, 'monthlyTime'))`: Extracts the value associated with the 'monthlyTime' key and parses it using the `parseTimeString` function.
*   `const cronExpression = getSubBlockValue(starterBlock, 'cronExpression') || null`: Extracts the value associated with the 'cronExpression' key from the `starterBlock`. If no value is found, defaults to `null`.
*   `if (cronExpression) { ... }`: Checks if a `cronExpression` is provided.
*   `const validation = validateCronExpression(cronExpression)`: Validates the cron expression using the `validateCronExpression` function.
*   `if (!validation.isValid) { throw new Error(\`Invalid cron expression: ${validation.error}\`) }`: If the cron expression is invalid, it throws an error with the error message from the validation result.
*   `return { ... }`: Returns an object containing all the extracted and parsed schedule values.

```typescript
/**
 * Helper function to create a date with the specified time in the correct timezone.
 * This function calculates the corresponding UTC time for a given local date,
 * local time, and IANA timezone name, correctly handling DST.
 *
 * @param dateInput Date string or Date object representing the local date.
 * @param timeStr Time string in format "HH:MM" representing the local time.
 * @param timezone IANA timezone string (e.g., 'America/Los_Angeles', 'Europe/Paris'). Defaults to 'UTC'.
 * @returns Date object representing the absolute point in time (UTC).
 */
export function createDateWithTimezone(
  dateInput: string | Date,
  timeStr: string,
  timezone = 'UTC'
): Date {
  try {
    // 1. Parse the base date and target time
    const baseDate = typeof dateInput === 'string' ? new Date(dateInput) : new Date(dateInput)
    const [targetHours, targetMinutes] = parseTimeString(timeStr)

    // Ensure baseDate reflects the date part only, setting time to 00:00:00 in UTC
    // This prevents potential issues if dateInput string includes time/timezone info.
    const year = baseDate.getUTCFullYear()
    const monthIndex = baseDate.getUTCMonth() // 0-based
    const day = baseDate.getUTCDate()

    // 2. Create a tentative UTC Date object using the target date and time components
    // This assumes, for a moment, that the target H:M were meant for UTC.
    const tentativeUTCDate = new Date(
      Date.UTC(year, monthIndex, day, targetHours, targetMinutes, 0)
    )

    // 3. If the target timezone is UTC, we're done.
    if (timezone === 'UTC') {
      return tentativeUTCDate
    }

    // 4. Format the tentative UTC date into the target timezone's local time components.
    // Use 'en-CA' locale for unambiguous YYYY-MM-DD and 24-hour format.
    const formatter = new Intl.DateTimeFormat('en-CA', {
      timeZone: timezone,
      year: 'numeric',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit', // Use 2-digit for consistency
      minute: '2-digit',
      second: '2-digit',
      hourCycle: 'h23', // Use 24-hour format (00-23)
    })

    const parts = formatter.formatToParts(tentativeUTCDate)
    const getPart = (type: Intl.DateTimeFormatPartTypes) =>
      parts.find((p) => p.type === type)?.value

    const formattedYear = Number.parseInt(getPart('year') || '0', 10)
    const formattedMonth = Number.parseInt(getPart('month') || '0', 10) // 1-based
    const formattedDay = Number.parseInt(getPart('day') || '0', 10)
    const formattedHour = Number.parseInt(getPart('hour') || '0', 10)
    const formattedMinute = Number.parseInt(getPart('minute') || '0', 10)

    // Create a Date object representing the local time *in the target timezone*
    // when the tentative UTC date occurs.
    // Note: month needs to be adjusted back to 0-based for Date.UTC()
    const actualLocalTimeInTargetZone = Date.UTC(
      formattedYear,
      formattedMonth - 1,
      formattedDay,
      formattedHour,
      formattedMinute,
      0 // seconds
    )

    // 5. Calculate the difference between the intended local time and the actual local time
    // that resulted from the tentative UTC date. This difference represents the offset
    // needed to adjust the UTC time.
    // Create the intended local time as a UTC timestamp for comparison purposes.
    const intendedLocalTimeAsUTC = Date.UTC(year, monthIndex, day, targetHours, targetMinutes, 0)

    // The offset needed for UTC time is the difference between the intended local time
    // and the actual local time (when both are represented as UTC timestamps).
    const offsetMilliseconds = intendedLocalTimeAsUTC - actualLocalTimeInTargetZone

    // 6. Adjust the tentative UTC date by the calculated offset.
    const finalUTCTimeMilliseconds = tentativeUTCDate.getTime() + offsetMilliseconds
    const finalDate = new Date(finalUTCTimeMilliseconds)

    return finalDate
  } catch (e) {
    logger.error('Error creating date with timezone:', e, { dateInput, timeStr, timezone })
    // Fallback to a simple UTC interpretation on error
    try {
      const baseDate = typeof dateInput === 'string' ? new Date(dateInput) : new Date(dateInput)
      const [hours, minutes] = parseTimeString(timeStr)
      const year = baseDate.getUTCFullYear()
      const monthIndex = baseDate.getUTCMonth()
      const day = baseDate.getUTCDate()
      return new Date(Date.UTC(year, monthIndex, day, hours, minutes, 0))
    } catch (fallbackError) {
      logger.error('Error during fallback date creation:', fallbackError)
      throw new Error(
        `Failed to create date with timezone (${timezone}): ${fallbackError instanceof Error ? fallbackError.message : String(fallbackError)}`
      )
    }
  }
}
```

*   `export function createDateWithTimezone(...)`: This function is the heart of timezone handling. It takes a date (string or `Date` object), a time string ("HH:MM"), and a timezone string as input. Its goal is to produce a `Date` object representing *the precise moment in time* corresponding to the given local date and time in the specified timezone.
*   `const baseDate = typeof dateInput === 'string' ? new Date(dateInput) : new Date(dateInput)`: Creates `Date` object based on date input.
*   `const [targetHours, targetMinutes] = parseTimeString(timeStr)`: Parses the time string into hours and minutes.
*   `const year = baseDate.getUTCFullYear(); ... const day = baseDate.getUTCDate()`:  Extracts the year, month, and day from the base date, ensuring that they are treated as UTC values. This is important to avoid timezone-related issues during calculations.
*   `const tentativeUTCDate = new Date(Date.UTC(year, monthIndex, day, targetHours, targetMinutes, 0))`: Constructs a "tentative" UTC date using the extracted year, month, day, hours, and minutes. This assumes, *for the moment*, that the provided hours and minutes were intended as UTC values.
*   `if (timezone === 'UTC') { return tentativeUTCDate }`: If the target timezone is UTC, the tentative UTC date is the final result, so it's returned directly.
*   `const formatter = new Intl.DateTimeFormat('en-CA', { ... })`: Creates an `Intl.DateTimeFormat` object, which is used to format dates and times according to specific locales and timezones. The `'en-CA'` locale is chosen for its unambiguous YYYY-MM-DD date format and 24-hour time format. The `timeZone` option is set to the target timezone.
*   `const parts = formatter.formatToParts(tentativeUTCDate)`: Formats the tentative UTC date into its individual components (year, month, day, hour, minute, etc.) using the `formatToParts` method of the `Intl.DateTimeFormat` object.
*   `const getPart = (type: Intl.DateTimeFormatPartTypes) => parts.find(p => p.type === type)?.value`: This defines a helper function to extract the value of a specific part from the formatted date parts.
*   `const formattedYear = Number.parseInt(getPart('year') || '0', 10); ... const formattedMinute = Number.parseInt(getPart('minute') || '0', 10)`: Extracts the year, month, day, hour, and minute from the formatted date parts and converts them to numbers.
*   `const actualLocalTimeInTargetZone = Date.UTC(formattedYear, formattedMonth - 1, formattedDay, formattedHour, formattedMinute, 0)`: Creates a `Date` object representing the local time in the target timezone when the tentative UTC date occurs.  Crucially, this is still a UTC timestamp, but it represents the *equivalent* local time in the target timezone.
*   `const intendedLocalTimeAsUTC = Date.UTC(year, monthIndex, day, targetHours, targetMinutes, 0)`: Creates a `Date` object representing the *intended* local time as a UTC timestamp. This is the desired local time that we want to achieve.
*   `const offsetMilliseconds = intendedLocalTimeAsUTC - actualLocalTimeInTargetZone`: Calculates the difference (in milliseconds) between the intended local time and the actual local time in the target timezone. This difference represents the offset that needs to be applied to the tentative UTC date to get the correct UTC representation of the desired local time.
*   `const finalUTCTimeMilliseconds = tentativeUTCDate.getTime() + offsetMilliseconds`: Applies the calculated offset to the tentative UTC date to get the final UTC time in milliseconds.
*   `const finalDate = new Date(finalUTCTimeMilliseconds)`: Creates a new `Date` object from the final UTC time in milliseconds. This `Date` object represents the absolute point in time (UTC) that corresponds to the given local date and time in the specified timezone.
*   `try { ... } catch (e) { ... }`: A `try...catch` block is used to handle potential errors during date and time conversion. If an error occurs, the code logs the error and attempts to use a fallback mechanism.
*   `const baseDate = typeof dateInput === 'string' ? new Date(dateInput) : new Date(dateInput)`: Within the `catch` block, the code attempts a fallback mechanism by recreating the `Date` object using a simpler approach.
*   `const [hours, minutes] = parseTimeString(timeStr)`: Parses the time string into hours and minutes.
*   `const year = baseDate.getUTCFullYear(); ... const day = baseDate.getUTCDate()`: Extracts the year, month, and day from the base date.
*   `return new Date(Date.UTC(year, monthIndex, day, hours, minutes, 0))`: Creates a new `Date` object using the extracted year, month, day, hours, and minutes, assuming they are UTC values. This is a simpler approach that may not handle timezones and DST transitions correctly, but it serves as a fallback in case the primary conversion method fails.
*   `throw new Error(\`Failed to create date with timezone (${timezone}): ${error instanceof Error ? error.message : String(error)}\`)`: If the fallback mechanism also fails, the code throws an error with a message indicating that the date creation failed and includes the timezone and error message.

```typescript
/**
 * Generate cron expression based on schedule type and values
 *
 * IMPORTANT: The generated cron expressions use local time values (hours/minutes)
 * from the user's configured timezone. When used with Croner, pass the timezone
 * option to ensure proper scheduling:
 *
 * Example:
 *   const cronExpr = generateCronExpression('daily', { dailyTime: [14, 30], timezone: 'America/Los_Angeles' })
 *   const cron = new Cron(cronExpr, { timezone: 'America/Los_Angeles' })
 *
 * This will schedule the job at 2:30 PM Pacific Time, which Croner will correctly
 * convert to the appropriate UTC time, handling DST transitions automatically.
 *
 * @param scheduleType - Type of schedule (minutes, hourly, daily, weekly, monthly, custom)
 * @param scheduleValues - Object containing schedule configuration including timezone
 * @returns Cron expression string representing the schedule in local time
 */
export function generateCronExpression(
  scheduleType: string,
  scheduleValues: ReturnType<typeof getScheduleTimeValues>
): string {
  switch (scheduleType) {
    case 'minutes':
      return `*/${scheduleValues.minutesInterval} * * * *`

    case 'hourly':
      return `${scheduleValues.hourlyMinute} * * * *`

    case 'daily': {
      const [hours, minutes] = scheduleValues.dailyTime
      return `${minutes} ${hours} * * *`
    }

    case 'weekly': {
      const [hours, minutes] = scheduleValues.weeklyTime
      return `${minutes} ${hours} * * ${scheduleValues.weeklyDay}`
    }

    case 'monthly': {
      const [hours, minutes] = scheduleValues.monthlyTime
      return `${minutes} ${hours} ${scheduleValues.monthlyDay} * *`
    }

    case 'custom': {
      if (!scheduleValues.cronExpression?.trim()) {
        throw new Error('Custom schedule requires a valid cron expression')
      }
      return scheduleValues.cronExpression
    }

    default:
      throw new Error(`Unsupported schedule type: ${scheduleType}`)
  }
}
```

*   `export function generateCronExpression(...)`: Defines an exported function `generateCronExpression` that takes a schedule type and schedule values as input and returns a cron expression string.
*   `switch (scheduleType) { ... }`: A `switch` statement is used to generate the cron expression based on the schedule type.
*   `case 'minutes'`: If the schedule type is 'minutes', it generates a cron expression that runs every `minutesInterval` minutes.
*   `case 'hourly'`: If the schedule type is 'hourly', it generates a cron expression that runs every hour at the specified `hourlyMinute`.
*   `case 'daily'`: If the schedule type is 'daily', it generates a cron expression that runs every day at the specified `dailyTime` (hours and minutes).
*   `case 'weekly'`: If the schedule type is 'weekly', it generates a cron expression that runs every week on the specified `weeklyDay` (day of the week) at the specified `weeklyTime` (hours and minutes).
*   `case 'monthly'`: If the schedule type is 'monthly', it generates a cron expression that runs every month on the specified `monthlyDay` (day of the month) at the specified `monthlyTime` (hours and minutes).
*   `case 'custom'`: If the schedule type is 'custom', it uses the provided `cronExpression` as the cron expression.
*   `if (!scheduleValues.cronExpression?.trim()) { throw new Error('Custom schedule requires a valid cron expression') }`: If the schedule type is 'custom' and no cron expression is provided, it throws an error.
*   `default`: If the schedule type is not supported, it throws an error.

```typescript
/**
 * Calculate the next run time based on schedule configuration
 * Uses Croner library with timezone support for accurate scheduling across timezones and DST transitions
 * @param scheduleType - Type of schedule (minutes, hourly, daily, etc)
 * @param scheduleValues - Object with schedule configuration values
 * @param lastRanAt - Optional last execution time
 * @returns Date object for next execution time
 */
export function calculateNextRunTime(
  scheduleType: string,
  scheduleValues: ReturnType<typeof getScheduleTimeValues>,
  lastRanAt?: Date | null
): Date {
  // Get timezone (default to UTC)
  const timezone = scheduleValues.timezone || 'UTC'

  // Get the current time
  const baseDate = new Date()

  // If we have both a start date and time, use them together with timezone awareness
  if (scheduleValues.scheduleStartAt && scheduleValues.scheduleTime) {
    try {
      logger.info(
        `Creating date with: startAt=${scheduleValues.scheduleStartAt}, time=${scheduleValues.scheduleTime}, timezone=${timezone}`
      )

      const combinedDate = createDateWithTimezone(
        scheduleValues.scheduleStartAt,
        scheduleValues.scheduleTime,
        timezone
      )

      logger.info(`Combined date result: ${combinedDate.toISOString()}`)

      // If the combined date is in the future, use it as our next run time
      if (combinedDate > baseDate) {
        return combinedDate
      }
    } catch (e) {
      logger.error('Error combining scheduled date and time:', e)
    }
  }
  // If only scheduleStartAt is set (without scheduleTime), parse it directly
  else if (scheduleValues.scheduleStartAt) {
    try {
      // Check if the date string already includes time information
      const startAtStr = scheduleValues.scheduleStartAt
      const hasTimeComponent =
        startAtStr.includes('T') && (startAtStr.includes(':') || startAtStr.includes('.'))

      if (hasTimeComponent) {
        // If the string already has time info, parse it directly but with timezone awareness
        const startDate = new Date(startAtStr)

        // If it's a UTC ISO string (ends with Z), use it directly
        if (startAtStr.endsWith('Z') && timezone === 'UTC') {
          if (startDate > baseDate) {
            return startDate
          }
        } else {
          // For non-UTC dates or when