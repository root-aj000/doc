```typescript
import { quickValidateEmail } from '@/lib/email/validation'
import type { Organization } from '@/stores/organization/types'

/**
 * Calculate seat usage for an organization
 */
export function calculateSeatUsage(org?: Organization | null) {
  const members = org?.members?.length ?? 0
  const pending = org?.invitations?.filter((inv) => inv.status === 'pending').length ?? 0
  return { used: members + pending, members, pending }
}

/**
 * Generate a URL-friendly slug from a name
 */
export function generateSlug(name: string): string {
  return name
    .toLowerCase()
    .replace(/[^a-z0-9]/g, '-') // Replace non-alphanumeric with hyphens
    .replace(/-+/g, '-') // Replace consecutive hyphens with single hyphen
    .replace(/^-|-$/g, '') // Remove leading and trailing hyphens
}

/**
 * Validate organization slug format
 */
export function validateSlug(slug: string): boolean {
  const slugRegex = /^[a-z0-9-_]+$/
  return slugRegex.test(slug)
}

/**
 * Validate email format
 */
export function validateEmail(email: string): boolean {
  return quickValidateEmail(email.trim().toLowerCase()).isValid
}
```

## Explanation of the Code

This TypeScript file provides a collection of utility functions related to organizations, slugs, and email validation.  Let's break down each function:

**1. `calculateSeatUsage(org?: Organization | null)`**

   * **Purpose:** This function calculates the seat usage for an organization, considering both active members and pending invitations.
   * **Input:**  `org?: Organization | null` -  An optional `Organization` object or `null`. The `?` means it can be undefined, and `| null` means it can be explicitly null. The `Organization` type is imported from `'@/stores/organization/types'`.  It likely defines the structure of an organization object, including properties like `members` and `invitations`.
   * **Logic:**
     * `const members = org?.members?.length ?? 0`
       * This line safely accesses the number of members in the organization.
       * `org?.members` uses optional chaining.  If `org` is `null` or `undefined`, or if `org.members` is `null` or `undefined`, the expression evaluates to `undefined` instead of throwing an error.  This prevents errors if the organization data is incomplete.
       * `?.length` attempts to access the `length` property of the `members` array (if it exists). Again, optional chaining prevents an error if `members` is undefined.
       * `?? 0` is the nullish coalescing operator. If the expression to its left is `null` or `undefined`, it returns `0`. This handles the case where the organization doesn't have any members (or the members property is missing), defaulting the member count to zero.
     * `const pending = org?.invitations?.filter((inv) => inv.status === 'pending').length ?? 0`
       * This line calculates the number of pending invitations.
       * `org?.invitations` safely accesses the `invitations` array, using optional chaining.
       * `.filter((inv) => inv.status === 'pending')` filters the `invitations` array, keeping only the invitations where the `status` property is equal to `'pending'`.  This assumes each invitation object in the `invitations` array has a `status` property.
       * `.length` gets the number of pending invitations.
       * `?? 0` uses the nullish coalescing operator to default the pending count to 0 if `org`, `org.invitations`, or the filtered array is `null` or `undefined`.
     * `return { used: members + pending, members, pending }`
       * Returns an object containing the calculated seat usage information:
         * `used`: The total number of used seats (members + pending invitations).
         * `members`: The number of active members.
         * `pending`: The number of pending invitations.
   * **Output:** An object with `used`, `members`, and `pending` properties, representing seat usage statistics.

**2. `generateSlug(name: string)`**

   * **Purpose:** This function generates a URL-friendly "slug" from a given name.  Slugs are typically used in URLs to represent resources in a human-readable and SEO-friendly way.
   * **Input:** `name: string` - The string from which to generate the slug (e.g., an organization name).
   * **Logic:**
     * `name.toLowerCase()`: Converts the input `name` to lowercase.  This ensures that the slug is case-insensitive.
     * `.replace(/[^a-z0-9]/g, '-')`:  Replaces any character that is *not* a lowercase letter (a-z) or a number (0-9) with a hyphen (`-`).  The `[^...]` inside the regular expression means "not these characters".  The `/g` flag means "replace all occurrences."
     * `.replace(/-+/g, '-')`: Replaces one or more consecutive hyphens with a single hyphen.  This prevents multiple hyphens from appearing next to each other in the slug.
     * `.replace(/^-|-$/g, '')`: Removes any leading or trailing hyphens.  The `^` in the regular expression means "start of the string", and the `$` means "end of the string".
   * **Output:** A URL-friendly slug string.  For example, if the input is "My Company! (v2)", the output will be "my-company-v2".

**3. `validateSlug(slug: string)`**

   * **Purpose:** This function validates whether a given string is a valid slug.
   * **Input:** `slug: string` - The string to validate.
   * **Logic:**
     * `const slugRegex = /^[a-z0-9-_]+$/`: Defines a regular expression that specifies the allowed characters for a valid slug.
       * `^`: Matches the beginning of the string.
       * `[a-z0-9-_]+`: Matches one or more lowercase letters (a-z), numbers (0-9), hyphens (`-`), or underscores (`_`).
       * `$`: Matches the end of the string.
     * `return slugRegex.test(slug)`:  Tests whether the input `slug` matches the regular expression.  The `test()` method returns `true` if the slug is valid (matches the pattern) and `false` otherwise.
   * **Output:** `true` if the slug is valid, `false` otherwise.

**4. `validateEmail(email: string)`**

   * **Purpose:** This function validates whether a given string is a valid email address.
   * **Input:** `email: string` - The email address to validate.
   * **Logic:**
     * `email.trim().toLowerCase()`:  Removes leading and trailing whitespace from the email string using `trim()` and converts the email string to lowercase using `toLowerCase()`. This ensures consistent validation regardless of case or surrounding whitespace.
     * `quickValidateEmail(...)`: Calls the `quickValidateEmail` function (imported from `'@/lib/email/validation'`) to perform the actual email validation. This function likely uses a fast, lightweight algorithm to check the basic format of the email address.
     * `.isValid`: Accesses the `isValid` property of the object returned by `quickValidateEmail`.  This property indicates whether the email address is considered valid by the `quickValidateEmail` function.
   * **Output:** `true` if the email address is valid, `false` otherwise.  It relies on the validation logic within the `quickValidateEmail` function.

**Summary**

This file provides essential utility functions for working with organization data, generating URL-friendly slugs, and validating data formats. The use of optional chaining and nullish coalescing operators makes the code more robust and less prone to errors when dealing with potentially incomplete data.  The use of regular expressions simplifies slug validation.  The separation of email validation logic into an external module (`quickValidateEmail`) promotes code reusability and maintainability.
