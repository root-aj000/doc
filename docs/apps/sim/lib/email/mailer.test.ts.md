```typescript
import { beforeEach, describe, expect, it, type Mock, vi } from 'vitest'

// Mock functions for Resend's send and batch send methods
const mockSend = vi.fn()
const mockBatchSend = vi.fn()

// Mock functions for Azure Communication Email Service's beginSend and pollUntilDone methods
const mockAzureBeginSend = vi.fn()
const mockAzurePollUntilDone = vi.fn()

// Mock the 'resend' module
vi.mock('resend', () => {
  return {
    Resend: vi.fn().mockImplementation(() => ({
      emails: {
        send: (...args: any[]) => mockSend(...args),
      },
      batch: {
        send: (...args: any[]) => mockBatchSend(...args),
      },
    })),
  }
})

// Mock the '@azure/communication-email' module
vi.mock('@azure/communication-email', () => {
  return {
    EmailClient: vi.fn().mockImplementation(() => ({
      beginSend: (...args: any[]) => mockAzureBeginSend(...args),
    })),
  }
})

// Mock the '@/lib/email/unsubscribe' module
vi.mock('@/lib/email/unsubscribe', () => ({
  isUnsubscribed: vi.fn(),
  generateUnsubscribeToken: vi.fn(),
}))

// Mock the '@/lib/env' module
vi.mock('@/lib/env', () => ({
  env: {
    RESEND_API_KEY: 'test-api-key',
    AZURE_ACS_CONNECTION_STRING: 'test-azure-connection-string',
    AZURE_COMMUNICATION_EMAIL_DOMAIN: 'test.azurecomm.net',
    NEXT_PUBLIC_APP_URL: 'https://test.sim.ai',
    FROM_EMAIL_ADDRESS: 'Sim <noreply@sim.ai>',
  },
}))

// Mock the '@/lib/urls/utils' module
vi.mock('@/lib/urls/utils', () => ({
  getEmailDomain: vi.fn().mockReturnValue('sim.ai'),
  getBaseUrl: vi.fn().mockReturnValue('https://test.sim.ai'),
}))

import { type EmailType, sendBatchEmails, sendEmail } from '@/lib/email/mailer'
import { generateUnsubscribeToken, isUnsubscribed } from '@/lib/email/unsubscribe'

describe('mailer', () => {
  const testEmailOptions = {
    to: 'test@example.com',
    subject: 'Test Subject',
    html: '<p>Test email content</p>',
  }

  beforeEach(() => {
    vi.clearAllMocks()
    ;(isUnsubscribed as Mock).mockResolvedValue(false)
    ;(generateUnsubscribeToken as Mock).mockReturnValue('mock-token-123')

    // Mock successful Resend response
    mockSend.mockResolvedValue({
      data: { id: 'test-email-id' },
      error: null,
    })

    mockBatchSend.mockResolvedValue({
      data: [{ id: 'batch-email-1' }, { id: 'batch-email-2' }],
      error: null,
    })

    // Mock successful Azure response
    mockAzurePollUntilDone.mockResolvedValue({
      status: 'Succeeded',
      id: 'azure-email-id',
    })

    mockAzureBeginSend.mockReturnValue({
      pollUntilDone: mockAzurePollUntilDone,
    })
  })

  describe('sendEmail', () => {
    it('should send a transactional email successfully', async () => {
      const result = await sendEmail({
        ...testEmailOptions,
        emailType: 'transactional',
      })

      expect(result.success).toBe(true)
      expect(result.message).toBe('Email sent successfully via Resend')
      expect(result.data).toEqual({ id: 'test-email-id' })

      // Should not check unsubscribe status for transactional emails
      expect(isUnsubscribed).not.toHaveBeenCalled()

      // Should call Resend with correct parameters
      expect(mockSend).toHaveBeenCalledWith({
        from: 'Sim <noreply@sim.ai>',
        to: testEmailOptions.to,
        subject: testEmailOptions.subject,
        html: testEmailOptions.html,
        headers: undefined, // No unsubscribe headers for transactional
      })
    })

    it('should send a marketing email with unsubscribe headers', async () => {
      const htmlWithToken = '<p>Test content</p><a href="{{UNSUBSCRIBE_TOKEN}}">Unsubscribe</a>'

      const result = await sendEmail({
        ...testEmailOptions,
        html: htmlWithToken,
        emailType: 'marketing',
      })

      expect(result.success).toBe(true)

      // Should check unsubscribe status
      expect(isUnsubscribed).toHaveBeenCalledWith(testEmailOptions.to, 'marketing')

      // Should generate unsubscribe token
      expect(generateUnsubscribeToken).toHaveBeenCalledWith(testEmailOptions.to, 'marketing')

      // Should call Resend with unsubscribe headers
      expect(mockSend).toHaveBeenCalledWith({
        from: 'Sim <noreply@sim.ai>',
        to: testEmailOptions.to,
        subject: testEmailOptions.subject,
        html: '<p>Test content</p><a href="mock-token-123">Unsubscribe</a>',
        headers: {
          'List-Unsubscribe':
            '<https://test.sim.ai/unsubscribe?token=mock-token-123&email=test%40example.com>',
          'List-Unsubscribe-Post': 'List-Unsubscribe=One-Click',
        },
      })
    })

    it('should skip sending if user has unsubscribed', async () => {
      ;(isUnsubscribed as Mock).mockResolvedValue(true)

      const result = await sendEmail({
        ...testEmailOptions,
        emailType: 'marketing',
      })

      expect(result.success).toBe(true)
      expect(result.message).toBe('Email skipped (user unsubscribed)')
      expect(result.data).toEqual({ id: 'skipped-unsubscribed' })

      // Should not call Resend
      expect(mockSend).not.toHaveBeenCalled()
    })

    it.concurrent('should handle Resend API errors and fallback to Azure', async () => {
      // Mock Resend to fail
      mockSend.mockResolvedValue({
        data: null,
        error: { message: 'API rate limit exceeded' },
      })

      const result = await sendEmail(testEmailOptions)

      expect(result.success).toBe(true)
      expect(result.message).toBe('Email sent successfully via Azure Communication Services')
      expect(result.data).toEqual({ id: 'azure-email-id' })

      // Should have tried Resend first
      expect(mockSend).toHaveBeenCalled()

      // Should have fallen back to Azure
      expect(mockAzureBeginSend).toHaveBeenCalled()
    })

    it.concurrent('should handle unexpected errors and fallback to Azure', async () => {
      // Mock Resend to throw an error
      mockSend.mockRejectedValue(new Error('Network error'))

      const result = await sendEmail(testEmailOptions)

      expect(result.success).toBe(true)
      expect(result.message).toBe('Email sent successfully via Azure Communication Services')
      expect(result.data).toEqual({ id: 'azure-email-id' })

      // Should have tried Resend first
      expect(mockSend).toHaveBeenCalled()

      // Should have fallen back to Azure
      expect(mockAzureBeginSend).toHaveBeenCalled()
    })

    it.concurrent('should use custom from address when provided', async () => {
      await sendEmail({
        ...testEmailOptions,
        from: 'custom@example.com',
      })

      expect(mockSend).toHaveBeenCalledWith(
        expect.objectContaining({
          from: 'custom@example.com',
        })
      )
    })

    it('should not include unsubscribe when includeUnsubscribe is false', async () => {
      await sendEmail({
        ...testEmailOptions,
        emailType: 'marketing',
        includeUnsubscribe: false,
      })

      expect(generateUnsubscribeToken).not.toHaveBeenCalled()
      expect(mockSend).toHaveBeenCalledWith(
        expect.objectContaining({
          headers: undefined,
        })
      )
    })

    it.concurrent('should replace unsubscribe token placeholders in HTML', async () => {
      const htmlWithPlaceholder = '<p>Content</p><a href="{{UNSUBSCRIBE_TOKEN}}">Unsubscribe</a>'

      await sendEmail({
        ...testEmailOptions,
        html: htmlWithPlaceholder,
        emailType: 'updates' as EmailType,
      })

      expect(mockSend).toHaveBeenCalledWith(
        expect.objectContaining({
          html: '<p>Content</p><a href="mock-token-123">Unsubscribe</a>',
        })
      )
    })
  })

  describe('Azure Communication Services fallback', () => {
    it('should fallback to Azure when Resend fails', async () => {
      // Mock Resend to fail
      mockSend.mockRejectedValue(new Error('Resend service unavailable'))

      const result = await sendEmail({
        ...testEmailOptions,
        emailType: 'transactional',
      })

      expect(result.success).toBe(true)
      expect(result.message).toBe('Email sent successfully via Azure Communication Services')
      expect(result.data).toEqual({ id: 'azure-email-id' })

      // Should have tried Resend first
      expect(mockSend).toHaveBeenCalled()

      // Should have fallen back to Azure
      expect(mockAzureBeginSend).toHaveBeenCalledWith({
        senderAddress: 'noreply@sim.ai',
        content: {
          subject: testEmailOptions.subject,
          html: testEmailOptions.html,
        },
        recipients: {
          to: [{ address: testEmailOptions.to }],
        },
        headers: {},
      })
    })

    it('should handle Azure Communication Services failure', async () => {
      // Mock both services to fail
      mockSend.mockRejectedValue(new Error('Resend service unavailable'))
      mockAzurePollUntilDone.mockResolvedValue({
        status: 'Failed',
        id: 'failed-id',
      })

      const result = await sendEmail({
        ...testEmailOptions,
        emailType: 'transactional',
      })

      expect(result.success).toBe(false)
      expect(result.message).toBe('Both Resend and Azure Communication Services failed')

      // Should have tried both services
      expect(mockSend).toHaveBeenCalled()
      expect(mockAzureBeginSend).toHaveBeenCalled()
    })
  })

  describe('sendBatchEmails', () => {
    const testBatchEmails = [
      { ...testEmailOptions, to: 'user1@example.com' },
      { ...testEmailOptions, to: 'user2@example.com' },
    ]

    it('should send batch emails via Resend successfully', async () => {
      const result = await sendBatchEmails({ emails: testBatchEmails })

      expect(result.success).toBe(true)
      expect(result.message).toBe('All batch emails sent successfully via Resend')
      expect(result.results).toHaveLength(2)
      expect(mockBatchSend).toHaveBeenCalled()
    })

    it('should fallback to individual sends when Resend batch fails', async () => {
      // Mock Resend batch to fail
      mockBatchSend.mockRejectedValue(new Error('Batch service unavailable'))

      const result = await sendBatchEmails({ emails: testBatchEmails })

      expect(result.success).toBe(true)
      expect(result.message).toBe('All batch emails sent successfully')
      expect(result.results).toHaveLength(2)

      // Should have tried Resend batch first
      expect(mockBatchSend).toHaveBeenCalled()

      // Should have fallen back to individual sends (which will use Resend since it's available)
      expect(mockSend).toHaveBeenCalledTimes(2)
    })

    it('should handle mixed success/failure in individual fallback', async () => {
      // Mock Resend batch to fail
      mockBatchSend.mockRejectedValue(new Error('Batch service unavailable'))

      // Mock first individual send to succeed, second to fail and Azure also fails
      mockSend
        .mockResolvedValueOnce({
          data: { id: 'email-1' },
          error: null,
        })
        .mockRejectedValueOnce(new Error('Individual send failure'))

      // Mock Azure to fail for the second email (first call succeeds, but second fails)
      mockAzurePollUntilDone.mockResolvedValue({
        status: 'Failed',
        id: 'failed-id',
      })

      const result = await sendBatchEmails({ emails: testBatchEmails })

      expect(result.success).toBe(false)
      expect(result.message).toBe('1/2 emails sent successfully')
      expect(result.results).toHaveLength(2)
      expect(result.results[0].success).toBe(true)
      expect(result.results[1].success).toBe(false)
    })
  })
})
```

