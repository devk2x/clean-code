# Clean Code Reference

> **Objective**: Focus on designing code that is **clean, maintainable, and well-architected**
> - **When to use**: Code review for design, refactor to improve architecture, audit OOP patterns
> - **Not included**: Code formatting, performance tuning, development process
> - **Suitable for**: Design review, class/function structure, error handling strategy, refactoring

---

## SECTION 1: FUNCTIONS — SMALL, FOCUSED, TESTABLE

Consolidated from: "Readable Code" + "Clean Code"

### Pattern 1.1: Single Responsibility — One Reason to Change

**Problem**: Functions do multiple things, hard to test  
**Solution**: One concern per function, ~10-20 lines  

```php
// ANTI-PATTERN: 50+ lines, 5 reasons to change
public function registerUser($data) {
    // Validate
    if (!$data['email']) throw new Exception('Email required');
    // Create DB record
    $user = new User($data);
    $user->save();
    // Send email
    Mail::to($data['email'])->send(new WelcomeMail($user));
    // Update cache
    cache()->put("user:{$user->id}", $user);
    // Log
    Log::info('User registered');
    return $user;
}

// BEST-PRACTICE: Split into focused functions
public function registerNewUser(array $data): User {
    $validated = $this->validateRegistrationData($data);
    $user = $this->createUserInDatabase($validated);
    $this->notifyUserOfRegistration($user);
    return $user;
}

private function validateRegistrationData(array $data): array {
    if (empty($data['email'])) throw new InvalidEmailException();
    if (empty($data['password'])) throw new InvalidPasswordException();
    return $data;
}

private function createUserInDatabase(array $data): User {
    $user = new User($data);
    $user->save();
    return $user;
}

private function notifyUserOfRegistration(User $user): void {
    Mail::to($user->email)->send(new WelcomeMail($user));
}
```

**AI Decision Rule**:
- 1 function = 1 reason to change
- Aim: 10-20 lines (max 30)
- Extract when: "and", "then", "or" in function description

---

### Pattern 1.2: Parameters — Max 3, Group Related

**Problem**: Too many parameters = confusing  
**Solution**: Use value objects, DTOs  

```php
// ANTI-PATTERN
public function createInvoice(
    $userId, $amount, $currency, $taxRate,
    $discount, $isDraft, $sendEmail, $memo,
    $dueDays, $referenceId
) { }

// BEST-PRACTICE
public function createInvoice(int $userId, InvoiceData $data): Invoice {
    return Invoice::create([
        'user_id' => $userId,
        'amount' => $data->amount,
        'currency' => $data->currency,
        'tax_rate' => $data->tax_rate,
        'discount' => $data->discount,
        'is_draft' => $data->is_draft,
        'due_days' => $data->due_days,
    ]);
}

class InvoiceData {
    public function __construct(
        public float $amount,
        public string $currency = 'USD',
        public float $taxRate = 0.0,
        public float $discount = 0.0,
        public bool $isDraft = false,
        public int $dueDays = 30,
    ) {}
}
```

**AI Decision Rule**:
- Max 3 scalar parameters
- >3 params = create data object
- Group related concepts (Address, PaymentData, etc)

---

## SECTION 2: ERROR HANDLING — DEFENSIVE PROGRAMMING

Consolidated from: "Clean Code" + "Code Complete"

### Pattern 2.1: Specific Exceptions + Context

**Problem**: Generic exceptions lose information  
**Solution**: Domain-specific exceptions with detail  

```php
// ANTI-PATTERN
throw new Exception('Error');
throw new RuntimeException('Invalid');

// BEST-PRACTICE: Specific exception, full context
throw new InvalidEmailException(
    "Email validation failed: expected 'user@domain.com' format, " .
    "but got '$email'. Please check spelling and try again."
);

throw new PaymentFailedException(
    "Payment processing for order #{$order->id} failed: " .
    "Card declined with code {$errorCode}. " .
    "Possible reasons: insufficient funds, expired card, fraud block. " .
    "Please update payment method and retry."
);
```

