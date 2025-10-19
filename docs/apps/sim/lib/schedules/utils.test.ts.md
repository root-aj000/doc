```typescript
/**
 * Tests for schedule utility functions
 */
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest'
import {
  type BlockState,
  calculateNextRunTime,
  createDateWithTimezone,
  generateCronExpression,
  getScheduleTimeValues,
  getSubBlockValue,
  parseCronToHumanReadable,
  parseTimeString,
  validateCronExpression,
} from '@/lib/schedules/utils'

describe('Schedule Utilities', () => {
  // This block groups tests related to the 'parseTimeString' utility function.
  describe('parseTimeString', () => {
    // This test case checks if the function correctly parses valid time strings in the format "HH:MM".
    it.concurrent('should parse valid time strings', () => {
      expect(parseTimeString('09:30')).toEqual([9, 30]) // Expects the function to return [9, 30] for "09:30".
      expect(parseTimeString('23:45')).toEqual([23, 45]) // Expects the function to return [23, 45] for "23:45".
      expect(parseTimeString('00:00')).toEqual([0, 0]) // Expects the function to return [0, 0] for "00:00".
    })

    // This test case checks if the function returns default values [9, 0] when given invalid inputs such as empty strings, null, undefined, or other invalid strings.
    it.concurrent('should return default values for invalid inputs', () => {
      expect(parseTimeString('')).toEqual([9, 0]) // Empty string should return default [9, 0].
      expect(parseTimeString(null)).toEqual([9, 0]) // Null should return default [9, 0].
      expect(parseTimeString(undefined)).toEqual([9, 0]) // Undefined should return default [9, 0].
      expect(parseTimeString('invalid')).toEqual([9, 0]) // Any invalid string should return default [9, 0].
    })

    // This test case checks if the function can handle malformed time strings, gracefully parsing them or providing reasonable outputs.
    it.concurrent('should handle malformed time strings', () => {
      expect(parseTimeString('9:30')).toEqual([9, 30]) // Missing leading zero for the hour.
      expect(parseTimeString('9:3')).toEqual([9, 3]) // Missing leading zero for the hour and minutes.
      expect(parseTimeString('9:')).toEqual([9, 0]) // Only hour is provided.
      expect(parseTimeString(':30')).toEqual([0, 30]) // Only minutes is provided.
    })

    // This test case checks if the function handles out-of-range time values (hours > 23, minutes > 59) without throwing errors, simply returning the provided out-of-range values.
    it.concurrent('should handle out-of-range time values', () => {
      expect(parseTimeString('25:30')).toEqual([25, 30]) // Hours greater than 23.
      expect(parseTimeString('10:75')).toEqual([10, 75]) // Minutes greater than 59.
      expect(parseTimeString('99:99')).toEqual([99, 99]) // Both hours and minutes out of range.
    })
  })

  // This block groups tests related to the 'getSubBlockValue' utility function.
  describe('getSubBlockValue', () => {
    // This test case checks if the function correctly retrieves values from the 'subBlocks' property of a 'BlockState' object.
    it.concurrent('should get values from block subBlocks', () => {
      // Defines a mock BlockState object with various subBlocks and their values.
      const block: BlockState = {
        type: 'starter',
        subBlocks: {
          scheduleType: { value: 'daily' }, // subBlock with string value
          scheduleTime: { value: '09:30' }, // subBlock with string value
          emptyValue: { value: '' }, // subBlock with empty string value
          nullValue: { value: null }, // subBlock with null value
        },
      } as BlockState

      expect(getSubBlockValue(block, 'scheduleType')).toBe('daily') // Retrieves the value of 'scheduleType'.
      expect(getSubBlockValue(block, 'scheduleTime')).toBe('09:30') // Retrieves the value of 'scheduleTime'.
      expect(getSubBlockValue(block, 'emptyValue')).toBe('') // Retrieves the value of 'emptyValue'.
      expect(getSubBlockValue(block, 'nullValue')).toBe('') // Retrieves the value of 'nullValue'.  Null is converted to empty string
      expect(getSubBlockValue(block, 'nonExistent')).toBe('') // Tries to retrieve a non-existent subBlock.  Defaults to empty string
    })

    // This test case checks if the function handles cases where the 'subBlocks' property is missing or empty, returning a default empty string.
    it.concurrent('should handle missing subBlocks', () => {
      const block = {
        type: 'starter',
        subBlocks: {}, // Empty subBlocks
      } as BlockState

      expect(getSubBlockValue(block, 'anyField')).toBe('') // Should return an empty string.
    })
  })

  // This block groups tests related to the 'getScheduleTimeValues' utility function.
  describe('getScheduleTimeValues', () => {
    // This test case checks if the function correctly extracts all relevant time values from a BlockState object.
    it.concurrent('should extract all time values from a block', () => {
      // Defines a mock BlockState with all possible scheduling fields.
      const block: BlockState = {
        type: 'starter',
        subBlocks: {
          scheduleTime: { value: '09:30' },
          minutesInterval: { value: '15' },
          hourlyMinute: { value: '45' },
          dailyTime: { value: '10:15' },
          weeklyDay: { value: 'MON' },
          weeklyDayTime: { value: '12:00' },
          monthlyDay: { value: '15' },
          monthlyTime: { value: '14:30' },
          scheduleStartAt: { value: '' },
          timezone: { value: 'UTC' },
        },
      } as BlockState

      const result = getScheduleTimeValues(block) // Calls the function being tested.

      //  Asserts that the function returns an object with all values extracted and transformed as expected.
      expect(result).toEqual({
        scheduleTime: '09:30',
        scheduleStartAt: '',
        timezone: 'UTC',
        minutesInterval: 15,
        hourlyMinute: 45,
        dailyTime: [10, 15],
        weeklyDay: 1, // MON = 1
        weeklyTime: [12, 0],
        monthlyDay: 15,
        monthlyTime: [14, 30],
        cronExpression: null,
      })
    })

    // This test case checks if the function uses default values when certain scheduling fields are missing from the BlockState.
    it.concurrent('should use default values for missing fields', () => {
      const block: BlockState = {
        type: 'starter',
        subBlocks: {
          // Minimal config
          scheduleType: { value: 'daily' },
        },
      } as BlockState

      const result = getScheduleTimeValues(block)

      expect(result).toEqual({
        scheduleTime: '',
        scheduleStartAt: '',
        timezone: 'UTC',
        minutesInterval: 15, // Default
        hourlyMinute: 0, // Default
        dailyTime: [9, 0], // Default
        weeklyDay: 1, // Default (MON)
        weeklyTime: [9, 0], // Default
        monthlyDay: 1, // Default
        monthlyTime: [9, 0], // Default
        cronExpression: null,
      })
    })
  })

  // This block groups tests related to the 'generateCronExpression' utility function.
  describe('generateCronExpression', () => {
    // This test case checks if the function correctly generates Cron expressions for different scheduling types (minutes, hourly, daily, weekly, monthly).
    it.concurrent('should generate correct cron expressions for different schedule types', () => {
      // Defines a set of schedule values to be used in the Cron expression generation.
      const scheduleValues = {
        scheduleTime: '09:30',
        minutesInterval: 15,
        hourlyMinute: 45,
        dailyTime: [10, 15] as [number, number],
        weeklyDay: 1, // Monday
        weeklyTime: [12, 0] as [number, number],
        monthlyDay: 15,
        monthlyTime: [14, 30] as [number, number],
        timezone: 'UTC',
        cronExpression: null,
      }

      // Minutes (every 15 minutes)
      expect(generateCronExpression('minutes', scheduleValues)).toBe('*/15 * * * *')

      // Hourly (at minute 45)
      expect(generateCronExpression('hourly', scheduleValues)).toBe('45 * * * *')

      // Daily (at 10:15)
      expect(generateCronExpression('daily', scheduleValues)).toBe('15 10 * * *')

      // Weekly (Monday at 12:00)
      expect(generateCronExpression('weekly', scheduleValues)).toBe('0 12 * * 1')

      // Monthly (15th at 14:30)
      expect(generateCronExpression('monthly', scheduleValues)).toBe('30 14 15 * *')
    })

    // This test case checks if the function correctly handles custom cron expressions.
    it.concurrent('should handle custom cron expressions', () => {
      // For this simplified test, let's skip the complex mocking
      // and just verify the 'custom' case is in the switch statement

      // Create a mock block with custom cron expression
      const mockBlock: BlockState = {
        type: 'starter',
        subBlocks: {
          cronExpression: { value: '*/5 * * * *' },
        },
      }

      // Create schedule values with the block as any since we're testing a special case
      const scheduleValues = {
        ...getScheduleTimeValues(mockBlock),
        // Override as BlockState to access the cronExpression
        // This simulates what happens in the actual code
        subBlocks: mockBlock.subBlocks,
      } as any

      // Now properly test the custom case
      const result = generateCronExpression('custom', scheduleValues)
      expect(result).toBe('*/5 * * * *')

      // Also verify other schedule types still work
      const standardScheduleValues = {
        scheduleTime: '',
        minutesInterval: 15,
        hourlyMinute: 30,
        dailyTime: [9, 0] as [number, number],
        weeklyDay: 1,
        weeklyTime: [10, 0] as [number, number],
        monthlyDay: 15,
        monthlyTime: [14, 30] as [number, number],
        timezone: 'UTC',
        cronExpression: null,
      }

      expect(generateCronExpression('minutes', standardScheduleValues)).toBe('*/15 * * * *')
    })

    // This test case checks if the function throws an error for invalid schedule types.
    it.concurrent('should throw for invalid schedule types', () => {
      const scheduleValues = {} as any
      expect(() => generateCronExpression('invalid-type', scheduleValues)).toThrow()
    })
  })

  // This block groups tests related to the 'calculateNextRunTime' utility function.
  describe('calculateNextRunTime', () => {
    // This beforeEach hook mocks the current date and time using `vi.useFakeTimers()` and `vi.setSystemTime()` to ensure consistent test results.
    beforeEach(() => {
      // Mock Date.now for consistent testing
      vi.useFakeTimers()
      vi.setSystemTime(new Date('2025-04-12T12:00:00.000Z')) // Noon on April 12, 2025
    })

    // This afterEach hook restores the real timers using `vi.useRealTimers()` after each test.
    afterEach(() => {
      vi.useRealTimers()
    })

    // This test case checks if the function correctly calculates the next run time for a "minutes" schedule type.
    it.concurrent('should calculate next run for minutes schedule using Croner', () => {
      // Defines a set of schedule values for a "minutes" schedule.
      const scheduleValues = {
        scheduleTime: '',
        scheduleStartAt: '',
        timezone: 'UTC',
        minutesInterval: 15,
        hourlyMinute: 0,
        dailyTime: [9, 0] as [number, number],
        weeklyDay: 1,
        weeklyTime: [9, 0] as [number, number],
        monthlyDay: 1,
        monthlyTime: [9, 0] as [number, number],
        cronExpression: null,
      }

      const nextRun = calculateNextRunTime('minutes', scheduleValues)

      // Just check that it's a valid date in the future
      expect(nextRun instanceof Date).toBe(true)
      expect(nextRun > new Date()).toBe(true)

      // Croner will calculate based on the cron expression */15 * * * *
      // The exact minute depends on Croner's calculation
    })

    it.concurrent('should handle scheduleStartAt with scheduleTime', () => {
      const scheduleValues = {
        scheduleTime: '14:30', // Specific start time
        scheduleStartAt: '2025-04-15', // Future date
        timezone: 'UTC',
        minutesInterval: 15,
        hourlyMinute: 0,
        dailyTime: [9, 0] as [number, number],
        weeklyDay: 1,
        weeklyTime: [9, 0] as [number, number],
        monthlyDay: 1,
        monthlyTime: [9, 0] as [number, number],
        cronExpression: null,
      }

      const nextRun = calculateNextRunTime('minutes', scheduleValues)

      // Should return the future start date with time
      expect(nextRun.getFullYear()).toBe(2025)
      expect(nextRun.getMonth()).toBe(3) // April
      expect(nextRun.getDate()).toBe(15)
    })

    // This test case checks if the function correctly calculates the next run time for an "hourly" schedule type.
    it.concurrent('should calculate next run for hourly schedule using Croner', () => {
      // Defines a set of schedule values for an "hourly" schedule.
      const scheduleValues = {
        scheduleTime: '',
        scheduleStartAt: '',
        timezone: 'UTC',
        minutesInterval: 15,
        hourlyMinute: 30,
        dailyTime: [9, 0] as [number, number],
        weeklyDay: 1,
        weeklyTime: [9, 0] as [number, number],
        monthlyDay: 1,
        monthlyTime: [9, 0] as [number, number],
        cronExpression: null,
      }

      const nextRun = calculateNextRunTime('hourly', scheduleValues)

      // Verify it's a valid future date using Croner's calculation
      expect(nextRun instanceof Date).toBe(true)
      expect(nextRun > new Date()).toBe(true)
      // Croner calculates based on cron "30 * * * *"
      expect(nextRun.getMinutes()).toBe(30)
    })

    // This test case checks if the function correctly calculates the next run time for a "daily" schedule type, taking into account the specified timezone.
    it.concurrent('should calculate next run for daily schedule using Croner with timezone', () => {
      // Defines a set of schedule values for a "daily" schedule with a specific timezone.
      const scheduleValues = {
        scheduleTime: '',
        scheduleStartAt: '',
        timezone: 'UTC',
        minutesInterval: 15,
        hourlyMinute: 0,
        dailyTime: [9, 0] as [number, number],
        weeklyDay: 1,
        weeklyTime: [9, 0] as [number, number],
        monthlyDay: 1,
        monthlyTime: [9, 0] as [number, number],
        cronExpression: null,
      }

      const nextRun = calculateNextRunTime('daily', scheduleValues)

      // Verify it's a future date at exactly 9:00 UTC using Croner
      expect(nextRun instanceof Date).toBe(true)
      expect(nextRun > new Date()).toBe(true)
      expect(nextRun.getUTCHours()).toBe(9)
      expect(nextRun.getUTCMinutes()).toBe(0)
    })

    // This test case checks if the function correctly calculates the next run time for a "weekly" schedule type, taking into account the specified timezone and day of the week.
    it.concurrent(
      'should calculate next run for weekly schedule using Croner with timezone',
      () => {
        const scheduleValues = {
          scheduleTime: '',
          scheduleStartAt: '',
          timezone: 'UTC',
          minutesInterval: 15,
          hourlyMinute: 0,
          dailyTime: [9, 0] as [number, number],
          weeklyDay: 1, // Monday
          weeklyTime: [10, 0] as [number, number],
          monthlyDay: 1,
          monthlyTime: [9, 0] as [number, number],
          cronExpression: null,
        }

        const nextRun = calculateNextRunTime('weekly', scheduleValues)

        // Should be next Monday at 10:00 AM UTC using Croner
        expect(nextRun.getUTCDay()).toBe(1) // Monday
        expect(nextRun.getUTCHours()).toBe(10)
        expect(nextRun.getUTCMinutes()).toBe(0)
      }
    )

    // This test case checks if the function correctly calculates the next run time for a "monthly" schedule type, taking into account the specified timezone and day of the month.
    it.concurrent(
      'should calculate next run for monthly schedule using Croner with timezone',
      () => {
        const scheduleValues = {
          scheduleTime: '',
          scheduleStartAt: '',
          timezone: 'UTC',
          minutesInterval: 15,
          hourlyMinute: 0,
          dailyTime: [9, 0] as [number, number],
          weeklyDay: 1,
          weeklyTime: [9, 0] as [number, number],
          monthlyDay: 15,
          monthlyTime: [14, 30] as [number, number],
          cronExpression: null,
        }

        const nextRun = calculateNextRunTime('monthly', scheduleValues)

        // Current date is 2025-04-12 12:00, so next run should be 2025-04-15 14:30 UTC using Croner
        expect(nextRun.getFullYear()).toBe(2025)
        expect(nextRun.getUTCMonth()).toBe(3) // April (0-indexed)
        expect(nextRun.getUTCDate()).toBe(15)
        expect(nextRun.getUTCHours()).toBe(14)
        expect(nextRun.getUTCMinutes()).toBe(30)
      }
    )

    // This test case checks if the function correctly handles the 'lastRanAt' parameter, although the Croner library calculates independently.
    it.concurrent(
      'should work with lastRanAt parameter (though Croner calculates independently)',
      () => {
        const scheduleValues = {
          scheduleTime: '',
          scheduleStartAt: '',
          timezone: 'UTC',
          minutesInterval: 15,
          hourlyMinute: 0,
          dailyTime: [9, 0] as [number, number],
          weeklyDay: 1,
          weeklyTime: [9, 0] as [number, number],
          monthlyDay: 1,
          monthlyTime: [9, 0] as [number, number],
          cronExpression: null,
        }

        // Last ran 10 minutes ago
        const lastRanAt = new Date()
        lastRanAt.setMinutes(lastRanAt.getMinutes() - 10)

        const nextRun = calculateNextRunTime('minutes', scheduleValues, lastRanAt)

        // With Croner, it calculates based on cron expression, not lastRanAt
        // Just verify we get a future date
        expect(nextRun instanceof Date).toBe(true)
        expect(nextRun > new Date()).toBe(true)
      }
    )

    // This test case checks if the function respects a future 'scheduleStartAt' date when calculating the next run time.
    it.concurrent('should respect future scheduleStartAt date', () => {
      const scheduleValues = {
        scheduleStartAt: '2025-04-22T20:50:00.000Z', // April 22, 2025 at 8:50 PM
        scheduleTime: '',
        timezone: 'UTC',
        minutesInterval: 10,
        hourlyMinute: 0,
        dailyTime: [9, 0] as [number, number],
        weeklyDay: 1,
        weeklyTime: [9, 0] as [number, number],
        monthlyDay: 1,
        monthlyTime: [9, 0] as [number, number],
        cronExpression: null,
      }

      const nextRun = calculateNextRunTime('minutes', scheduleValues)

      // Should be exactly April 22, 2025 at 8:50 PM (the future start date)
      expect(nextRun.toISOString()).toBe('2025-04-22T20:50:00.000Z')
    })

    // This test case checks if the function ignores a past 'scheduleStartAt' date and calculates the next run time based on the current date.
    it.concurrent('should ignore past scheduleStartAt date', () => {
      const scheduleValues = {
        scheduleStartAt: '2025-04-10T20:50:00.000Z', // April 10, 2025 at 8:50 PM (in the past)
        scheduleTime: '',
        timezone: 'UTC',
        minutesInterval: 10,
        hourlyMinute: 0,
        dailyTime: [9, 0] as [number, number],
        weeklyDay: 1,
        weeklyTime: [9, 0] as [number, number],
        monthlyDay: 1,
        monthlyTime: [9, 0] as [number, number],
        cronExpression: null,
      }

      const nextRun = calculateNextRunTime('minutes', scheduleValues)

      // Should not use the past date but calculate normally
      expect(nextRun > new Date()).toBe(true)
      expect(nextRun.getMinutes() % 10).toBe(0) // Should align with the interval
    })
  })

  // This block groups tests related to the 'validateCronExpression' utility function.
  describe('validateCronExpression', () => {
    // This test case checks if the function validates correct Cron expressions, ensuring that they are considered valid and return a 'nextRun' date.
    it.concurrent('should validate correct cron expressions', () => {
      expect(validateCronExpression('0 9 * * *')).toEqual({
        isValid: true,
        nextRun: expect.any(Date),
      })
      expect(validateCronExpression('*/15 * * * *')).toEqual({
        isValid: true,
        nextRun: expect.any(Date),
      })
      expect(validateCronExpression('30 14 15 * *')).toEqual({
        isValid: true,
        nextRun: expect.any(Date),
      })
    })

    // This test case checks if the function validates Cron expressions with a timezone, ensuring that they are considered valid and return a 'nextRun' date.
    it.concurrent('should validate cron expressions with timezone', () => {
      const result = validateCronExpression('0 9 * * *', 'America/Los_Angeles')
      expect(result.isValid).toBe(true)
      expect(result.nextRun).toBeInstanceOf(Date)
    })

    // This test case checks if the function rejects invalid Cron expressions, ensuring that they are considered invalid and return an error message.
    it.concurrent('should reject invalid cron expressions', () => {
      expect(validateCronExpression('invalid')).toEqual({
        isValid: false,
        error: expect.stringContaining('invalid'),
      })
      expect(validateCronExpression('60 * * * *')).toEqual({
        isValid: false,
        error: expect.any(String),
      })
      expect(validateCronExpression('')).toEqual({
        isValid: false,
        error: 'Cron expression cannot be empty',
      })
      expect(validateCronExpression('   ')).toEqual({
        isValid: false,
        error: 'Cron expression cannot be empty',
      })
    })

    // This test case checks if the function detects impossible Cron expressions, such as those that produce no future occurrences.
    it.concurrent('should detect impossible cron expressions', () => {
      // This would be February 31st - impossible date
      expect(validateCronExpression('0 0 31 2 *')).toEqual({
        isValid: false,
        error: 'Cron expression produces no future occurrences',
      })
    })
  })

  // This block groups tests related to the 'parseCronToHumanReadable' utility function.
  describe('parseCronToHumanReadable', () => {
    // This test case checks if the function parses common Cron patterns and converts them into human-readable descriptions using the 'cronstrue' library.
    it.concurrent('should parse common cron patterns using cronstrue', () => {
      // cronstrue produces "Every minute" for '* * * * *'
      expect(parseCronToHumanReadable('* * * * *')).toBe('Every minute')

      // cronstrue produces "Every 15 minutes" for '*/15 * * * *'
      expect(parseCronToHumanReadable('*/15 * * * *')).toBe('Every 15 minutes')

      // cronstrue produces "At 30 minutes past the hour" for '30 * * * *'
      expect(parseCronToHumanReadable('30 * * * *')).toContain('30 minutes past the hour')

      // cronstrue produces "At 09:00 AM" for '0 9 * * *'
      expect(parseCronToHumanReadable('0 9 * * *')).toContain('09:00 AM')

      // cronstrue produces "At 02:30 PM" for '30 14 * * *'
      expect(parseCronToHumanReadable('30 14 * * *')).toContain('02:30 PM')

      // cronstrue produces "At 09:00 AM, only on Monday" for '0 9 * * 1'
      expect(parseCronToHumanReadable('0 9 * * 1')).toContain('Monday')

      // cronstrue produces "At 02:30 PM, on day 15 of the month" for '30 14 15 * *'
      expect(parseCronToHumanReadable('30 14 15 * *')).toContain('15')
    })

    // This test case checks if the function includes timezone information in the human-readable description when a timezone is provided.
    it.concurrent('should include timezone information when provided', () => {
      const resultPT = parseCronToHumanReadable('0 9 * * *', 'America/Los_Angeles')
      expect(resultPT).toContain('(PT)')
      expect(resultPT).toContain('09:00 AM')

      const resultET = parseCronToHumanReadable('30 14 * * *', 'America/New_York')
      expect(resultET).toContain('(ET)')
      expect(resultET).toContain('02:30 PM')

      const resultUTC = parseCronToHumanReadable('0 12 * * *', 'UTC')
      expect(resultUTC).not.toContain('(UTC)') // UTC should not be explicitly shown
    })

    // This test case checks if the function handles complex Cron patterns using the 'cronstrue' library.
    it.concurrent('should handle complex patterns with cronstrue', () => {
      // cronstrue can handle complex patterns better than our custom parser
      const result1 = parseCronToHumanReadable('0 9 * * 1-5')
      expect(result1).toContain('Monday through Friday')

      const result2 = parseCronToHumanReadable('0 9 1,15 * *')
      expect(result2).toContain('day 1 and 15')
    })

    // This test case checks if the function returns a fallback description for invalid Cron patterns.
    it.concurrent('should return a fallback for invalid patterns', () => {
      const result = parseCronToHumanReadable('invalid cron')
      // Should fallback to "Schedule: <expression>"
      expect(result).toContain('Schedule:')
      expect(result).toContain('invalid cron')
    })
  })

  // This block groups tests related to timezone-aware scheduling with Croner.
  describe('Timezone-aware scheduling with Croner', () => {
    // This test case checks if the function correctly calculates a daily schedule in Pacific Time, taking into account Daylight Saving Time (DST).
    it.concurrent('should calculate daily schedule in Pacific Time correctly', () => {
      const scheduleValues = {
        scheduleTime: '',
        scheduleStartAt: '',
        timezone: 'America/Los_Angeles',
        minutesInterval: 15,
        hourlyMinute: 0,
        dailyTime: [9, 0] as [number, number], // 9 AM Pacific
        weeklyDay: 1,
        weeklyTime: [9, 0] as [number, number],
        monthlyDay: 1,
        monthlyTime: [9, 0] as [number, number],
        cronExpression: null,
      }

      const nextRun = calculateNextRunTime('daily', scheduleValues)

      // 9 AM Pacific should be 16:00 or 17:00 UTC depending on DST
      // Croner handles this automatically
      expect(nextRun instanceof Date).toBe(true)
      expect(nextRun > new Date()).toBe(true)
    })

    // This test case checks if the function correctly handles DST transitions for schedules.
    it.concurrent('should handle DST transition for schedules', () => {
      // Set a date during DST transition in March
      vi.useFakeTimers()
      vi.setSystemTime(new Date('2025-03-08T10:00:00.000Z')) // Before DST

      const scheduleValues = {
        scheduleTime: '',
        scheduleStartAt: '',
        timezone: 'America/New_York',
        minutesInterval: 15,
        hourlyMinute: 0,
        dailyTime: [14, 0] as [number, number], // 2 PM Eastern
        weeklyDay: 1,
        weeklyTime: [14, 0] as [number, number],
        monthlyDay: 1,
        monthlyTime: [14, 0] as [number, number],
        cronExpression: null,
      }

      const nextRun = calculateNextRunTime('daily', scheduleValues)

      // Croner should handle DST transition correctly
      expect(nextRun instanceof Date).toBe(true)
      expect(nextRun > new Date()).toBe(true)

      vi.useRealTimers()
    })

    // This test case checks if the function correctly calculates a weekly schedule in the Tokyo timezone.
    it.concurrent('should calculate weekly schedule in Tokyo timezone', () => {
      const scheduleValues = {
        scheduleTime: '',
        scheduleStartAt: '',
        timezone: 'Asia/Tokyo',
        minutesInterval: 15,
        hourlyMinute: 0,
        dailyTime: [9, 0] as [number, number],
        weeklyDay: 1, // Monday
        weeklyTime: [10, 0] as [number, number], // 10 AM Japan Time
        monthlyDay: 1,
        monthlyTime: [9, 0] as [number, number],
        cronExpression: null,
      }

      const nextRun = calculateNextRunTime('weekly', scheduleValues)

      // Verify it's a valid future date
      // Tokyo is UTC+9, so 10 AM JST = 1 AM UTC
      expect(nextRun instanceof Date).toBe(true)
      expect(nextRun > new Date()).toBe(true)
    })

    // This test case checks if the function correctly handles a custom Cron expression with a timezone.
    it.concurrent('should handle custom cron with timezone', () => {
      const scheduleValues = {
        scheduleTime: '',
        scheduleStartAt: '',
        timezone: 'Europe/London',
        minutesInterval: 15,
        hourlyMinute: 0,
        dailyTime: [9, 0] as [number, number],
        weeklyDay: 1,
        weeklyTime: [9, 0] as [number, number],
        monthlyDay: 1,
        monthlyTime: [9, 0] as [number, number],
        cronExpression: '30 15 * * *', // 3:30 PM London time
      }

      const nextRun = calculateNextRunTime('custom', scheduleValues)

      // Verify it's a valid future date
      expect(nextRun instanceof Date).toBe(true)
      expect(nextRun > new Date()).toBe(true)
    })

    // This test case checks if the function correctly handles a monthly schedule on the last day of the month.
    it.concurrent('should handle monthly schedule on last day of month', () => {
      vi.useFakeTimers()
      vi.setSystemTime(new Date('2025-02-15T12:00:00.000Z'))

      const scheduleValues = {
        scheduleTime: '',
        scheduleStartAt: '',
        timezone: 'America/Chicago',
        minutesInterval: 15,
        hourlyMinute: 0,
        dailyTime: [9, 0] as [number, number],
        weeklyDay: 1,
        weeklyTime: [9