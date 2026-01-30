# fix/literal-password-heuristic

## Problem

`no-literal-password` and `no-insecure-comparison` rules use a name-based heuristic
(`is_password()` in `crates/linter/src/rule/utils/security.rs`) that matches any identifier
ending with `password`, `token`, `secret`, `apikey`, or `api_key` (case-insensitive).

This produces false positives when those words appear in contexts unrelated to credentials:

### `no-literal-password` false positives (10/10)

```php
// OAuth grant type name — not a credential
const PASSWORD = 'password';           // in GrantType enum
const CLIENT_CREDENTIALS = 'client_credentials';

// UI label / form field name
'resetPassword' => 'Reset Password'    // translation key

// Error code / validation key
'invalidToken' => 'Token is invalid'   // error message mapping
```

### `no-insecure-comparison` false positives (8/10)

```php
// Enum comparison — GrantType::Password is not a credential
$grantType === GrantType::Password

// String key comparison — not a credential value
$action === 'resetPassword'
```

## Cause

`crates/linter/src/rule/utils/security.rs` lines 161-191:

`is_password()` performs suffix matching only. It does not consider:

1. **Enum cases** — `GrantType::Password` is a type discriminant, not a credential
2. **String values** — Only the variable/constant _name_ is checked, not the assigned _value_.
   A constant named `PASSWORD` with value `'password'` (an OAuth grant type string) is not
   a hardcoded credential.
3. **Context** — Whether the name is a translation key, error code, form field name, etc.

## Solution

Add exclusions to reduce false positives without weakening the security check:

1. **Skip enum case declarations**: When the password-like name is an enum case name,
   it is a type discriminant, not a credential storage.
   - In `no_literal_password.rs` check, skip `ClassLikeConstantItem` nodes inside an enum.

2. **Skip class constant comparisons on enum types**: In `no_insecure_comparison.rs`,
   when `get_password()` matches a class constant access on an enum, skip it.

3. **Value heuristic (optional, lower priority)**: If the assigned string value exactly
   matches one of the keywords (e.g., `const PASSWORD = 'password'`), it is likely a
   type discriminant (like OAuth grant type), not a real credential. Real credentials
   are high-entropy strings. This is heuristic but would catch the most common FP pattern.

## Tasks

- [ ] In `no_literal_password.rs`: skip reporting when the node is an enum case declaration
- [ ] In `no_insecure_comparison.rs`: skip when `get_password()` matches an enum class constant
- [ ] Add test: enum case named `Password` should NOT be flagged by `no-literal-password`
- [ ] Add test: `GrantType::Password === $x` should NOT be flagged by `no-insecure-comparison`
- [ ] Add test: `$password = 'hunter2'` should still be flagged (true positive)
- [ ] Run existing tests

## Verification

- [ ] `cargo test -p mago-linter` passes
- [ ] Enum case `Password = 'password'` is NOT reported by `no-literal-password`
- [ ] `GrantType::Password === $input` is NOT reported by `no-insecure-comparison`
- [ ] `$password = 'secret123'` IS still reported by `no-literal-password`
- [ ] `$password == $input` IS still reported by `no-insecure-comparison`