**AI Decision Rule**:
- Create domain-specific exceptions
- Include: context, actual value, expected value, remedy
- Format: `What | Why | How to fix`

---

### Pattern 2.2: Defensive Programming — Validate at Boundaries

**Problem**: Assuming external input is valid leads to bugs  
**Solution**: Validate at system boundaries, not internally  

From "Code Complete" — validate external input

```php
// ANTI-PATTERN: Assume input is valid
public function processUserAge($age) {
    return $age + 1;
}

// BEST-PRACTICE: Validate at entry point
public function setUserAge(int $age): void {
    const MIN_AGE = 0;
    const MAX_AGE = 150;
    
    if ($age < self::MIN_AGE || $age > self::MAX_AGE) {
        throw new InvalidAgeException(
            "Age must be between {MIN_AGE} and {MAX_AGE}, got $age"
        );
    }
    
    $this->age = $age;
}

// Design by contract: specify preconditions
public function withdrawFunds(float $amount): void {
    // Precondition: amount must be positive and available
    if ($amount <= 0) {
        throw new InvalidWithdrawAmount('Amount must be positive');
    }
    if ($amount > $this->balance) {
        throw new InsufficientFundsException(
            "Cannot withdraw $amount, balance is {$this->balance}"
        );
    }
    
    $this->balance -= $amount;
}
```

**AI Decision Rule**:
- Validate at system boundaries (user input, API responses)
- Use assertions for internal assumptions
- Design by contract: specify preconditions

---

## SECTION 3: CLASSES — DESIGN & RESPONSIBILITY & COMMON MISTAKES

Consolidated from: "Clean Code"

### Pattern 3.1: Single Responsibility — One Reason to Change

**Problem**: Class does multiple things  
**Solution**: Each class = one responsibility  

```php
// ANTI-PATTERN: Class with 5 reasons to change
class User {
    public function validateEmail($email) { }
    public function hashPassword($pwd) { }
    public function saveToDatabase() { }
    public function sendEmail() { }
    public function generateJwtToken() { }
}

// BEST-PRACTICE: Split responsibilities
class User {
    public function Construct(
        private UserValidator $validator,
        private UserRepository $repository,
    ) {}
    
    public function create(array $data): User {
        $this->validator->validate($data);
        return $this->repository->save(new User($data['email']));
    }
}

class UserValidator {
    public function validate(array $data): void {
        if (!filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
            throw new InvalidEmailException();
        }
    }
}

class PasswordHasher {
    public function hash(string $password): string {
        return password_hash($password, PASSWORD_BCRYPT);
    }
}
```

**AI Decision Rule**:
- Count responsibilities: >3 = signal to split
- One reason to change = tight, focused class
- Easier to test, reuse, modify

---

### Pattern 3.2: Encapsulation — Hide Implementation

**Problem**: Exposing internal structure  
**Solution**: Private fields, public methods  

```php
// ANTI-PATTERN
class BankAccount {
    public $balance = 0;
    public $transactions = [];
}

$account->balance = -1000; // No validation!

// BEST-PRACTICE
class BankAccount {
    private float $balance = 0;
    private array $transactions = [];
    
    public function deposit(float $amount): void {
        if ($amount <= 0) {
            throw new InvalidAmountException();
        }
        $this->balance += $amount;
        $this->recordTransaction('deposit', $amount);
    }
    
    public function getBalance(): float {
        return $this->balance;
    }
}
```

**AI Decision Rule**:
- Private by default, public only if needed
- Expose intent (methods), not implementation (fields)
- Protect invariants (balance never negative)

---

### Pattern 3.3: Common OOP Mistakes — Inheritance, Type Hints, Composition

**Problem**: Misuse of inheritance, missing type hints, tight coupling  
**Solution**: Use composition, add type hints, decouple  

