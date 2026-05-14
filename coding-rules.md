# Coding Rules & Standards

> **Objective**: Define mandatory standards and conventions for code submission and CI/CD validation
> - **When to use**: Pre-commit checks, code review enforcement, CI/CD validation, linting rules
> - **Not included**: Best practices, optimization guidance, design patterns
> - **Suitable for**: Code enforcement, linting configuration, automated gate validation

---

## SECTION 1: NAMING CONVENTIONS — MANDATORY STANDARDS

From: Project coding standards

### Rule 1.1: Variable Naming (ENFORCED)

**Standard**: Use camelCase for all variables

```php
// ❌ FAIL
$user_age = 25;
$UserEmail = "test@example.com";
$u = User::find($id);

// ✅ PASS
$userAge = 25;
$userEmail = "test@example.com";
$user = User::find($id);
```

**CI/CD Check**:
- Regex: `\$[a-z][a-zA-Z0-9]*` (no underscores)
- Exception: `$_GET`, `$_POST`, `$GLOBALS` (superglobals only)
- Penalty: Auto-fail pull request

---

### Rule 1.2: Method Naming (ENFORCED)

**Standard**: Use camelCase for all methods

```php
// ❌ FAIL
public function get_user_by_email() { }
public function ProcessPayment() { }
private function validate_order_data() { }

// ✅ PASS
public function getUserByEmail() { }
public function processPayment() { }
private function validateOrderData() { }
```

**CI/CD Check**:
- Regex: `function [a-z][a-zA-Z0-9]*\(` (no underscores, lowercase start)
- Penalty: Auto-fail pull request

---

### Rule 1.3: Class Naming (ENFORCED)

**Standard**: Use PascalCase for all classes, interfaces, enums

```php
// ❌ FAIL
class user_service { }
interface payment_processor { }
enum user_status { }

// ✅ PASS
class UserService { }
interface PaymentProcessor { }
enum UserStatus { }
```

**CI/CD Check**:
- Regex: `^class [A-Z][a-zA-Z0-9]*` (uppercase start)
- Applies to: class, interface, trait, enum
- Penalty: Auto-fail pull request

---

### Rule 1.4: Constant Naming (ENFORCED)

**Standard**: Use UPPER_SNAKE_CASE for all constants

```php
// ❌ FAIL
const MaxAttempts = 3;
const maxLoginTime = 3600;
const MAX_ATTEMPTS = 3; // inside class without const prefix

// ✅ PASS
const MAX_ATTEMPTS = 3;
const DEFAULT_DISCOUNT_RATE = 0.2;
const API_TIMEOUT_MS = 5000;
```

**CI/CD Check**:
- Regex: `^const [A-Z_]+` (uppercase + underscores only)
- Penalty: Auto-fail pull request

---

## SECTION 2: CODE STRUCTURE — HARD CONSTRAINTS

From: Clean Code + project requirements

### Rule 2.1: File Size Limit (ENFORCED)

**Standard**: Max 300 lines per file

```php
// ❌ FAIL
class MegaService {
    // 350+ lines of code
}

// ✅ PASS
class UserService {
    // ~250 lines
}

class PaymentService {
    // ~200 lines
}
```

**CI/CD Check**:
- Line count per file: `wc -l < file.php`
- Penalty: Warning at 300 lines, auto-fail at 400+
- Exception: Generated files, migration files

---

### Rule 2.2: Method Size Limit (ENFORCED)

**Standard**: Max 30 lines per method

```php
// ❌ FAIL
public function processOrder($order) {
    // 40+ lines of mixed logic
    // validate, calculate, charge, notify, log
}

// ✅ PASS
public function processOrder(Order $order): OrderResult {
    $this->validateOrder($order);
    $total = $this->calculateTotal($order);
    $payment = $this->processPayment($total);
    $this->notifyCustomer($order, $payment);
    return new OrderResult($order, $payment);
}
```

**CI/CD Check**:
- Count lines between `function` and closing `}`
- Penalty: Warning at 30 lines, auto-fail at 50+
- Exception: Simple getters/setters

---

### Rule 2.3: Nesting Depth Limit (ENFORCED)

**Standard**: Max 3 levels of nesting