### Purpose of this file

This file contains unit tests for the `mailer` module, specifically the `sendEmail` and `sendBatchEmails` functions. It uses `vitest` as the testing framework and mocks external dependencies like `resend`, `@azure/communication-email`, and internal modules like `unsubscribe` and `env` to isolate the mailer logic.

The tests cover various scenarios, including:

- Successfully sending transactional and marketing emails using Resend.
- Handling user unsubscribes and skipping email sending.
- Implementing a fallback mechanism to Azure Communication Services when Resend fails.
- Sending batch emails and handling potential failures, including falling back to individual sends.
- Substituting unsubscribe token placeholders in email templates.

In summary, the purpose is to ensure the reliability and correctness of email sending functionality, covering different email types, success and failure scenarios, and the fallback mechanism.

### Simplified Logic

The code uses mocking extensively to control the behavior of external dependencies and isolate the `sendEmail` and `sendBatchEmails` functions.  This approach allows for focused testing without relying on actual external services. Key simplifications are achieved by:

1.  **Mocking External Services:** The `resend` and `@azure/communication-email` modules are fully mocked, allowing the tests to simulate success or failure scenarios without actually sending emails.
2.  **Mocking Configuration:** The `env` module is mocked to provide consistent test configuration values.
3.  **Clear Setup and Teardown:** The `beforeEach` block ensures that mocks are reset before each test, providing a clean slate.
4.  **Focused Assertions:** Each test focuses on a specific aspect of the mailer logic, making the assertions clear and easy to understand.