```php
// ANTI-PATTERN 1: Deep inheritance hierarchy (fragile base class)
class Animal {
    public function move() { }
}
class Mammal extends Animal {
    public function produceMilk() { }
}
class Dog extends Mammal {
    public function bark() { }
}
class Cat extends Mammal {
    public function meow() { }
}
// Problem: Changes to Mammal affect Dog & Cat unexpectedly

// BEST-PRACTICE 1: Use composition instead
interface Animal {
    public function move(): void;
}

interface Mammal {
    public function produceMilk(): void;
}

class Dog implements Animal {
    private AnimalBehavior $behavior;
    private MilkProduction $lactation;
    
    public function move(): void { $this->behavior->move(); }
    public function produceMilk(): void { $this->lactation->produce(); }
    public function bark(): void { /* bark logic */ }
}

// ANTI-PATTERN 2: Missing type hints
class OrderProcessor {
    public function process($order, $payment) {
        return $this->calculate($order) + $payment;
    }
}

// BEST-PRACTICE 2: Add type hints everywhere
class OrderProcessor {
    public function process(Order $order, Payment $payment): float {
        return $this->calculateTotal($order) + $payment->amount();
    }
    
    private function calculateTotal(Order $order): float {
        return $order->subtotal() * (1 + Order::TAX_RATE);
    }
}

// ANTI-PATTERN 3: Circular dependencies
class UserService {
    private OrderService $orderService;
}
class OrderService {
    private UserService $userService; // Circular!
}

// BEST-PRACTICE 3: Inject dependencies, break cycles
class UserService {
    private OrderRepository $orders; // Use repository, not service
}
class OrderService {
    private UserRepository $users;
}

// ANTI-PATTERN 4: God Object - too many responsibilities
class User {
    public function validate() { }
    public function hashPassword() { }
    public function saveToDb() { }
    public function sendEmail() { }
    public function generateToken() { }
    public function createInvoice() { }
    public function calculateTaxes() { }
    // Too many reasons to change!
}

// BEST-PRACTICE 4: Split into focused classes
class User {
    public function Construct(
        public string $email,
        private string $passwordHash,
    ) {}
}

class UserValidator {
    public function validate(array $data): void { }
}

class PasswordHasher {
    public function hash(string $password): string { }
}

class UserRepository {
    public function save(User $user): void { }
}

class UserNotifier {
    public function sendWelcomeEmail(User $user): void { }
}

class AuthTokenGenerator {
    public function generateFor(User $user): string { }
}
```

**AI Decision Rule**:
- Use composition over inheritance (simplifies design)
- Always use type hints (properties, parameters, return types)
- Inject dependencies (avoid circular deps, improves testability)
- Max 3-5 methods per class (signal: split if more)
- Private fields + public methods (encapsulation)
- Each class = 1 reason to change (SRP)

---

## SECTION 4: SOLID PRINCIPLES — DESIGN FOUNDATIONS

Consolidated from: "Clean Code" + "Code Complete"

> **SOLID** = 5 core design principles ensuring code flexibility, maintainability, extensibility

### Pattern 4.1: O — Open/Closed Principle

**Problem**: Adding new features requires modifying existing code  
**Solution**: Open for extension, closed for modification (use polymorphism)  

```php
// ANTI-PATTERN: Closed for extension (switch on types)
public function calculateSalary(Employee $emp): float {
    switch ($emp->type) {
        case 'engineer':
            return $emp->base_salary + $emp->bonus;
        case 'manager':
            return $emp->base_salary + $emp->bonus + $emp->allowance;
        case 'intern':
            return $emp->base_salary;
    }
    return 0;
}
// Adding new employee type requires modifying this function

// BEST-PRACTICE: Open for extension (polymorphism)
interface Employee {
    public function calculateSalary(): float;
}

class Engineer implements Employee {
    public function Construct(
        private float $baseSalary,
        private float $bonus,
    ) {}
    
    public function calculateSalary(): float {
        return $this->base_salary + $this->bonus;
    }
}

class Manager implements Employee {
    public function Construct(
        private float $baseSalary,
        private float $bonus,
        private float $allowance,
    ) {}
    
    public function calculateSalary(): float {
        return $this->base_salary + $this->bonus + $this->allowance;
    }
}

// New employee type = create new class, no modification needed
$employees = [$engineer, $manager, $intern];
$totalSalary = array_sum(array_map(fn($e) => $e->calculateSalary(), $employees));
```