```php
// ❌ FAIL - 5 levels
if ($user) {
    if ($user->isActive) {
        if ($user->hasPayment) {
            if ($payment->isValid) {
                if ($order->total > 0) {
                    $this->process();
                }
            }
        }
    }
}

// ✅ PASS - 2 levels
if (!$user || !$user->isActive) return false;
if (!$user->hasPayment) return false;
if (!$payment->isValid) return false;
if ($order->total <= 0) return false;
$this->process();
```

**CI/CD Check**:
- Measure nesting depth per method
- Penalty: Warning at 3, auto-fail at 5+

---

## SECTION 3: ERROR HANDLING — MANDATORY

From: Clean Code + security requirements

### Rule 3.1: Exception Types (ENFORCED)

**Standard**: Must use specific exception types, never generic

```php
// ❌ FAIL
throw new Exception('Invalid email');
throw new RuntimeException('Payment failed');
throw new \stdClass();

// ✅ PASS
throw new InvalidEmailException('Email format invalid: ' . $email);
throw new PaymentFailedException('Card declined: ' . $errorCode);
throw new OrderNotFoundException('Order #' . $orderId . ' not found');
```

**CI/CD Check**:
- Scan for: `throw new Exception`, `throw new RuntimeException`
- Penalty: Auto-fail pull request
- Requirement: Must have custom exception classes for domain

---

### Rule 3.2: Exception Context (ENFORCED)

**Standard**: All exceptions must include context

```php
// ❌ FAIL
throw new InvalidAgeException();
throw new PaymentFailedException("Failed");

// ✅ PASS
throw new InvalidAgeException(
    "Age validation failed: expected 18-150, got $age"
);
throw new PaymentFailedException(
    "Payment declined for order #{$orderId}: {$errorCode}. " .
    "Possible causes: insufficient funds, expired card, fraud block"
);
```

**CI/CD Check**:
- Parse exception constructor calls
- Penalty: Warning if message < 20 chars, auto-fail if empty

---

### Rule 3.3: Input Validation (ENFORCED)

**Standard**: All public method parameters must be validated at entry point

```php
// ❌ FAIL
public function setUserAge($age) {
    $this->age = $age; // No validation!
}

// ✅ PASS
public function setUserAge(int $age): void {
    if ($age < 0 || $age > 150) {
        throw new InvalidAgeException(
            "Age must be 0-150, got $age"
        );
    }
    $this->age = $age;
}
```

**CI/CD Check**:
- Flag public methods without type hints
- Penalty: Warning for missing validation, auto-fail for direct assignment

---

## SECTION 4: TESTING REQUIREMENTS — MANDATORY

From: Quality assurance + code coverage requirements

### Rule 4.1: Code Coverage Minimum (ENFORCED)

**Standard**: Min 70% overall, 100% for critical paths

```php
// Test Coverage Requirements:
public function validateEmail($email) { }
    // ✅ MUST have unit test
    
public function processPayment($order) { }
    // ✅ MUST have unit test (critical path)
    // ✅ MUST have integration test
    
private function formatDate($date) { }
    // ⚠️ Can skip (private, simple)
```

**CI/CD Check**:
- Run: `phpunit --coverage-html`
- Penalty: Warning < 70%, auto-fail < 60%
- Critical paths: Auto-fail < 95%

---

### Rule 4.2: Test Naming (ENFORCED)

**Standard**: Test names must follow pattern: `test<Subject><Condition><Expected>`

```php
// ❌ FAIL
public function testUser() { }
public function test1() { }
public function testEmail() { }

// ✅ PASS
public function testValidateEmailAcceptsStandardFormat() { }
public function testValidateEmailRejectsMissingDomain() { }
public function testUserCannotAccessAdminWithoutRole() { }
public function testPaymentFailsWhenCardDeclined() { }
```

**CI/CD Check**:
- Regex: `test[A-Z][a-zA-Z]+` (must start with capital after 'test')
- Penalty: Warning for unclear names, auto-fail for `test1`, `testX`

---

### Rule 4.3: New Code Requires Tests (ENFORCED)

**Standard**: All new public methods must have corresponding tests

```php
// When submitting PR:
// - Added: public function getUserPosts() { }
//   → MUST have: public function testGetUserPostsReturnsArrayOfPosts() { }

// - Modified: private function formatDate() { }
//   → Tests optional (private method)
```