### Code Explanation Line by Line

```typescript
import { beforeEach, describe, expect, it, type Mock, vi } from 'vitest'
```

*   **Import statements:** Imports necessary functions and types from the `vitest` testing framework.
    *   `beforeEach`: A hook that runs before each test in a `describe` block.
    *   `describe`:  Groups related tests together.
    *   `expect`: Used for making assertions in tests.
    *   `it`: Defines an individual test case.
    *   `type Mock`:  A TypeScript type for mock functions.
    *   `vi`:  The `vitest` object providing mocking functionalities like `vi.fn()` and `vi.mock()`.

```typescript
const mockSend = vi.fn()
const mockBatchSend = vi.fn()
const mockAzureBeginSend = vi.fn()
const mockAzurePollUntilDone = vi.fn()
```

*   **Mock function declarations:**  Creates mock functions using `vi.fn()`. These mocks will replace the actual functions from the Resend and Azure email services during testing.
    *   `mockSend`:  Mocks the `send` method of the Resend `emails` object.
    *   `mockBatchSend`: Mocks the `send` method of the Resend `batch` object.
    *   `mockAzureBeginSend`: Mocks the `beginSend` method of the Azure `EmailClient`.
    *   `mockAzurePollUntilDone`: Mocks the `pollUntilDone` method returned by `beginSend` in Azure.

