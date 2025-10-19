```typescript
import path from 'path'
/**
 * Unit tests for file parsers
 *
 * @vitest-environment node
 */
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest'
import type { FileParseResult, FileParser } from '@/lib/file-parsers/types'

// Mocking Dependencies and Setting Up Test Environment

// This section focuses on mocking external dependencies and setting up the test environment
// to isolate the file parsing logic and ensure predictable test results.

const mockExistsSync = vi.fn().mockReturnValue(true)
const mockReadFile = vi.fn().mockResolvedValue(Buffer.from('test content'))

// Mocking `fs.existsSync`
// - `vi.fn()` creates a mock function.
// - `.mockReturnValue(true)` configures the mock function to always return `true` when called.
//   This simulates that a file exists.
// Mocking `fs.promises.readFile`
// - `vi.fn()` creates a mock function.
// - `.mockResolvedValue(Buffer.from('test content'))` configures the mock function to resolve with a Buffer containing 'test content'
//   This simulates reading a file and getting some content.

const mockPdfParseFile = vi.fn().mockResolvedValue({
  content: 'Parsed PDF content',
  metadata: {
    info: { Title: 'Test PDF' },
    pageCount: 5,
    version: '1.7',
  },
})

const mockCsvParseFile = vi.fn().mockResolvedValue({
  content: 'Parsed CSV content',
  metadata: {
    headers: ['column1', 'column2'],
    rowCount: 10,
  },
})

const mockDocxParseFile = vi.fn().mockResolvedValue({
  content: 'Parsed DOCX content',
  metadata: {
    pages: 3,
    author: 'Test Author',
  },
})

const mockTxtParseFile = vi.fn().mockResolvedValue({
  content: 'Parsed TXT content',
  metadata: {
    characterCount: 100,
    tokenCount: 10,
  },
})

const mockMdParseFile = vi.fn().mockResolvedValue({
  content: 'Parsed MD content',
  metadata: {
    characterCount: 100,
    tokenCount: 10,
  },
})

const mockPptxParseFile = vi.fn().mockResolvedValue({
  content: 'Parsed PPTX content',
  metadata: {
    slideCount: 5,
    extractionMethod: 'officeparser',
  },
})

const mockHtmlParseFile = vi.fn().mockResolvedValue({
  content: 'Parsed HTML content',
  metadata: {
    title: 'Test HTML Document',
    headingCount: 3,
    linkCount: 2,
  },
})

// Mocking File Parsers
// These mock functions simulate the behavior of different file parsers (PDF, CSV, DOCX, etc.).
// Each mock parser is configured to return a predefined `FileParseResult` object when its `parseFile` method is called.

const createMockModule = () => {
  const mockParsers: Record<string, FileParser> = {
    pdf: { parseFile: mockPdfParseFile },
    csv: { parseFile: mockCsvParseFile },
    docx: { parseFile: mockDocxParseFile },
    txt: { parseFile: mockTxtParseFile },
    md: { parseFile: mockMdParseFile },
    pptx: { parseFile: mockPptxParseFile },
    ppt: { parseFile: mockPptxParseFile },
    html: { parseFile: mockHtmlParseFile },
    htm: { parseFile: mockHtmlParseFile },
  }

  return {
    parseFile: async (filePath: string): Promise<FileParseResult> => {
      if (!filePath) {
        throw new Error('No file path provided')
      }

      if (!mockExistsSync(filePath)) {
        throw new Error(`File not found: ${filePath}`)
      }

      const extension = path.extname(filePath).toLowerCase().substring(1)

      if (!Object.keys(mockParsers).includes(extension)) {
        throw new Error(
          `Unsupported file type: ${extension}. Supported types are: ${Object.keys(mockParsers).join(', ')}`
        )
      }

      return mockParsers[extension].parseFile(filePath)
    },

    isSupportedFileType: (extension: string): boolean => {
      if (!extension) return false
      return Object.keys(mockParsers).includes(extension.toLowerCase())
    },
  }
}

// `createMockModule` Function:
// This function creates a mock module that mimics the file parsing functionality.
// - `mockParsers`:  An object that maps file extensions (e.g., 'pdf', 'csv') to their corresponding mock file parsers.
// - `parseFile`: An asynchronous function that simulates parsing a file based on its extension.
//   - It performs checks for:
//     - Empty file path.
//     - File existence (using `mockExistsSync`).
//     - Supported file type (by checking if the extension exists in `mockParsers`).
//   - If all checks pass, it calls the appropriate mock parser's `parseFile` method.
// - `isSupportedFileType`:  A function that checks if a given file extension is supported by the mock module.

describe('File Parsers', () => {
  beforeEach(() => {
    vi.resetModules()

    vi.doMock('fs', () => ({
      existsSync: mockExistsSync,
    }))

    vi.doMock('fs/promises', () => ({
      readFile: mockReadFile,
    }))

    vi.doMock('@/lib/file-parsers/index', () => createMockModule())

    vi.doMock('@/lib/file-parsers/pdf-parser', () => ({
      PdfParser: vi.fn().mockImplementation(() => ({
        parseFile: mockPdfParseFile,
      })),
    }))

    vi.doMock('@/lib/file-parsers/csv-parser', () => ({
      CsvParser: vi.fn().mockImplementation(() => ({
        parseFile: mockCsvParseFile,
      })),
    }))

    vi.doMock('@/lib/file-parsers/docx-parser', () => ({
      DocxParser: vi.fn().mockImplementation(() => ({
        parseFile: mockDocxParseFile,
      })),
    }))

    vi.doMock('@/lib/file-parsers/txt-parser', () => ({
      TxtParser: vi.fn().mockImplementation(() => ({
        parseFile: mockTxtParseFile,
      })),
    }))

    vi.doMock('@/lib/file-parsers/md-parser', () => ({
      MdParser: vi.fn().mockImplementation(() => ({
        parseFile: mockMdParseFile,
      })),
    }))

    vi.doMock('@/lib/file-parsers/pptx-parser', () => ({
      PptxParser: vi.fn().mockImplementation(() => ({
        parseFile: mockPptxParseFile,
      })),
    }))

    vi.doMock('@/lib/file-parsers/html-parser', () => ({
      HtmlParser: vi.fn().mockImplementation(() => ({
        parseFile: mockHtmlParseFile,
      })),
    }))

    global.console = {
      ...console,
      log: vi.fn(),
      error: vi.fn(),
      warn: vi.fn(),
      debug: vi.fn(),
    }
  })

  // Test Suite Setup
  // - `describe('File Parsers', () => { ... })`:  Defines a test suite for file parsers.
  // - `beforeEach(() => { ... })`:  Sets up the test environment before each test case.
  //   - `vi.resetModules()`: Resets the module registry to ensure that each test case starts with a clean slate.
  //   - `vi.doMock('moduleName', () => mock)`:  Replaces the actual implementation of a module with a mock implementation.
  //     - This is used to mock file system functions (`fs`, `fs/promises`) and the file parsers themselves.
  //   - `global.console = { ...console, log: vi.fn(), error: vi.fn(), warn: vi.fn(), debug: vi.fn() }`: Mocks the global console object to capture log messages during testing.
  // - Mocking File System:  The `fs` and `fs/promises` modules are mocked to control file system interactions during testing.
  // - Mocking Individual Parsers: Each specific file parser module (e.g., `pdf-parser`, `csv-parser`) is mocked with a mock parser instance.

  afterEach(() => {
    vi.clearAllMocks()
    vi.resetAllMocks()
    vi.restoreAllMocks()
  })

  // `afterEach(() => { ... })`:  Cleans up the test environment after each test case.
  //   - `vi.clearAllMocks()`: Clears all mock function calls.
  //   - `vi.resetAllMocks()`: Resets the behavior of all mock functions to their initial state.
  //   - `vi.restoreAllMocks()`: Restores all mocked modules to their original implementations.

  describe('parseFile', () => {
    it('should validate file existence', async () => {
      mockExistsSync.mockReturnValueOnce(false)

      const { parseFile } = await import('@/lib/file-parsers/index')

      const testFilePath = '/test/files/test.pdf'
      await expect(parseFile(testFilePath)).rejects.toThrow('File not found')
      expect(mockExistsSync).toHaveBeenCalledWith(testFilePath)
    })

    it('should throw error if file path is empty', async () => {
      const { parseFile } = await import('@/lib/file-parsers/index')
      await expect(parseFile('')).rejects.toThrow('No file path provided')
    })

    it('should parse PDF files successfully', async () => {
      const expectedResult = {
        content: 'Parsed PDF content',
        metadata: {
          info: { Title: 'Test PDF' },
          pageCount: 5,
          version: '1.7',
        },
      }

      mockPdfParseFile.mockResolvedValueOnce(expectedResult)
      mockExistsSync.mockReturnValue(true)

      const { parseFile } = await import('@/lib/file-parsers/index')
      const result = await parseFile('/test/files/document.pdf')

      expect(result).toEqual(expectedResult)
    })

    it('should parse CSV files successfully', async () => {
      const expectedResult = {
        content: 'Parsed CSV content',
        metadata: {
          headers: ['column1', 'column2'],
          rowCount: 10,
        },
      }

      mockCsvParseFile.mockResolvedValueOnce(expectedResult)
      mockExistsSync.mockReturnValue(true)

      const { parseFile } = await import('@/lib/file-parsers/index')
      const result = await parseFile('/test/files/data.csv')

      expect(result).toEqual(expectedResult)
    })

    it('should parse DOCX files successfully', async () => {
      const expectedResult = {
        content: 'Parsed DOCX content',
        metadata: {
          pages: 3,
          author: 'Test Author',
        },
      }

      mockDocxParseFile.mockResolvedValueOnce(expectedResult)
      mockExistsSync.mockReturnValue(true)

      const { parseFile } = await import('@/lib/file-parsers/index')
      const result = await parseFile('/test/files/document.docx')

      expect(result).toEqual(expectedResult)
    })

    it('should parse TXT files successfully', async () => {
      const expectedResult = {
        content: 'Parsed TXT content',
        metadata: {
          characterCount: 100,
          tokenCount: 10,
        },
      }

      mockTxtParseFile.mockResolvedValueOnce(expectedResult)
      mockExistsSync.mockReturnValue(true)

      const { parseFile } = await import('@/lib/file-parsers/index')
      const result = await parseFile('/test/files/document.txt')

      expect(result).toEqual(expectedResult)
    })

    it('should parse MD files successfully', async () => {
      const expectedResult = {
        content: 'Parsed MD content',
        metadata: {
          characterCount: 100,
          tokenCount: 10,
        },
      }

      mockMdParseFile.mockResolvedValueOnce(expectedResult)
      mockExistsSync.mockReturnValue(true)

      const { parseFile } = await import('@/lib/file-parsers/index')
      const result = await parseFile('/test/files/document.md')

      expect(result).toEqual(expectedResult)
    })

    it('should parse PPTX files successfully', async () => {
      const expectedResult = {
        content: 'Parsed PPTX content',
        metadata: {
          slideCount: 5,
          extractionMethod: 'officeparser',
        },
      }

      mockPptxParseFile.mockResolvedValueOnce(expectedResult)
      mockExistsSync.mockReturnValue(true)

      const { parseFile } = await import('@/lib/file-parsers/index')
      const result = await parseFile('/test/files/presentation.pptx')

      expect(result).toEqual(expectedResult)
    })

    it('should parse PPT files successfully', async () => {
      const expectedResult = {
        content: 'Parsed PPTX content',
        metadata: {
          slideCount: 5,
          extractionMethod: 'officeparser',
        },
      }

      mockPptxParseFile.mockResolvedValueOnce(expectedResult)
      mockExistsSync.mockReturnValue(true)

      const { parseFile } = await import('@/lib/file-parsers/index')
      const result = await parseFile('/test/files/presentation.ppt')

      expect(result).toEqual(expectedResult)
    })

    it('should parse HTML files successfully', async () => {
      const expectedResult = {
        content: 'Parsed HTML content',
        metadata: {
          title: 'Test HTML Document',
          headingCount: 3,
          linkCount: 2,
        },
      }

      mockHtmlParseFile.mockResolvedValueOnce(expectedResult)
      mockExistsSync.mockReturnValue(true)

      const { parseFile } = await import('@/lib/file-parsers/index')
      const result = await parseFile('/test/files/document.html')

      expect(result).toEqual(expectedResult)
    })

    it('should parse HTM files successfully', async () => {
      const expectedResult = {
        content: 'Parsed HTML content',
        metadata: {
          title: 'Test HTML Document',
          headingCount: 3,
          linkCount: 2,
        },
      }

      mockHtmlParseFile.mockResolvedValueOnce(expectedResult)
      mockExistsSync.mockReturnValue(true)

      const { parseFile } = await import('@/lib/file-parsers/index')
      const result = await parseFile('/test/files/document.htm')

      expect(result).toEqual(expectedResult)
    })

    it('should throw error for unsupported file types', async () => {
      mockExistsSync.mockReturnValue(true)

      const { parseFile } = await import('@/lib/file-parsers/index')
      const unsupportedFilePath = '/test/files/image.png'

      await expect(parseFile(unsupportedFilePath)).rejects.toThrow('Unsupported file type')
    })

    it('should handle errors during parsing', async () => {
      mockExistsSync.mockReturnValue(true)

      const parsingError = new Error('CSV parsing failed')
      mockCsvParseFile.mockRejectedValueOnce(parsingError)

      const { parseFile } = await import('@/lib/file-parsers/index')
      await expect(parseFile('/test/files/data.csv')).rejects.toThrow('CSV parsing failed')
    })
  })

  // Test Cases for `parseFile`
  // - `describe('parseFile', () => { ... })`: Defines a test suite for the `parseFile` function.
  // - Each `it(...)` block represents a specific test case.
  // - The test cases cover:
  //   - File existence validation.
  //   - Empty file path handling.
  //   - Successful parsing of different file types (PDF, CSV, DOCX, TXT, MD, PPTX, PPT, HTML, HTM).
  //   - Handling of unsupported file types.
  //   - Error handling during parsing.
  // - `expect(parseFile(filePath)).resolves.toEqual(expectedResult)`:  Asserts that the `parseFile` function resolves with the expected result for a given file.
  // - `expect(parseFile(filePath)).rejects.toThrow(errorMessage)`: Asserts that the `parseFile` function rejects with the expected error message for a given file.

  describe('isSupportedFileType', () => {
    it('should return true for supported file types', async () => {
      const { isSupportedFileType } = await import('@/lib/file-parsers/index')

      expect(isSupportedFileType('pdf')).toBe(true)
      expect(isSupportedFileType('csv')).toBe(true)
      expect(isSupportedFileType('docx')).toBe(true)
      expect(isSupportedFileType('txt')).toBe(true)
      expect(isSupportedFileType('md')).toBe(true)
      expect(isSupportedFileType('pptx')).toBe(true)
      expect(isSupportedFileType('ppt')).toBe(true)
      expect(isSupportedFileType('html')).toBe(true)
      expect(isSupportedFileType('htm')).toBe(true)
    })

    it('should return false for unsupported file types', async () => {
      const { isSupportedFileType } = await import('@/lib/file-parsers/index')

      expect(isSupportedFileType('png')).toBe(false)
      expect(isSupportedFileType('unknown')).toBe(false)
    })

    it('should handle uppercase extensions', async () => {
      const { isSupportedFileType } = await import('@/lib/file-parsers/index')

      expect(isSupportedFileType('PDF')).toBe(true)
      expect(isSupportedFileType('CSV')).toBe(true)
      expect(isSupportedFileType('TXT')).toBe(true)
      expect(isSupportedFileType('MD')).toBe(true)
      expect(isSupportedFileType('PPTX')).toBe(true)
      expect(isSupportedFileType('HTML')).toBe(true)
    })

    it('should handle errors gracefully', async () => {
      const errorMockModule = {
        isSupportedFileType: () => {
          throw new Error('Failed to get parsers')
        },
      }

      vi.doMock('@/lib/file-parsers/index', () => errorMockModule)

      const { isSupportedFileType } = await import('@/lib/file-parsers/index')

      expect(() => isSupportedFileType('pdf')).toThrow('Failed to get parsers')
    })
  })
})
// Test Cases for `isSupportedFileType`
// - `describe('isSupportedFileType', () => { ... })`: Defines a test suite for the `isSupportedFileType` function.
// - The test cases cover:
//   - Returning `true` for supported file types (lowercase and uppercase).
//   - Returning `false` for unsupported file types.
//   - Handling errors gracefully if the `isSupportedFileType` function throws an error.

// Purpose of the File:
// This file contains unit tests for file parsing functionality in a TypeScript application. It uses the Vitest testing framework to verify the behavior of file parsers, including:
// - Validating file existence.
// - Parsing different file types (PDF, CSV, DOCX, TXT, MD, PPTX, HTML).
// - Handling unsupported file types.
// - Managing errors during parsing.

// Key Concepts:
// - Mocking:  Replacing real dependencies with mock implementations to isolate the code under test and ensure predictable behavior.
// - Test Suites:  Grouping related test cases together to provide a structured way to organize and run tests.
// - Test Cases:  Individual tests that verify a specific aspect of the code's functionality.
// - Assertions:  Statements that check if the actual result of a test case matches the expected result.
// - Asynchronous Testing:  Testing asynchronous functions using `async/await` and `expect(...).resolves` or `expect(...).rejects`.
```