**CI/CD Check**:
- Detect new public methods: `git diff HEAD -- *.php | grep 'public function'`
- Verify tests exist for new methods
- Penalty: Auto-fail if no matching test

---

## SECTION 5: DOCUMENTATION — MANDATORY

From: Code maintainability + team communication

### Rule 5.1: Class Docblocks (ENFORCED)

**Standard**: Every class must have docblock with @author, @version, @description

```php
// ❌ FAIL
class UserService {
    public function create($data) { }
}

// ✅ PASS
/**
 * Manages user account creation, updates, and deletions
 * 
 * Responsibilities:
 * - Validate user data
 * - Persist to database
 * - Notify related services
 * 
 * @author John Doe
 * @version 1.0
 * @since 2024-01-15
 */
class UserService {
    public function create(array $data): User { }
}
```

**CI/CD Check**:
- Regex: `class [A-Z]` must have `/**` above it
- Penalty: Auto-fail if docblock missing

---

### Rule 5.2: Public Method Docblocks (ENFORCED)

**Standard**: Every public method must have @param, @return, @throws documentation

```php
// ❌ FAIL
public function getUserById($id) { }

// ✅ PASS
/**
 * Retrieve user by ID
 * 
 * @param int $userId User's unique identifier
 * @return User User object with all properties
 * @throws UserNotFoundException If user does not exist
 */
public function getUserById(int $userId): User { }
```

**CI/CD Check**:
- Check all `public function` declarations
- Require: `@param`, `@return`, `@throws` (if applicable)
- Penalty: Auto-fail if missing

---

### Rule 5.3: Type Hints (ENFORCED)

**Standard**: All parameters and return types must have explicit type hints

```php
// ❌ FAIL
public function processOrder($order) {
    return $result;
}

// ✅ PASS
public function processOrder(Order $order): OrderResult {
    return new OrderResult($order);
}
```

**CI/CD Check**:
- Parse function signatures
- Detect missing type hints
- Penalty: Warning for missing hints, auto-fail for public methods without hints

---

## SECTION 6: CODE QUALITY GATES — HARD CONSTRAINTS

From: Technical debt prevention + quality baseline

### Rule 6.1: No Code Duplication (ENFORCED)

**Standard**: Same logic cannot appear 2+ times without extraction

```php
// ❌ FAIL - Duplicate validation
public function createUser($email) {
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        throw new InvalidEmailException();
    }
}

public function updateUserEmail($email) {
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        throw new InvalidEmailException();
    }
}

// ✅ PASS - Extract to validator
class EmailValidator {
    public function validate(string $email): void {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidEmailException();
        }
    }
}
```

**CI/DC Check**:
- Run: `phpcpd src/` (PHP Copy/Paste Detector)
- Threshold: Max 5% duplication
- Penalty: Auto-fail if > 10% duplication

---

### Rule 6.2: No Magic Numbers (ENFORCED)

**Standard**: All hardcoded numbers must be named constants

```php
// ❌ FAIL
if ($age > 18) { }
if ($attempts > 3) { }
$discount = $price * 0.2;

// ✅ PASS
const ADULT_AGE_THRESHOLD = 18;
const MAX_LOGIN_ATTEMPTS = 3;
const DEFAULT_DISCOUNT_RATE = 0.2;

if ($age > self::ADULT_AGE_THRESHOLD) { }
if ($attempts > self::MAX_LOGIN_ATTEMPTS) { }
$discount = $price * self::DEFAULT_DISCOUNT_RATE;
```

**CI/CD Check**:
- Scan for: `> 1`, `< 100`, `* 0.2`, etc. in conditional/assignment
- Exception: Loop counters (0, 1), array offsets
- Penalty: Warning for each magic number

---

### Rule 6.3: No Commented Code (ENFORCED)

**Standard**: Never commit commented-out code

```php
// ❌ FAIL
public function process() {
    // $this->oldLogic();
    // $result = calculate($data);
    $this->newLogic();
}

// ✅ PASS
public function process() {
    $this->newLogic();
}
// Git history has the old code - no need to keep it commented
```

**CI/CD Check**:
- Scan for: Lines starting with `//` containing `= `, `;`, `()`
- Penalty: Auto-fail if > 3 lines of commented code