```typescript
vi.mock('resend', () => {
  return {
    Resend: vi.fn().mockImplementation(() => ({
      emails: {
        send: (...args: any[]) => mockSend(...args),
      },
      batch: {
        send: (...args: any[]) => mockBatchSend(...args),
      },
    })),
  }
})
```

*   **Mocking the `resend` module:**  This line uses `vi.mock()` to replace the actual `resend` module with a mock implementation.
    *   The callback function returns an object that mimics the structure of the `resend` module.
    *   `Resend`: A mock class constructor which returns an object with `emails` and `batch` properties.
    *   `mockImplementation`:  Defines the behavior of the mock `Resend` class.  It returns an object with `emails` and `batch` properties, each having a `send` method that calls the corresponding mock function (`mockSend` or `mockBatchSend`).
    *   `(...args: any[]) => mockSend(...args)`: This creates a function that accepts any arguments and passes them to the `mockSend` function.  This allows the tests to inspect the arguments passed to the `send` method.

```typescript
vi.mock('@azure/communication-email', () => {
  return {
    EmailClient: vi.fn().mockImplementation(() => ({
      beginSend: (...args: any[]) => mockAzureBeginSend(...args),
    })),
  }
})
```

*   **Mocking the `@azure/communication-email` module:** Similar to the `resend` mock, this replaces the Azure email module with a mock.
    *   `EmailClient`: A mock class constructor that returns an object with a `beginSend` method.
    *   `mockImplementation`: Defines the behavior of the mock `EmailClient` class.  It returns an object with a `beginSend` method that calls the `mockAzureBeginSend` function.

