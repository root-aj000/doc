```typescript
/**
 * URL Validator for MCP Servers
 *
 * Provides SSRF (Server-Side Request Forgery) protection by validating
 * MCP server URLs against common attack patterns and dangerous destinations.
 */

import { createLogger } from '@/lib/logs/console/logger'

const logger = createLogger('McpUrlValidator')

// Blocked IPv4 ranges
const PRIVATE_IP_RANGES = [
  /^127\./, // Loopback (127.0.0.0/8)
  /^10\./, // Private class A (10.0.0.0/8)
  /^172\.(1[6-9]|2[0-9]|3[01])\./, // Private class B (172.16.0.0/12)
  /^192\.168\./, // Private class C (192.168.0.0/16)
  /^169\.254\./, // Link-local (169.254.0.0/16)
  /^0\./, // Invalid range
]

// Blocked IPv6 ranges
const PRIVATE_IPV6_RANGES = [
  /^::1$/, // Localhost
  /^::ffff:/, // IPv4-mapped IPv6
  /^fc00:/, // Unique local (fc00::/7)
  /^fd00:/, // Unique local (fd00::/8)
  /^fe80:/, // Link-local (fe80::/10)
]

// Blocked hostnames - SSRF protection
const BLOCKED_HOSTNAMES = [
  'localhost',
  // Cloud metadata endpoints
  'metadata.google.internal', // Google Cloud metadata
  'metadata.gce.internal', // Google Compute Engine metadata (legacy)
  '169.254.169.254', // AWS/Azure/GCP metadata service IP
  'metadata.azure.com', // Azure Instance Metadata Service
  'instance-data.ec2.internal', // AWS EC2 instance metadata (internal)
  // Service discovery endpoints
  'consul', // HashiCorp Consul
  'etcd', // etcd key-value store
]

// Blocked ports
const BLOCKED_PORTS = [
  22, // SSH
  23, // Telnet
  25, // SMTP
  53, // DNS
  110, // POP3
  143, // IMAP
  993, // IMAPS
  995, // POP3S
  1433, // SQL Server
  1521, // Oracle
  3306, // MySQL
  5432, // PostgreSQL
  6379, // Redis
  9200, // Elasticsearch
  27017, // MongoDB
]

export interface UrlValidationResult {
  isValid: boolean
  error?: string
  normalizedUrl?: string
}

export function validateMcpServerUrl(urlString: string): UrlValidationResult {
  if (!urlString || typeof urlString !== 'string') {
    return {
      isValid: false,
      error: 'URL is required and must be a string',
    }
  }

  let url: URL
  try {
    url = new URL(urlString.trim())
  } catch (error) {
    return {
      isValid: false,
      error: 'Invalid URL format',
    }
  }

  if (!['http:', 'https:'].includes(url.protocol)) {
    return {
      isValid: false,
      error: 'Only HTTP and HTTPS protocols are allowed',
    }
  }

  const hostname = url.hostname.toLowerCase()

  if (BLOCKED_HOSTNAMES.includes(hostname)) {
    return {
      isValid: false,
      error: `Hostname '${hostname}' is not allowed for security reasons`,
    }
  }

  if (isIPv4(hostname)) {
    for (const range of PRIVATE_IP_RANGES) {
      if (range.test(hostname)) {
        return {
          isValid: false,
          error: `Private IP addresses are not allowed: ${hostname}`,
        }
      }
    }
  }

  if (isIPv6(hostname)) {
    for (const range of PRIVATE_IPV6_RANGES) {
      if (range.test(hostname)) {
        return {
          isValid: false,
          error: `Private IPv6 addresses are not allowed: ${hostname}`,
        }
      }
    }
  }

  if (url.port) {
    const port = Number.parseInt(url.port, 10)
    if (BLOCKED_PORTS.includes(port)) {
      return {
        isValid: false,
        error: `Port ${port} is not allowed for security reasons`,
      }
    }
  }

  if (url.toString().length > 2048) {
    return {
      isValid: false,
      error: 'URL is too long (maximum 2048 characters)',
    }
  }

  if (url.protocol === 'https:' && url.port === '80') {
    return {
      isValid: false,
      error: 'HTTPS URLs should not use port 80',
    }
  }

  if (url.protocol === 'http:' && url.port === '443') {
    return {
      isValid: false,
      error: 'HTTP URLs should not use port 443',
    }
  }

  logger.debug(`Validated MCP server URL: ${hostname}`)

  return {
    isValid: true,
    normalizedUrl: url.toString(),
  }
}

function isIPv4(hostname: string): boolean {
  const ipv4Regex =
    /^(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$/
  return ipv4Regex.test(hostname)
}

function isIPv6(hostname: string): boolean {
  const cleanHostname = hostname.replace(/^\[|\]$/g, '')

  const ipv6Regex =
    /^(?:[0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}$|^::$|^::1$|^(?:[0-9a-fA-F]{1,4}:)*::[0-9a-fA-F]{1,4}(?::[0-9a-fA-F]{1,4})*$/

  return ipv6Regex.test(cleanHostname)
}
```