**AI Decision Rule**:
- Switch statements on types = OCP violation
- Use polymorphism: interface → implementations
- New behavior = new class, not code modification

---

### Pattern 4.2: L — Liskov Substitution Principle

**Problem**: Subtype breaks behavior of parent type  
**Solution**: Subtypes must be substitutable for parent type  

```php
// ANTI-PATTERN: Violates LSP
class Bird {
    public function fly(): void {
        echo "Flying...";
    }
}

class Penguin extends Bird {
    public function fly(): void {
        throw new Exception("Penguins cannot fly!");
    }
}

// This breaks LSP: can't substitute Penguin for Bird
function make_bird_fly(Bird $bird) {
    $bird->fly(); // Crashes if penguin passed!
}

// BEST-PRACTICE: Use composition, not inheritance
interface Animal {
    public function move(): void;
}

interface Flyer {
    public function fly(): void;
}

class Bird implements Animal, Flyer {
    public function move(): void { echo "Moving..."; }
    public function fly(): void { echo "Flying..."; }
}

class Penguin implements Animal {
    public function move(): void { echo "Waddling..."; }
    // Doesn't implement Flyer - no false contract
}

// Now we can safely check capabilities
function make_animal_fly(Flyer $flyer) {
    $flyer->fly(); // Safe - only things that truly fly
}
```

**AI Decision Rule**:
- Subclass shouldn't violate parent behavior
- Use interfaces (contracts) over inheritance
- If subclass can't fulfill contract, don't extend

---

### Pattern 4.3: I — Interface Segregation Principle

**Problem**: Client forced to depend on methods it doesn't use  
**Solution**: Split fat interface into smaller, focused interfaces  

```php
// ANTI-PATTERN: Fat interface
interface Worker {
    public function work(): void;
    public function eat(): void;
    public function sleep(): void;
}

class Robot implements Worker {
    public function work(): void { echo "Working..."; }
    public function eat(): void { /* Robot doesn't eat! */ }
    public function sleep(): void { /* Robot doesn't sleep! */ }
}

// Robot forced to implement eat() and sleep() even though it doesn't need them

// BEST-PRACTICE: Segregated interfaces
interface Workable {
    public function work(): void;
}

interface Eatable {
    public function eat(): void;
}

interface Sleepable {
    public function sleep(): void;
}

class Robot implements Workable {
    public function work(): void { echo "Working..."; }
}

class Human implements Workable, Eatable, Sleepable {
    public function work(): void { echo "Working..."; }
    public function eat(): void { echo "Eating..."; }
    public function sleep(): void { echo "Sleeping..."; }
}

// Robot only implements what it needs
// Clients depend on specific interfaces, not fat ones
```

**AI Decision Rule**:
- Fat interface = multiple unrelated methods
- Split into smaller, focused interfaces
- Clients depend on what they actually use

---

### Pattern 4.4: D — Dependency Inversion Principle

**Problem**: High-level code depends on low-level implementations  
**Solution**: Both depend on abstractions (interfaces)  

