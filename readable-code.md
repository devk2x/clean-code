# Readable Code Reference

> **Objective**: Focus on writing code that is **easy to read, understand, and clear**
> - **When to use**: Review new code, refactor for clarity, audit naming/formatting
> - **Not included**: Design architecture, OOP patterns, performance tuning
> - **Suitable for**: Code review, naming conventions, control flow clarity, test structure

---

## SECTION 1: NAMING — CLEAR & INTENTIONAL

Consolidated from: "Readable Code" + "Clean Code" + "Code Complete"

### Pattern 1.1: Variables — Describe Intent & Scope

**Problem**: Short/generic names hide meaning  
**Solution**: Name scope size — long names for broad scope, short for narrow  

```php
// ANTI-PATTERN
$u = User::find($id);
$x = $u->age;
if ($x > 18) { }

// BEST-PRACTICE
$user = User::find($id);
$userAge = $user->age;
$adultAgeThreshold = 18;
if ($userAge > $adultAgeThreshold) { }

// In loops: short names OK for small scope
foreach ($users as $u) {
    echo $u->name; // OK - scope: 1 line
}

// In methods: long names for clarity
private function calculateMonthlyRevenueForActiveMerchants(): float {
    // Broader scope, explicit name
}
```

**AI Decision Rule**:
- Variable name length ~ scope size
- Include type/unit: `$timeoutMs`, `$isActive`, `$userCount`
- Avoid: `$temp`, `$data`, `$val`, `$result` (meaningless)

---

### Pattern 1.2: Functions — Verb + Intent

**Problem**: Names don't describe action or return  
**Solution**: Prefix verb + clear intent  

```php
// ANTI-PATTERN
function process($user) { }
function handle($data) { }

// BEST-PRACTICE
function getActiveUsersBySubscriptionTier(string $tier): array { }
function isEmailValidFormat(string $email): bool { }
function calculateMonthlySubscriptionCost(User $user): float { }

// Naming patterns:
// Getters: get* (returns value)
// Setters: set* (modifies state)
// Checks: is*, has*, can* (returns bool)
// Actions: verb+noun (sendEmail, processOrder)
```

**AI Decision Rule**:
- Boolean functions: `is_*`, `has_*`, `can_*`, `should_*`
- Action functions: start with verb
- Avoid: vague verbs (process, handle, do, work, manage)

---

### Pattern 1.3: Constants & Magic Values

**Problem**: Hardcoded numbers scattered without context  
**Solution**: Named constants with semantic meaning  

```php
// ANTI-PATTERN
if ($user->age > 18) { }
if ($attempts > 3) { }
if ($status !== 'active') { }
$discount = $price * 0.2;

// BEST-PRACTICE
const ADULT_AGE_THRESHOLD = 18;
const MAX_LOGIN_ATTEMPTS = 3;
const DEFAULT_DISCOUNT_RATE = 0.2;

enum UserStatus: string {
    case ACTIVE = 'active';
    case INACTIVE = 'inactive';
}

if ($user->age > self::ADULT_AGE_THRESHOLD) { }
if ($attempts > self::MAX_LOGIN_ATTEMPTS) { }
if ($status !== UserStatus::ACTIVE->value) { }
$discount = $price * self::DEFAULT_DISCOUNT_RATE;
```

**AI Decision Rule**:
- Extract magic numbers if: used 2+ times OR intent unclear
- Use enums for status/type strings
- Constants = UPPER_SNAKE_CASE

---

## SECTION 2: COMMENTS — WHY NOT WHAT

Consolidated from: "Readable Code" + "Clean Code"

### Pattern 2.1: Document Intent, Not Implementation

**Problem**: Comments repeat obvious code  
**Solution**: Explain WHY, constraints, decisions  

```php
// ANTI-PATTERN
$user = User::find($id);        // Get user by ID
$isAdmin = $user->hasRole();   // Check admin role
if ($isAdmin) { /* ... */ }    // If admin, process

// BEST-PRACTICE
// Cache user for 1 hour to respect API quota (100 req/day limit)
$user = cache()->remember("user:$id", 3600, fn() => User::find($id));

// Only allow admin access on paid tier
// (free tier can access public features, admins get beta features)
$hasAdminRole = $user->hasRole('admin');
$isPaidSubscriber = $user->subscription->status === 'paid';
if ($hasAdminRole && $isPaidSubscriber) {
    $this->grantAdminAccess($user);
}
```