```typescript
vi.mock('@/lib/email/unsubscribe', () => ({
  isUnsubscribed: vi.fn(),
  generateUnsubscribeToken: vi.fn(),
}))
```

*   **Mocking the `@/lib/email/unsubscribe` module:** Mocks the functions related to unsubscribe functionality.
    *   `isUnsubscribed`:  A mock function to simulate whether a user is unsubscribed.
    *   `generateUnsubscribeToken`:  A mock function to generate an unsubscribe token.

```typescript
vi.mock('@/lib/env', () => ({
  env: {
    RESEND_API_KEY: 'test-api-key',
    AZURE_ACS_CONNECTION_STRING: 'test-azure-connection-string',
    AZURE_COMMUNICATION_EMAIL_DOMAIN: 'test.azurecomm.net',
    NEXT_PUBLIC_APP_URL: 'https://test.sim.ai',
    FROM_EMAIL_ADDRESS: 'Sim <noreply@sim.ai>',
  },
}))
```

*   **Mocking the `@/lib/env` module:** Mocks the environment variables used by the mailer.  This ensures that the tests run with consistent configuration.
    *   Returns an object with an `env` property, containing mock environment variables.

```typescript
vi.mock('@/lib/urls/utils', () => ({
  getEmailDomain: vi.fn().mockReturnValue('sim.ai'),
  getBaseUrl: vi.fn().mockReturnValue('https://test.sim.ai'),
}))
```

*   **Mocking the `@/lib/urls/utils` module:** Mocks utility functions related to URLs.
    *   `getEmailDomain`: Mocks the function to return the email domain.
    *   `getBaseUrl`: Mocks the function to return the base URL of the application.

```typescript
import { type EmailType, sendBatchEmails, sendEmail } from '@/lib/email/mailer'
import { generateUnsubscribeToken, isUnsubscribed } from '@/lib/email/unsubscribe'
```

*   **Importing the mailer functions:** Imports the functions being tested (`sendEmail` and `sendBatchEmails`) and the actual `generateUnsubscribeToken` and `isUnsubscribed` functions to be mocked later.
    *   `type EmailType`:  Imports the TypeScript type `EmailType`.