```php
// ANTI-PATTERN: High-level depends on low-level
class PaymentProcessor {
    private StripeGateway $stripe; // Concrete dependency
    
    public function processPayment(float $amount): void {
        $this->stripe->charge($amount); // Tied to Stripe
    }
}

// If we want to use PayPal, must modify PaymentProcessor

// BEST-PRACTICE: Depend on abstraction
interface PaymentGateway {
    public function charge(float $amount): PaymentResult;
}

class StripeGateway implements PaymentGateway {
    public function charge(float $amount): PaymentResult { /* ... */ }
}

class PayPalGateway implements PaymentGateway {
    public function charge(float $amount): PaymentResult { /* ... */ }
}

class PaymentProcessor {
    private PaymentGateway $gateway; // Abstract dependency
    
    public function Construct(PaymentGateway $gateway) {
        $this->gateway = $gateway; // Injected
    }
    
    public function processPayment(float $amount): void {
        $this->gateway->charge($amount); // Works with any gateway
    }
}

// Can swap gateways without modifying PaymentProcessor
$processor = new PaymentProcessor(new StripeGateway());
// or
$processor = new PaymentProcessor(new PayPalGateway());
```

**AI Decision Rule**:
- Don't inject concrete classes (StripeGateway)
- Inject interfaces (PaymentGateway)
- High-level ↔ Low-level both depend on abstraction

---

### Pattern 4.5: S — Single Responsibility Principle (SOLID Recap)

**Problem**: Class/function with multiple reasons to change  
**Solution**: One responsibility = one reason to change  

```php
// ANTI-PATTERN: Multiple responsibilities
class User {
    public function validateEmail($email) { } // Responsibility 1
    public function hashPassword($pwd) { }    // Responsibility 2
    public function saveToDb() { }           // Responsibility 3
    public function sendEmail() { }           // Responsibility 4
}
// 4 reasons to change this class

// BEST-PRACTICE: One responsibility per class
class User {
    public function Construct(
        public string $email,
        private string $passwordHash,
    ) {}
}

class UserValidator {
    public function validateEmail($email) { } // Responsibility: validation
}

class PasswordHasher {
    public function hash($password) { } // Responsibility: hashing
}

class UserRepository {
    public function save(User $user) { } // Responsibility: persistence
}

class UserNotifier {
    public function sendEmail(User $user) { } // Responsibility: notification
}
// Each class has ONE reason to change
```

**AI Decision Rule**:
- Count reasons to change: >3 = split class
- One responsibility = tight, testable, reusable
- Each class = single reason to change

---

## SECTION 5: DATA STRUCTURES & VARIABLES — TABLE-DRIVEN DESIGN

Consolidated from: "Readable Code" + "Code Complete"

### Pattern 5.2: Table-Driven Methods (Code Complete)

**Problem**: Logic scattered in if/switch statements  
**Solution**: Use data structures instead  

```php
// ANTI-PATTERN: Logic in conditionals
public function getDiscountRate(string $customerType) {
    if ($customerType === 'gold') {
        return 0.20;
    } elseif($customerType === 'silver') {
        return 0.15;
    } elseif($customerType === 'bronze') {
        return 0.10;
    }
    return 0;
}

// BEST-PRACTICE: Table-driven
private const DISCOUNT_RATES = [
    'gold' => 0.20,
    'silver' => 0.15,
    'bronze' => 0.10,
    'default' => 0,
];

public function getDiscountRate(string $customerType): float {
    return self::DISCOUNT_RATES[$customerType] ?? self::DISCOUNT_RATES['default'];
}

// More complex: table with objects
private const TIER_CONFIGS = [
    'free' => ['max_storage' => 1, 'max_users' => 1, 'features' => []],
    'pro' => ['max_storage' => 100, 'max_users' => 10, 'features' => ['api', 'sso']],
    'enterprise' => ['max_storage' => 9999, 'max_users' => 9999, 'features' => ['api', 'sso', 'custom']],
];

public function getTierConfig(string $tier): array {
    return self::TIER_CONFIGS[$tier] ?? self::TIER_CONFIGS['free'];
}
```

**AI Decision Rule**:
- Use tables for static mappings
- Avoid switch/if for enumerated values
- Better: data instead of logic

---