---

## SECTION 7: SECURITY RULES — MANDATORY

From: Security best practices + OWASP guidelines

### Rule 7.1: No Hardcoded Credentials (ENFORCED)

**Standard**: Never hardcode passwords, API keys, or secrets

```php
// ❌ FAIL
const DB_PASSWORD = "myPassword123";
$apiKey = "sk_live_abc123def456";
$this->stripe->setApiKey("sk_test_secret");

// ✅ PASS
const DB_PASSWORD = getenv('DB_PASSWORD');
$apiKey = config('stripe.api_key');
$this->stripe->setApiKey(env('STRIPE_SECRET_KEY'));
```

**CI/CD Check**:
- Regex scan: `password`, `apiKey`, `secret`, `token` assignments
- Exclude: Configuration files, .env files, comments
- Penalty: Auto-fail + alert security team

---

### Rule 7.2: Input Validation (ENFORCED)

**Standard**: All external input must be validated at system boundary

```php
// ❌ FAIL
public function submitForm($data) {
    $user = User::where('email', $data['email'])->first();
}

// ✅ PASS
public function submitForm(array $data): User {
    $validated = validator($data, [
        'email' => 'required|email',
        'age' => 'required|integer|min:0|max:150'
    ])->validate();
    
    return User::where('email', $validated['email'])->first();
}
```

**CI/CD Check**:
- Flag: Direct use of `$_GET`, `$_POST`, `$request->input()`
- Require: Validation before use
- Penalty: Warning for unvalidated input

---

### Rule 7.3: SQL Injection Prevention (ENFORCED)

**Standard**: Never use string concatenation in SQL queries

```php
// ❌ FAIL - SQL Injection vulnerability
$user = DB::select("SELECT * FROM users WHERE email = '" . $email . "'");

// ✅ PASS - Parameterized queries
$user = User::where('email', $email)->first();
$users = DB::select("SELECT * FROM users WHERE email = ?", [$email]);
$users = DB::select("SELECT * FROM users WHERE email = :email", ['email' => $email]);
```

**CI/CD Check**:
- Scan for: `DB::select(` with `"` containing `.`
- Flag: String concatenation in queries
- Penalty: Auto-fail + alert security team

---

## REFERENCE GUIDE FOR CI/CD ENFORCEMENT

### Penalty Levels

| Level | Action | Recovery |
|-------|--------|----------|
| **Info** | Display in PR comment | No action needed |
| **Warning** | Display in CI logs | Can merge with approval |
| **Auto-fail** | Block PR merge | Must fix before resubmit |
| **Alert** | Notify security team | Manual review required |

### CI/CD Tools Configuration

```yaml
# .gitlab-ci.yml or similar
quality_gates:
  naming_conventions:
    tool: phpcs
    standard: PSR12-camelCase
    fail_threshold: 0 errors

  code_coverage:
    tool: phpunit
    minimum: 70%
    critical_paths: 100%

  duplication:
    tool: phpcpd
    threshold: 5%

  security:
    tool: phpstan
    level: 9

  documentation:
    tool: custom-docblock-check
    requirement: all public methods
```

### Auto-Fix Tools

```bash
# Can be auto-fixed by pre-commit hook:
phpcbf --standard=PSR12 src/          # Fix naming
```

---

## SUMMARY

**Focus**: Enforce mandatory coding standards through CI/CD automation  
**7 sections, 15 rules**: Naming, Structure, Error Handling, Testing, Documentation, Quality, Security  
**For AI**: Code review enforcement, linting configuration, CI/CD gate validation

---

## CI/CD INTEGRATION CHECKLIST

```
Before merging PR, verify:
☑ All naming conventions pass (variables, methods, classes, constants)
☑ File size < 300 lines
☑ Method size < 30 lines
☑ Nesting depth ≤ 3 levels
☑ No generic exceptions
☑ All exceptions have context
☑ Public methods have type hints
☑ New public methods have tests
☑ Code coverage ≥ 70%
☑ No code duplication > 5%
☑ No magic numbers
☑ No commented-out code
☑ No hardcoded credentials
☑ All input validated
☑ No SQL injection vulnerabilities
☑ All classes have docblocks
☑ All public methods have docblocks
```

---