## Explanation of the Code

This TypeScript code provides a robust URL validator specifically designed for MCP (presumably "My Custom Protocol") servers. Its primary goal is to prevent Server-Side Request Forgery (SSRF) attacks by carefully scrutinizing provided URLs. SSRF vulnerabilities allow attackers to make requests to internal resources or external systems on behalf of the server, potentially leading to data breaches or other malicious actions.

Here's a breakdown of each section:

**1. File Header:**

```typescript
/**
 * URL Validator for MCP Servers
 *
 * Provides SSRF (Server-Side Request Forgery) protection by validating
 * MCP server URLs against common attack patterns and dangerous destinations.
 */
```

This section is a JSDoc-style comment that describes the purpose of the file. It highlights that the code is designed to validate URLs intended for MCP servers and, more importantly, to protect against SSRF attacks.

**2. Imports:**

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```

This line imports a function called `createLogger` from a module located at `'@/lib/logs/console/logger'`.  This function is likely used to create a logger instance for recording events and debugging information during the URL validation process. The `@/` alias typically refers to the project's root directory, meaning the `logger.ts` file is within the project's logging library.

**3. Logger Instance:**

```typescript
const logger = createLogger('McpUrlValidator')
```

This line creates a logger instance using the `createLogger` function imported in the previous step.  The string `'McpUrlValidator'` is passed as an argument, likely to serve as an identifier or category for log messages generated by this particular validator. This helps in filtering and analyzing logs.

**4. Blocked IPv4 Ranges:**

```typescript
// Blocked IPv4 ranges
const PRIVATE_IP_RANGES = [
  /^127\./, // Loopback (127.0.0.0/8)
  /^10\./, // Private class A (10.0.0.0/8)
  /^172\.(1[6-9]|2[0-9]|3[01])\./, // Private class B (172.16.0.0/12)
  /^192\.168\./, // Private class C (192.168.0.0/16)
  /^169\.254\./, // Link-local (169.254.0.0/16)
  /^0\./, // Invalid range
]
```

This section defines an array called `PRIVATE_IP_RANGES`. It contains regular expressions that match common private IPv4 address ranges.  These ranges are blocked to prevent requests to internal services or the loopback address, which could be exploited in an SSRF attack.  The comments next to each regex explain the specific IP range it targets.

*   `^127\.`:  Matches the loopback address range (127.0.0.0/8), used for local testing and communication.
*   `^10\.`: Matches the private Class A address range (10.0.0.0/8), commonly used in private networks.
*   `^172\.(1[6-9]|2[0-9]|3[01])\.`:  Matches the private Class B address range (172.16.0.0/12).  The `(1[6-9]|2[0-9]|3[01])` part ensures that only the valid second octets (16-31) are matched.
*   `^192\.168\.`: Matches the private Class C address range (192.168.0.0/16), also commonly used in private networks.
*   `^169\.254\.`: Matches the link-local address range (169.254.0.0/16), used for automatic address assignment when a DHCP server is not available.  These addresses are typically not routable.
*   `^0\.`: Matches IPs starting with 0, which are invalid.

**5. Blocked IPv6 Ranges:**

```typescript
// Blocked IPv6 ranges
const PRIVATE_IPV6_RANGES = [
  /^::1$/, // Localhost
  /^::ffff:/, // IPv4-mapped IPv6
  /^fc00:/, // Unique local (fc00::/7)
  /^fd00:/, // Unique local (fd00::/8)
  /^fe80:/, // Link-local (fe80::/10)
]
```

Similar to the IPv4 section, this defines an array `PRIVATE_IPV6_RANGES` containing regular expressions that match private IPv6 address ranges. The purpose is the same: to block requests to potentially vulnerable internal IPv6 addresses.

*   `^::1$`: Matches the IPv6 loopback address.
*   `^::ffff:`: Matches IPv4-mapped IPv6 addresses, which are used to represent IPv4 addresses within an IPv6 environment.  Blocking these can prevent bypassing IPv4 restrictions.
*   `^fc00:`: Matches Unique Local Addresses (ULA) with global ID (fc00::/7).
*   `^fd00:`: Matches Unique Local Addresses (ULA) without global ID (fd00::/8).
*   `^fe80:`: Matches link-local IPv6 addresses.

**6. Blocked Hostnames:**

```typescript
// Blocked hostnames - SSRF protection
const BLOCKED_HOSTNAMES = [
  'localhost',
  // Cloud metadata endpoints
  'metadata.google.internal', // Google Cloud metadata
  'metadata.gce.internal', // Google Compute Engine metadata (legacy)
  '169.254.169.254', // AWS/Azure/GCP metadata service IP
  'metadata.azure.com', // Azure Instance Metadata Service
  'instance-data.ec2.internal', // AWS EC2 instance metadata (internal)
  // Service discovery endpoints
  'consul', // HashiCorp Consul
  'etcd', // etcd key-value store
]
```

This array, `BLOCKED_HOSTNAMES`, contains a list of hostnames that are explicitly blocked.  This is a critical part of SSRF protection. It includes:

*   `localhost`: The local machine.
*   Cloud metadata endpoints:  These endpoints expose sensitive information about the server instance running in cloud environments (AWS, Azure, Google Cloud).  Accessing these endpoints could leak credentials or configuration details. The specific endpoints listed are common for Google Cloud, AWS, and Azure.
*   Service discovery endpoints:  `consul` and `etcd` are service discovery tools. Accessing these can reveal internal service configurations and potentially lead to unauthorized access.

**7. Blocked Ports:**

```typescript
// Blocked ports
const BLOCKED_PORTS = [
  22, // SSH
  23, // Telnet
  25, // SMTP
  53, // DNS
  110, // POP3
  143, // IMAP
  993, // IMAPS
  995, // POP3S
  1433, // SQL Server
  1521, // Oracle
  3306, // MySQL
  5432, // PostgreSQL
  6379, // Redis
  9200, // Elasticsearch
  27017, // MongoDB
]
```

The `BLOCKED_PORTS` array lists common ports that are often associated with potentially vulnerable or sensitive services. Blocking these ports reduces the attack surface by preventing requests to these services from the MCP server.  The comments indicate the services associated with each port.  For example, port 22 is used for SSH, and port 3306 is used for MySQL.

**8. UrlValidationResult Interface:**

```typescript
export interface UrlValidationResult {
  isValid: boolean
  error?: string
  normalizedUrl?: string
}
```

This interface defines the structure of the object returned by the `validateMcpServerUrl` function. It includes:

*   `isValid`: A boolean indicating whether the URL is valid.
*   `error`: An optional string containing an error message if the URL is invalid.
*   `normalizedUrl`: An optional string containing the normalized (e.g., trimmed) URL if it is valid.

**9. `validateMcpServerUrl` Function:**

```typescript
export function validateMcpServerUrl(urlString: string): UrlValidationResult {
  if (!urlString || typeof urlString !== 'string') {
    return {
      isValid: false,
      error: 'URL is required and must be a string',
    }
  }

  let url: URL
  try {
    url = new URL(urlString.trim())
  } catch (error) {
    return {
      isValid: false,
      error: 'Invalid URL format',
    }
  }

  if (!['http:', 'https:'].includes(url.protocol)) {
    return {
      isValid: false,
      error: 'Only HTTP and HTTPS protocols are allowed',
    }
  }

  const hostname = url.hostname.toLowerCase()

  if (BLOCKED_HOSTNAMES.includes(hostname)) {
    return {
      isValid: false,
      error: `Hostname '${hostname}' is not allowed for security reasons`,
    }
  }

  if (isIPv4(hostname)) {
    for (const range of PRIVATE_IP_RANGES) {
      if (range.test(hostname)) {
        return {
          isValid: false,
          error: `Private IP addresses are not allowed: ${hostname}`,
        }
      }
    }
  }

  if (isIPv6(hostname)) {
    for (const range of PRIVATE_IPV6_RANGES) {
      if (range.test(hostname)) {
        return {
          isValid: false,
          error: `Private IPv6 addresses are not allowed: ${hostname}`,
        }
      }
    }
  }

  if (url.port) {
    const port = Number.parseInt(url.port, 10)
    if (BLOCKED_PORTS.includes(port)) {
      return {
        isValid: false,
        error: `Port ${port} is not allowed for security reasons`,
      }
    }
  }

  if (url.toString().length > 2048) {
    return {
      isValid: false,
      error: 'URL is too long (maximum 2048 characters)',
    }
  }

  if (url.protocol === 'https:' && url.port === '80') {
    return {
      isValid: false,
      error: 'HTTPS URLs should not use port 80',
    }
  }

  if (url.protocol === 'http:' && url.port === '443') {
    return {
      isValid: false,
      error: 'HTTP URLs should not use port 443',
    }
  }

  logger.debug(`Validated MCP server URL: ${hostname}`)

  return {
    isValid: true,
    normalizedUrl: url.toString(),
  }
}
```

This is the main function that performs the URL validation. Let's break it down step by step:

1.  **Input Validation:**

    ```typescript
    if (!urlString || typeof urlString !== 'string') {
      return {
        isValid: false,
        error: 'URL is required and must be a string',
      }
    }
    ```

    It first checks if the input `urlString` is valid (not null or empty) and is of type string. If not, it returns an error indicating that the URL is required and must be a string.

2.  **URL Parsing:**

    ```typescript
    let url: URL
    try {
      url = new URL(urlString.trim())
    } catch (error) {
      return {
        isValid: false,
        error: 'Invalid URL format',
      }
    }
    ```

    It attempts to create a `URL` object from the input string.  The `trim()` method removes leading/trailing whitespace.  The `try...catch` block handles potential errors during URL parsing. If the URL is not in a valid format, it returns an error.

3.  **Protocol Check:**

    ```typescript
    if (!['http:', 'https:'].includes(url.protocol)) {
      return {
        isValid: false,
        error: 'Only HTTP and HTTPS protocols are allowed',
      }
    }
    ```

    It verifies that the URL uses either the HTTP or HTTPS protocol. Other protocols are not allowed for security reasons.

4.  **Hostname Check:**

    ```typescript
    const hostname = url.hostname.toLowerCase()

    if (BLOCKED_HOSTNAMES.includes(hostname)) {
      return {
        isValid: false,
        error: `Hostname '${hostname}' is not allowed for security reasons`,
      }
    }
    ```

    It extracts the hostname from the URL and converts it to lowercase for case-insensitive comparison. Then, it checks if the hostname is present in the `BLOCKED_HOSTNAMES` array. If it is, the URL is considered invalid.

5.  **IPv4 Address Check:**

    ```typescript
    if (isIPv4(hostname)) {
      for (const range of PRIVATE_IP_RANGES) {
        if (range.test(hostname)) {
          return {
            isValid: false,
            error: `Private IP addresses are not allowed: ${hostname}`,
          }
        }
      }
    }
    ```

    If the hostname is a valid IPv4 address (determined by the `isIPv4` function), it iterates through the `PRIVATE_IP_RANGES` array and checks if the hostname matches any of the blocked private IP ranges.

6.  **IPv6 Address Check:**

    ```typescript
    if (isIPv6(hostname)) {
      for (const range of PRIVATE_IPV6_RANGES) {
        if (range.test(hostname)) {
          return {
            isValid: false,
            error: `Private IPv6 addresses are not allowed: ${hostname}`,
          }
        }
      }
    }
    ```

    If the hostname is a valid IPv6 address (determined by the `isIPv6` function), it performs a similar check as with IPv4, iterating through the `PRIVATE_IPV6_RANGES` array.

7.  **Port Check:**

    ```typescript
    if (url.port) {
      const port = Number.parseInt(url.port, 10)
      if (BLOCKED_PORTS.includes(port)) {
        return {
          isValid: false,
          error: `Port ${port} is not allowed for security reasons`,
        }
      }
    }
    ```

    If the URL includes a port number, it converts it to an integer and checks if it is present in the `BLOCKED_PORTS` array.

8.  **URL Length Check:**

    ```typescript
    if (url.toString().length > 2048) {
      return {
        isValid: false,
        error: 'URL is too long (maximum 2048 characters)',
      }
    }
    ```

    It checks if the length of the entire URL string exceeds 2048 characters. This prevents excessively long URLs, which could potentially be used in denial-of-service attacks or to bypass security filters.

9.  **Protocol/Port Combination Check:**

    ```typescript
    if (url.protocol === 'https:' && url.port === '80') {
      return {
        isValid: false,
        error: 'HTTPS URLs should not use port 80',
      }
    }

    if (url.protocol === 'http:' && url.port === '443') {
      return {
        isValid: false,
        error: 'HTTP URLs should not use port 443',
      }
    }
    ```

    These checks ensure that HTTPS URLs don't use port 80 (the standard HTTP port) and HTTP URLs don't use port 443 (the standard HTTPS port).  Using the wrong port with a protocol is often a configuration error or an attempt to bypass security measures.

10. **Successful Validation:**

    ```typescript
    logger.debug(`Validated MCP server URL: ${hostname}`)

    return {
      isValid: true,
      normalizedUrl: url.toString(),
    }
    ```

    If all the checks pass, the function logs a debug message indicating that the URL has been validated and returns an object indicating that the URL is valid. The `normalizedUrl` field contains the string representation of the validated URL object.

**10. `isIPv4` Function:**

```typescript
function isIPv4(hostname: string): boolean {
  const ipv4Regex =
    /^(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$/
  return ipv4Regex.test(hostname)
}
```

This function checks if a given string is a valid IPv4 address. It uses a regular expression to match the standard IPv4 address format (four sets of numbers between 0 and 255, separated by dots).

**11. `isIPv6` Function:**

```typescript
function isIPv6(hostname: string): boolean {
  const cleanHostname = hostname.replace(/^\[|\]$/g, '')

  const ipv6Regex =
    /^(?:[0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}$|^::$|^::1$|^(?:[0-9a-fA-F]{1,4}:)*::[0-9a-fA-F]{1,4}(?::[0-9a-fA-F]{1,4})*$/

  return ipv6Regex.test(cleanHostname)
}
```

This function checks if a given string is a valid IPv6 address.

1.  `const cleanHostname = hostname.replace(/^\[|\]$/g, '')` - remove square brackets that may surround the IPv6 address.
2.  It uses a regular expression to match the complex IPv6 address format. The regex accounts for various valid IPv6 representations, including full addresses, shortened addresses with `::`, and the loopback address `::1`.

**In Summary:**

This code implements a comprehensive URL validator that prioritizes security by preventing SSRF attacks. It blocks access to private IP ranges, cloud metadata endpoints, service discovery tools, and potentially vulnerable ports. It also enforces protocol and port consistency and limits the length of URLs to prevent exploitation.  The use of regular expressions for IP address validation and the explicit blocking of dangerous hostnames and ports make this a robust and effective security measure. The logging also allows for auditing URL validation attempts.