## SECTION 6: CODE SMELLS — RED FLAGS TO REFACTOR

Consolidated from: "Clean Code"

### Red Flags Summary

**Duplicate Code**: Same logic 2+ places → Extract to function  
**Long Methods**: >20 lines → Break into focused functions  
**Long Classes**: >300 lines OR >3 responsibilities → Split class  
**Long Parameter Lists**: >3 parameters → Use DTO/value object  
**Magic Numbers**: Hardcoded values → Named constants  
**Dead Code**: Unused functions/branches → Delete (version control has history)  
**Feature Envy**: Method uses other class's getters more than own → Move logic  
**Shotgun Surgery**: Change requires updates in many places → Consolidate related code  
**Switch Statements**: On type/status → Use polymorphism or strategy pattern  

### Pattern 6.1: Duplicate Code

**Problem**: Same validation/logic scattered across multiple methods  
**Solution**: Extract to reusable validator class  

```php
// ANTI-PATTERN: Duplicate validation
class UserService {
    public function create($email, $password) {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) throw new Exception();
    }
    
    public function updateEmail($email) {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) throw new Exception();
    }
}

// BEST-PRACTICE: Extract to reusable validator
class UserValidator {
    public function validateEmail($email) {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidEmailException();
        }
    }
}

class UserService {
    public function Construct(private UserValidator $validator) {}
    
    public function create($email, $password) {
        $this->validator->validateEmail($email);
    }
    
    public function updateEmail($email) {
        $this->validator->validateEmail($email);
    }
}
```

**AI Decision Rule**:
- Flag if same logic appears 2+ places
- Extract to reusable function/class
- DRY = prevents bugs from partial updates

---

### Pattern 6.2: Switch Statements on Types

**Problem**: Switch statements on type require code modification for new types (OCP violation)  
**Solution**: Use polymorphism/strategy pattern  

```php
// ANTI-PATTERN: Requires change when new types added
public function processPayment($method, $amount) {
    switch ($method) {
        case 'credit_card':
            return $this->processCreditCard($amount);
        case 'paypal':
            return $this->processPaypal($amount);
        // Must add case for every new payment method
    }
}

// BEST-PRACTICE: Use strategy pattern
interface PaymentProcessor {
    public function process(float $amount): PaymentResult;
}

class CreditCardProcessor implements PaymentProcessor {
    public function process(float $amount): PaymentResult { /* ... */ }
}

class PayPalProcessor implements PaymentProcessor {
    public function process(float $amount): PaymentResult { /* ... */ }
}

public function processPayment(PaymentProcessor $processor, float $amount): PaymentResult {
    return $processor->process($amount);
}
// New payment methods: create new class, no code modification needed
```

**AI Decision Rule**:
- Switch on type/status that changes = maintenance burden
- Use polymorphism or strategy pattern instead
- New types don't require code modification

---

### Pattern 6.3: Magic Numbers

**Problem**: Hardcoded values without semantic meaning hide intent  
**Solution**: Extract to named constants  

```php
// ANTI-PATTERN: Magic numbers
if ($age > 18) { }
if ($attempts > 3) { }
$discount = $price * 0.2;
sleep(5000);

// BEST-PRACTICE: Named constants with intent
const ADULT_AGE_THRESHOLD = 18;
const MAX_LOGIN_ATTEMPTS = 3;
const DEFAULT_DISCOUNT_RATE = 0.2;
const API_TIMEOUT_MS = 5000;

if ($age > self::ADULT_AGE_THRESHOLD) { }
if ($attempts > self::MAX_LOGIN_ATTEMPTS) { }
$discount = $price * self::DEFAULT_DISCOUNT_RATE;
sleep(self::API_TIMEOUT_MS);
```

**AI Decision Rule**:
- Flag hardcoded numbers without semantic context
- Extract to named constants (reveals intent)
- Critical for maintenance and requirement changes

---

## SECTION 7: BOUNDARIES — EXTERNAL CODE & INTEGRATION