**AI Decision Rule**:
- Comment WHY: business rule, constraint, decision
- Skip WHAT: code already shows what it does
- Skip obvious: "increment", "check if", "loop through"

---

### Pattern 2.2: When NOT to Comment — Code Should Self-Document

**Problem**: Stale comments become lies  
**Solution**: Write clear code instead  

```php
// ANTI-PATTERN
// This method processes user data
public function processUserData($data) {
    // Validate email
    $email = $data['email'] ?? null;
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        throw new Exception('Invalid');
    }
    // More validation...
}

// BEST-PRACTICE: Clear names eliminate need for comments
public function validateAndExtractUserEmail(array $data): string {
    $email = $data['email'] ?? null;
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        throw new InvalidEmailException("Email '$email' invalid format");
    }
    return $email;
}
```

**AI Decision Rule**:
- Good code > good comments
- If tempted to comment WHAT, rename variable/function instead
- Comments = only for non-obvious WHY

---

## SECTION 3: CONTROL FLOW — CLEAR & MINIMAL NESTING

Consolidated from: "Readable Code" + "Clean Code"

### Pattern 3.1: Guard Clauses — Early Return

**Problem**: Deep nesting = cognitive overload  
**Solution**: Invert conditions, return early  

```php
// ANTI-PATTERN: 5 levels of nesting
public function processPayment(Order $order) {
    if ($order->status === 'active') {
        if ($order->total > 0) {
            if ($this->user->hasPaymentMethod()) {
                if ($this->validatePayment($order)) {
                    $payment = $this->charge($order->total);
                    return $payment;
                }
            }
        }
    }
    return null;
}

// BEST-PRACTICE: Guard clauses, linear flow
public function processPayment(Order $order): ?Payment {
    if ($order->status !== 'active') {
        return null;
    }
    if ($order->total <= 0) {
        return null;
    }
    if (!$this->user->hasPaymentMethod()) {
        return null;
    }
    if (!$this->validatePayment($order)) {
        return null;
    }
    return $this->charge($order->total);
}
```

**AI Decision Rule**:
- Nesting >2 levels = extract guard clause
- Test invalid conditions first, return early
- Happy path at end of function

---

### Pattern 3.2: Standard Loop Patterns

**Problem**: Manual loop patterns with counters, unclear intent  
**Solution**: Use `foreach`, collection methods, standard patterns  

From "Code Complete" — use standard patterns

```php
// STANDARD: Iterate through collection
foreach ($users as $user) {
    echo $user->name;
}

// STANDARD: Iterate with index
foreach ($items as $index => $item) {
    $total += $item->price * ($index + 1);
}

// STANDARD: Search for item
$foundUser = null;
foreach ($users as $user) {
    if ($user->id === $searchId) {
        $foundUser = $user;
        break;
    }
}
// Better: use array functions
$foundUser = collect($users)->firstWhere('id', $searchId);

// ANTI-PATTERN: Using loop counters with arrays
for ($i = 0; $i < count($array); $i++) {
    echo $array[$i];
}
// Better: foreach
foreach ($array as $element) {
    echo $element;
}
```