```typescript
describe('mailer', () => {
  const testEmailOptions = {
    to: 'test@example.com',
    subject: 'Test Subject',
    html: '<p>Test email content</p>',
  }

  beforeEach(() => {
    vi.clearAllMocks()
    ;(isUnsubscribed as Mock).mockResolvedValue(false)
    ;(generateUnsubscribeToken as Mock).mockReturnValue('mock-token-123')

    // Mock successful Resend response
    mockSend.mockResolvedValue({
      data: { id: 'test-email-id' },
      error: null,
    })

    mockBatchSend.mockResolvedValue({
      data: [{ id: 'batch-email-1' }, { id: 'batch-email-2' }],
      error: null,
    })

    // Mock successful Azure response
    mockAzurePollUntilDone.mockResolvedValue({
      status: 'Succeeded',
      id: 'azure-email-id',
    })

    mockAzureBeginSend.mockReturnValue({
      pollUntilDone: mockAzurePollUntilDone,
    })
  })

  describe('sendEmail', () => {
    it('should send a transactional email successfully', async () => {
      const result = await sendEmail({
        ...testEmailOptions,
        emailType: 'transactional',
      })

      expect(result.success).toBe(true)
      expect(result.message).toBe('Email sent successfully via Resend')
      expect(result.data).toEqual({ id: 'test-email-id' })

      // Should not check unsubscribe status for transactional emails
      expect(isUnsubscribed).not.toHaveBeenCalled()

      // Should call Resend with correct parameters
      expect(mockSend).toHaveBeenCalledWith({
        from: 'Sim <noreply@sim.ai>',
        to: testEmailOptions.to,
        subject: testEmailOptions.subject,
        html: testEmailOptions.html,
        headers: undefined, // No unsubscribe headers for transactional
      })
    })

    it('should send a marketing email with unsubscribe headers', async () => {
      const htmlWithToken = '<p>Test content</p><a href="{{UNSUBSCRIBE_TOKEN}}">Unsubscribe</a>'

      const result = await sendEmail({
        ...testEmailOptions,
        html: htmlWithToken,
        emailType: 'marketing',
      })

      expect(result.success).toBe(true)

      // Should check unsubscribe status
      expect(isUnsubscribed).toHaveBeenCalledWith(testEmailOptions.to, 'marketing')

      // Should generate unsubscribe token
      expect(generateUnsubscribeToken).toHaveBeenCalledWith(testEmailOptions.to, 'marketing')

      // Should call Resend with unsubscribe headers
      expect(mockSend).toHaveBeenCalledWith({
        from: 'Sim <noreply@sim.ai>',
        to: testEmailOptions.to,
        subject: testEmailOptions.subject,
        html: '<p>Test content</p><a href="mock-token-123">Unsubscribe</a>',
        headers: {
          'List-Unsubscribe':
            '<https://test.sim.ai/unsubscribe?token=mock-token-123&email=test%40example.com>',
          'List-Unsubscribe-Post': 'List-Unsubscribe=One-Click',
        },
      })
    })

    it('should skip sending if user has unsubscribed', async () => {
      ;(isUnsubscribed as Mock).mockResolvedValue(true)

      const result = await sendEmail({
        ...testEmailOptions,
        emailType: 'marketing',
      })

      expect(result.success).toBe(true)
      expect(result.message).toBe('Email skipped (user unsubscribed)')
      expect(result.data).toEqual({ id: 'skipped-unsubscribed' })

      // Should not call Resend
      expect(mockSend).not.toHaveBeenCalled()
    })

    it.concurrent('should handle Resend API errors and fallback to Azure', async () => {
      // Mock Resend to fail
      mockSend.mockResolvedValue({
        data: null,
        error: { message: 'API rate limit exceeded' },
      })

      const result = await sendEmail(testEmailOptions)

      expect(result.success).toBe(true)
      expect(result.message).toBe('Email sent successfully via Azure Communication Services')
      expect(result.data).toEqual({ id: 'azure-email-id' })

      // Should have tried Resend first
      expect(mockSend).toHaveBeenCalled()

      // Should have fallen back to Azure
      expect(mockAzureBeginSend).toHaveBeenCalled()
    })

    it.concurrent('should handle unexpected errors and fallback to Azure', async () => {
      // Mock Resend to throw an error
      mockSend.mockRejectedValue(new Error('Network error'))

      const result = await sendEmail(testEmailOptions)

      expect(result.success).toBe(true)
      expect(result.message).toBe('Email sent successfully via Azure Communication Services')
      expect(result.data).toEqual({ id: 'azure-email-id' })

      // Should have tried Resend first
      expect(mockSend).toHaveBeenCalled()

      // Should have fallen back to Azure
      expect(mockAzureBeginSend).toHaveBeenCalled()
    })

    it.concurrent('should use custom from address when provided', async () => {
      await sendEmail({
        ...testEmailOptions,
        from: 'custom@example.com',
      })

      expect(mockSend).toHaveBeenCalledWith(
        expect.objectContaining({
          from: 'custom@example.com',
        })
      )
    })

    it('should not include unsubscribe when includeUnsubscribe is false', async () => {
      await sendEmail({
        ...testEmailOptions,
        emailType: 'marketing',
        includeUnsubscribe: false,
      })

      expect(generateUnsubscribeToken).not.toHaveBeenCalled()
      expect(mockSend).toHaveBeenCalledWith(
        expect.objectContaining({
          headers: undefined,
        })
      )
    })

    it.concurrent('should replace unsubscribe token placeholders in HTML', async () => {
      const htmlWithPlaceholder = '<p>Content</p><a href="{{UNSUBSCRIBE_TOKEN}}">Unsubscribe</a>'

      await sendEmail({
        ...testEmailOptions,
        html: htmlWithPlaceholder,
        emailType: 'updates' as EmailType,
      })

      expect(mockSend).toHaveBeenCalledWith(
        expect.objectContaining({
          html: '<p>Content</p><a href="mock-token-123">Unsubscribe</a>',
        })
      )
    })
  })

  describe('Azure Communication Services fallback', () => {
    it('should fallback to Azure when Resend fails', async () => {
      // Mock Resend to fail
      mockSend.mockRejectedValue(new Error('Resend service unavailable'))

      const result = await sendEmail({
        ...testEmailOptions,
        emailType: 'transactional',
      })

      expect(result.success).toBe(true)
      expect(result.message).toBe('Email sent successfully via Azure Communication Services')
      expect(result.data).toEqual({ id: 'azure-email-id' })

      // Should have tried Resend first
      expect(mockSend).toHaveBeenCalled()

      // Should have fallen back to Azure
      expect(mockAzureBeginSend).toHaveBeenCalledWith({
        senderAddress: 'noreply@sim.ai',
        content: {
          subject: testEmailOptions.subject,
          html: testEmailOptions.html,
        },
        recipients: {
          to: [{ address: testEmailOptions.to }],
        },
        headers: {},
      })
    })

    it('should handle Azure Communication Services failure', async () => {
      // Mock both services to fail
      mockSend.mockRejectedValue(new Error('Resend service unavailable'))
      mockAzurePollUntilDone.mockResolvedValue({
        status: 'Failed',
        id: 'failed-id',
      })

      const result = await sendEmail({
        ...testEmailOptions,
        emailType: 'transactional',
      })

      expect(result.success).toBe(false)
      expect(result.message).toBe('Both Resend and Azure Communication Services failed')

      // Should have tried both services
      expect(mockSend).toHaveBeenCalled()
      expect(mockAzureBeginSend).toHaveBeenCalled()
    })
  })

  describe('sendBatchEmails', () => {
    const testBatchEmails = [
      { ...testEmailOptions, to: 'user1@example.com' },
      { ...testEmailOptions, to: 'user2@example.com' },
    ]

    it('should send batch emails via Resend successfully', async () => {
      const result = await sendBatchEmails({ emails: testBatchEmails })

      expect(result.success).toBe(true)
      expect(result.message).toBe('All batch emails sent successfully via Resend')
      expect(result.results).toHaveLength(2)
      expect(mockBatchSend).toHaveBeenCalled()
    })

    it('should fallback to individual sends when Resend batch fails', async () => {
      // Mock Resend batch to fail
      mockBatchSend.mockRejectedValue(new Error('Batch service unavailable'))

      const result = await sendBatchEmails({ emails: testBatchEmails })

      expect(result.success).toBe(true)
      expect(result.message).toBe('All batch emails sent successfully')
      expect(result.results).toHaveLength(2)

      // Should have tried Resend batch first
      expect(mockBatchSend).toHaveBeenCalled()

      // Should have fallen back to individual sends (which will use Resend since it's available)
      expect(mockSend).toHaveBeenCalledTimes(2)
    })

    it('should handle mixed success/failure in individual fallback', async () => {
      // Mock Resend batch to fail
      mockBatchSend.mockRejectedValue(new Error('Batch service unavailable'))

      // Mock first individual send to succeed, second to fail and Azure also fails
      mockSend
        .mockResolvedValueOnce({
          data: { id: 'email-1' },
          error: null,
        })
        .mockRejectedValueOnce(new Error('Individual send failure'))

      // Mock Azure to fail for the second email (first call succeeds, but second fails)
      mockAzurePollUntilDone.mockResolvedValue({
        status: 'Failed',
        id: 'failed-id',
      })

      const result = await sendBatchEmails({ emails: testBatchEmails })

      expect(result.success).toBe(false)
      expect(result.message).toBe('1/2 emails sent successfully')
      expect(result.results).toHaveLength(2)
      expect(result.results[0].success).toBe(true)
      expect(result.results[1].success).toBe(false)
    })
  })
})
```