Consolidated from: "Clean Code"

### Pattern 7.1: Wrap Third-Party Libraries

**Problem**: Direct dependency on external library  
**Solution**: Adapter pattern isolation  

```php
// ANTI-PATTERN: Direct Stripe dependency
class OrderProcessor {
    public function process(Order $order) {
        $charge = Stripe::charge($order->total);
    }
}

// BEST-PRACTICE: Wrapped adapter
interface PaymentGateway {
    public function charge(float $amount): PaymentResult;
}

class StripePaymentGateway implements PaymentGateway {
    private Stripe $stripe;
    
    public function charge(float $amount): PaymentResult {
        try {
            $charge = $this->stripe->charges->create([
                'amount' => intval($amount * 100),
                'currency' => 'usd',
            ]);
            return new PaymentResult(true, $charge->id);
        } catch (CardException $e) {
            return new PaymentResult(false, null, $e->getMessage());
        }
    }
}

class OrderProcessor {
    private PaymentGateway $gateway;
    
    public function process(Order $order): void {
        $result = $this->gateway->charge($order->total);
        if (!$result->isSuccessful()) {
            throw new PaymentFailedException($result->error());
        }
    }
}
```

**AI Decision Rule**:
- Always wrap external libraries in adapters
- Isolate behind interfaces
- Easier to test, swap, upgrade

---

### Pattern 7.2: Learning Tests

**Problem**: Don't understand external library  
**Solution**: Write tests documenting behavior  

```php
// Learning test: How does Laravel's Auth work?
public function testLaravelGuardsAuthenticatedRoutes() {
    $user = User::factory()->create();
    $this->actingAs($user);
    
    $response = $this->get('/admin');
    $response->assertStatus(200); // Can access
}

public function testLaravelRedirectsUnauthenticated() {
    $response = $this->get('/admin');
    $response->assertRedirect('/login');
}

// Learning test: Carbon date manipulation
public function testCarbonDateArithmetic() {
    $date = Carbon::create(2024, 5, 15);
    $future = $date->addDays(30);
    
    $this->assertEquals(6, $future->month);
    $this->assertEquals(14, $future->day);
}
```

**AI Decision Rule**:
- Write learning tests before using unfamiliar library
- Tests document expected behavior + edge cases
- Catches compatibility issues early

---

## REFERENCE GUIDE FOR AI IMPLEMENTATION

### When to Apply

| Pattern | Priority | Apply For | Skip For |
|---------|----------|-----------|----------|
| Functions (3) | ALWAYS | All code | Never |
| Error Handling (8) | HIGH | New code, boundaries | Non-critical |
| Classes (9) | MEDIUM | OOP code | Procedural |
| **SOLID (10)** | **HIGH** | **Architecture review** | **Simple code** |
| Table-Driven (6) | MEDIUM | Logic in data | Simple conditionals |
| Code Smells (11) | MEDIUM | Code review | Already clean |
| Boundaries (12) | MEDIUM | External libs | No dependencies |

### AI Proposal Format

```
[SECTION X.Y] | PATTERN: [Name]
├─ Issue: [What's wrong with design]
├─ Suggestion: [How to fix]
├─ Priority: HIGH|MEDIUM|LOW
└─ Focus: Clean Design
```

---

## SUMMARY

**Focus**: Write clean, maintainable, and well-architected code  
**7 sections, 18 patterns**: Functions (3), Error Handling (2), Classes (3), **SOLID (5)**, Table-Driven (1), Code Smells (3), Boundaries (2)

**Sections**:
- S3: Functions Design
- S8: Error Handling
- S9: Classes Design
- **S10: SOLID Principles** (5 patterns: O, L, I, D, S)
- S6: Data Structures (Table-Driven)
- S11: Code Smells (3 red flags)
- S12: Boundaries (Integration)

**For AI**: Design review, architecture audit, SOLID compliance check