**AI Decision Rule**:
- Use `foreach` for iteration (not `for` with counters)
- Use Laravel collection methods: `map`, `filter`, `first`, `find`
- Break early if found (don't continue unnecessarily)

---

## SECTION 4: EXPRESSIONS — BREAK DOWN COMPLEX LOGIC

Consolidated from: "Readable Code"

### Pattern 4.1: Extract Boolean Logic

**Problem**: Giant boolean expression hard to verify  
**Solution**: Each condition = named variable  

```php
// ANTI-PATTERN
if ($user->isActive && $user->hasRole('admin') && 
    ($user->lastLogin > strtotime('-30 days') || $user->isServiceAccount) &&
    $user->subscription->status === 'paid' && $user->email_verified) {
    // Grant access
}

// BEST-PRACTICE
$isAccountActive = $user->isActive;
$hasAdminRole = $user->hasRole('admin');
$recentlyActive = $user->lastLogin > strtotime('-30 days');
$isServiceAccount = $user->isServiceAccount;
$hasActivity = $recentlyActive || $isServiceAccount;
$isPaidSubscriber = $user->subscription->status === 'paid';
$emailVerified = $user->email_verified;

$canGrantAccess = $isAccountActive && $hasAdminRole && 
                    $hasActivity && $isPaidSubscriber && $emailVerified;

if ($canGrantAccess) {
    $this->grantAdminAccess($user);
}
```

**AI Decision Rule**:
- Extract if: >2 operators OR intent unclear
- Each variable = 1 semantic concept
- Final variable summarizes business logic

---

### Pattern 4.2: Break Down Financial Calculations

**Problem**: Dense calculations hard to verify (critical for money!)  
**Solution**: Each step explicit  

```php
// ANTI-PATTERN: Dense formula
$finalPrice = ($basePrice * (1 - $discount)) + 
               ($shipping * $weightFactor) - 
               ($loyalty ?? 0) + 
               ($taxRate * $basePrice);

// BEST-PRACTICE: Step-by-step
$discountedPrice = $basePrice * (1 - $discount);
$shippingFee = $shipping * $weightFactor;
$subtotal = $discountedPrice + $shippingFee;
$subtotalAfterLoyalty = $subtotal - ($loyalty ?? 0);
$taxAmount = $taxRate * $basePrice;
$finalPrice = $subtotalAfterLoyalty + $taxAmount;
```

**AI Decision Rule**:
- Each operation = separate variable
- Verify intermediate results
- Critical for financial/payment code

---

## SECTION 5: DATA STRUCTURES & VARIABLES — SCOPE & COHESION

Consolidated from: "Readable Code" + "Code Complete"

### Pattern 5.1: Keep Scope Small

**Problem**: Variables declared early, used late  
**Solution**: Declare at point of use  

```php
// ANTI-PATTERN
public function getUserStats($userId) {
    $user = User::find($userId);
    $startDate = Carbon::now()->subMonth();
    $orders = Order::all();
    
    // 20 lines later...
    $userOrders = $orders->where('user_id', $userId)->where('created_at', '>=', $startDate);
    return $userOrders->count();
}

// BEST-PRACTICE
public function getUserStats($userId): int {
    $oneMonthAgo = Carbon::now()->subMonth();
    $userOrders = Order::where('user_id', $userId)
        ->where('created_at', '>=', $oneMonthAgo)
        ->get();
    return $userOrders->count();
}
```

**AI Decision Rule**:
- Declare variable as late as possible
- Scope = as narrow as possible
- Reduces context to maintain mentally

---

## SECTION 6: FORMATTING & AESTHETICS

Consolidated from: "Readable Code" + "Clean Code"

### Pattern 6.1: Consistent Indentation & Spacing

**Problem**: Inconsistent formatting, hard to read  
**Solution**: Consistent style throughout codebase  

```php
// ANTI-PATTERN: Inconsistent
class User {
public function getName(){
return $this->name;
}
    public function setName($name) {
        $this->name = $name;
    }
}

// BEST-PRACTICE: Consistent indentation (4 spaces)
class User {
    public function getName(): string {
        return $this->name;
    }
    
    public function setName(string $name): void {
        $this->name = $name;
    }
}
```

**AI Decision Rule**:
- Consistent indentation: 4 spaces or 2 spaces (not tabs)
- Blank lines between methods (group related logic)
- Consistent spacing around operators: `$a + $b` not `$a+$b`

---

### Pattern 6.2: Line Length & Method Formatting

**Problem**: Long lines hard to scan  
**Solution**: Keep lines ≤100 characters  

```php
// ANTI-PATTERN: Line > 100 chars
public function calculateTotalWithTaxAndShippingAndDiscount($basePrice, $taxRate, $shippingCost, $discountPercentage, $customerType, $isInternationalShipping) {

// BEST-PRACTICE: Break long lines
public function calculateTotalWithTaxAndShipping(
    float $basePrice,
    float $taxRate,
    float $shippingCost,
    float $discountPercentage
): float {
    $discounted = $basePrice * (1 - $discountPercentage);
    $tax = $discounted * $taxRate;
    return $discounted + $tax + $shippingCost;
}

// Parameters aligned
$result = $this->process($parameterOne,
    $parameterTwo,
    $parameterThree,
    $parameterFour
);
```

**AI Decision Rule**:
- Max line length: 100 characters
- Break long parameter lists
- Align related code blocks
- Empty lines to separate concerns

---

### Pattern 6.3: Meaningful Whitespace

**Problem**: Code blocks unclear  
**Solution**: Use spacing to show structure  

```php
// ANTI-PATTERN: No grouping
public function processOrder($order) {
    $this->validate($order);
    $total = $this->calculateTotal($order);
    $payment = $this->processPayment($total);
    $this->updateInventory($order);
    $this->notifyCustomer($order);
    return new OrderResult($order, $payment);
}

// BEST-PRACTICE: Group related logic with blank lines
public function processOrder(Order $order): OrderResult {
    // Validation
    $this->validate($order);
    $this->checkInventoryAvailable($order);
    
    // Calculate totals
    $subtotal = $this->calculateSubtotal($order);
    $tax = $this->calculateTax($subtotal);
    $total = $subtotal + $tax + $this->calculateShipping($order);
    
    // Process payment
    $payment = $this->processPayment($total, $order->payment_method);
    if (!$payment->isSuccessful()) {
        throw new PaymentFailedException($payment->errorMessage());
    }
    
    // Update system
    $this->updateInventory($order);
    $this->notifyCustomerOfOrder($order);
    
    return new OrderResult($order, $payment);
}
```

**AI Decision Rule**:
- Blank lines separate logical phases (10-15 lines per group)
- Related statements grouped together
- Visual hierarchy shows code structure

---

## SECTION 7: TESTING — STRUCTURE & CLARITY

Consolidated from: "Readable Code"

### Pattern 7.1: AAA Structure — Arrange, Act, Assert

**Problem**: Setup/execution/verification mixed  
**Solution**: 3 clear sections  

```php
// ANTI-PATTERN: Mixed sections
public function testDiscount() {
    $product = new Product(['price' => 100]);
    $this->assertEquals(80, $product->applyDiscount(0.2));
    $product->price = 100;
    $product->save();
    $this->assertTrue($product->id > 0);
}

// BEST-PRACTICE: Arrange → Act → Assert
public function testDiscountCalculationAppliesRateCorrectly() {
    // ARRANGE: Setup
    $product = new Product(['price' => 100]);
    $discountRate = 0.2;
    $expectedPrice = 80;
    
    // ACT: Execute
    $finalPrice = $product->applyDiscount($discountRate);
    
    // ASSERT: Verify
    $this->assertEquals($expectedPrice, $finalPrice);
}
```

**AI Decision Rule**:
- 3 sections with blank lines
- One logical assertion per test
- Name describes scenario, not just what's tested

---

### Pattern 7.2: Test Names — Describe Scenario

**Problem**: Vague test names don't describe purpose  
**Solution**: Pattern: test_[subject]_[condition]_[expected]  

```php
// ANTI-PATTERN
public function testUser() { }
public function test1() { }
public function testEmail() { }

// BEST-PRACTICE
public function testUserCannotAccessAdminWithoutRole() { }
public function testPaymentFailsWhenCardDeclined() { }
public function testEmailValidationAcceptsStandardFormat() { }
public function testEmailValidationRejectsMissingDomain() { }

// From name alone, understand scenario completely
```

**AI Decision Rule**:
- Format: `test_[subject]_[condition]_[expected_result]`
- Each test = 1 scenario
- Name should explain to someone unfamiliar

---

## REFERENCE GUIDE FOR AI IMPLEMENTATION

### When to Apply

| Pattern | Priority | Apply For | Skip For |
|---------|----------|-----------|----------|
| Naming | ALWAYS | All code | Never |
| Comments | HIGH | Complex logic, WHY | Obvious code |
| Control Flow | ALWAYS | All code | Never |
| Expressions | HIGH | Complex boolean, math | Simple expressions |
| Data Scope | ALWAYS | All code | Never |
| Formatting | MEDIUM | New code | Legacy code |
| Testing | HIGH | Test code | Legacy untested |

### AI Proposal Format

```
[SECTION X.Y] | PATTERN: [Name]
├─ Issue: [What's unclear]
├─ Suggestion: [How to fix]
├─ Priority: HIGH|MEDIUM|LOW
└─ Focus: Readability
```

---

## SUMMARY

**Focus**: Write code that is easy to read, understand, and clear  
**7 sections, 15 patterns**: Naming, Comments, Control Flow, Expressions, Data Scope, Formatting, Testing  
**For AI**: Code review, clarity audit, naming consistency check