*   **`describe('mailer', () => { ... })`:**  Defines a test suite for the `mailer` module.

*   **`const testEmailOptions = { ... }`:**  Defines a constant object containing the default email options used in the tests.

*   **`beforeEach(() => { ... })`:**  This function runs before each test case within the `describe` block. It's used to set up the testing environment.
    *   `vi.clearAllMocks()`:  Resets all mock functions, clearing any previous calls and return values.  This ensures that each test starts with a clean slate.
    *   `(isUnsubscribed as Mock).mockResolvedValue(false)`: Configures the `isUnsubscribed` mock function to return `false` by default, meaning the user is not unsubscribed.  The `as Mock` type assertion is used to tell TypeScript that `isUnsubscribed` is a mock function.
    *   `(generateUnsubscribeToken as Mock).mockReturnValue('mock-token-123')`: Configures the `generateUnsubscribeToken` mock function to return a static unsubscribe token.
    *   `mockSend.mockResolvedValue({ data: { id: 'test-email-id' }, error: null })`: Configures the `mockSend` function to simulate a successful email send with Resend, returning a resolved Promise with sample data.
    *   `mockBatchSend.mockResolvedValue({ data: [{ id: 'batch-email-1' }, { id: 'batch-email-2' }], error: null })`: Configures the `mockBatchSend` function to simulate a successful batch email send with Resend.
    *   `mockAzurePollUntilDone.mockResolvedValue({ status: 'Succeeded', id: 'azure-email-id' })`:  Configures the Azure polling function to return a successful result.
    *   `mockAzureBeginSend.mockReturnValue({ pollUntilDone: mockAzurePollUntilDone })`:  Configures the Azure beginSend function to return an object with `pollUntilDone` function.

*   **`describe('sendEmail', () => { ... })`:** Defines a test suite specifically for the `sendEmail` function.

*   **`it('should send a transactional email successfully', async () => { ... })`:**  Defines a test case to verify that `sendEmail` sends transactional emails successfully.
    *   `const result = await sendEmail({ ...testEmailOptions, emailType: 'transactional' })`: Calls the `sendEmail` function with the test email options and `emailType` set to `'transactional'`.
    *   `expect(result.success).toBe(true)`: Asserts that the `success` property of the result is `true`.
    *   `expect(result.message).toBe('Email sent successfully via Resend')`:  Asserts that the `message` property matches the expected success message.
    *   `expect(result.data).toEqual({ id: 'test-email-id' })`: Asserts that the `data` property matches the mock Resend response.
    *   `expect(isUnsubscribed).not.toHaveBeenCalled()`: Asserts that the `isUnsubscribed` function was *not* called, as unsubscribe checks are not performed for transactional emails.
    *   `expect(mockSend).toHaveBeenCalledWith(...)`:  Asserts that the `mockSend` function was called with the expected arguments, including the sender address, recipient, subject, HTML content, and no headers (because it's transactional).

*   **`it('should send a marketing email with unsubscribe headers', async () => { ... })`:** Defines a test case for sending marketing emails, which should include unsubscribe headers.  It tests that `isUnsubscribed` and `generateUnsubscribeToken` were called, and that correct headers are passed to Resend.

*   **`it('should skip sending if user has unsubscribed', async () => { ... })`:**  Defines a test case to verify that email sending is skipped if the user has unsubscribed.

*   **`it.concurrent('should handle Resend API errors and fallback to Azure', async () => { ... })`:** Defines a test case to verify that the function correctly falls back to Azure Communication Services when the Resend API returns an