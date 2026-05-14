# Code Quality Reference

> **Mục tiêu**: Tập trung vào cách duy trì chất lượng code: **loại bỏ duplication, optimize wisely, design process**
> - **Khi nào dùng**: Code review cho duplication, performance profiling, planning complex functions
> - **Không bao gồm**: Code formatting, readability, design architecture
> - **Phù hợp cho**: DRY audit, performance optimization, design planning, pseudocode review

---

## SECTION 13: WRITING LESS CODE & DRY PRINCIPLE

From: "The Art of Readable Code" + "Code Complete"

### Pattern 13.1: DRY — Don't Repeat Yourself

**Problem**: Same logic appears multiple times  
**Solution**: Extract to reusable function  

```php
// ANTI-PATTERN: Duplicate validation
class UserService {
    public function createUser($email, $password) {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidEmailException();
        }
        if (strlen($password) < 8) {
            throw new WeakPasswordException();
        }
        // Create user
    }
    
    public function updateUserEmail($userId, $newEmail) {
        if (!filter_var($newEmail, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidEmailException();
        }
        // Update email
    }
    
    public function sendResetEmail($email) {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidEmailException();
        }
        // Send reset
    }
}

// BEST-PRACTICE: Extract validator
class EmailValidator {
    public function validate(string $email): void {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidEmailException("Invalid email: $email");
        }
    }
}

class UserService {
    private EmailValidator $validator;
    
    public function createUser($email, $password) {
        $this->validator->validate($email);
        $this->validatePassword($password);
        // Create user
    }
    
    public function updateUserEmail($userId, $newEmail) {
        $this->validator->validate($newEmail);
        // Update email
    }
    
    public function sendResetEmail($email) {
        $this->validator->validate($email);
        // Send reset
    }
}
```

**AI Decision Rule**:
- Duplicate code appears 3+ times → extract
- Extract when: same logic OR same pattern
- Group validators, formatters, calculators

---

### Pattern 13.2: Remove Dead Code

**Problem**: Unused functions, commented code clutter  
**Solution**: Delete (version control has history)  

```php
// ANTI-PATTERN: Dead code
public function oldCalculatePrice($items) {
    // Replaced by new_calculate_price_v2
    // return array_sum($items);
}

public function processOrder($order) {
    $result = $this->chargePayment($order);
    return $result;
    // Unreachable code
    $this->sendNotification($order);
    $this->logOrder($order);
}

private function deprecatedUserValidation($user) {
    // No longer used
}

// BEST-PRACTICE: Delete immediately
public function chargePayment(Order $order): PaymentResult {
    return $this->payment_gateway->charge($order->total);
}
```

**AI Decision Rule**:
- Delete commented code (Git has history)
- Remove unreachable code
- Delete unused functions
- Don't create "backwards compatibility" branches

---

### Pattern 13.3: Simplify Complex Conditions

**Problem**: Complex expressions instead of simpler approach  
**Solution**: Use library functions, simplify logic  

```php
// ANTI-PATTERN: Manual complexity
$result = [];
for ($i = 0; $i < count($users); $i++) {
    if ($users[$i]['age'] > 18 && $users[$i]['status'] === 'active') {
        $result[] = $users[$i]['name'];
    }
}

// BEST-PRACTICE: Use built-in functions
$adultActiveNames = collect($users)
    ->filter(fn($u) => $u['age'] > 18 && $u['status'] === 'active')
    ->pluck('name')
    ->toArray();

// Or more explicit
$adultUsers = array_filter($users, function($user) {
    return $user['age'] > 18 && $user['status'] === 'active';
});
$names = array_map(fn($u) => $u['name'], $adultUsers);
```

**AI Decision Rule**:
- Use language/library features (avoid manual loops)
- Simpler code = easier to understand
- Less code = less bugs

---

## SECTION 14: CODE TUNING & PERFORMANCE

From: "Code Complete" + practical considerations

### Pattern 14.1: When to Optimize — Measure First

**Problem**: Optimize without knowing where bottleneck is  
**Solution**: Profile before optimizing  

```php
// DON'T DO THIS: Premature optimization
public function getUserCount(): int {
    // Over-complicated for 0.1% performance gain
    $cacheKey = 'user_count_' . date('Y-m-d-H-i');
    if (cache()->has($cacheKey)) {
        return cache()->get($cacheKey);
    }
    $count = DB::table('users')->count();
    cache()->put($cacheKey, $count, 60);
    return $count;
}

// DO THIS: Simple, then measure if needed
public function getUserCount(): int {
    return DB::table('users')->count();
}

// If performance issue, THEN optimize:
public function getUserCount(): int {
    return cache()->remember('user_count', 3600, 
        fn() => DB::table('users')->count()
    );
}
```

**AI Decision Rule**:
- Write correct code first
- Optimize only if profiling shows bottleneck
- Optimize hot paths only (10% of code uses 90% of time)

---

### Pattern 14.2: Common Performance Patterns

**Problem**: N+1 queries, inefficient algorithms  
**Solution**: Use proven patterns  

```php
// ANTI-PATTERN: N+1 query problem
foreach ($users as $user) {
    echo $user->profile->bio;  // Query per user!
}

// BEST-PRACTICE: Eager load
$users = User::with('profile')->get();
foreach ($users as $user) {
    echo $user->profile->bio;  // Single query via join
}

// ANTI-PATTERN: Inefficient search (O(n))
function findUserByEmail($email) {
    foreach ($users as $user) {
        if ($user['email'] === $email) return $user;
    }
}

// BEST-PRACTICE: Indexed lookup (O(1))
$usersByEmail = arrayColumn($users, null, 'email');
return $usersByEmail[$email] ?? null;

// Or database query
return User::where('email', $email)->first();
```

**AI Decision Rule**:
- N+1 queries: use eager loading (with)
- Large collections: use indexed lookups
- Database: use indexes on frequently searched columns
- Trade-off: memory vs speed intelligently

---

### Pattern 14.3: Performance Trade-offs

**Problem**: Optimizing for speed while sacrificing readability  
**Solution**: Comment trade-off decisions  

```php
// BEST-PRACTICE: Document performance trade-offs
// Trade-off: Cache user for 1 hour (stale by 1 hour max)
// Benefit: Reduce DB load by 90% (measured in production)
// Risk: User permissions change, not reflected until cache expires
// Mitigation: Clear cache when user role changes
$user = cache()->remember(
    "user:{$userId}",
    3600,
    fn() => User::find($userId)
);

// Trade-off: Store denormalized user_email in orders table
// Benefit: Faster queries (no join needed)
// Risk: Email changes not reflected in historical orders
// Mitigation: Acceptable (historical email relevant, not current)
public function getOrderWithCustomerEmail($orderId) {
    return DB::table('orders')
        ->where('id', $orderId)
        ->get(['customer_name', 'customer_email']); // Denormalized
}
```

**AI Decision Rule**:
- Only optimize if profiling shows bottleneck
- Document trade-off: what sacrificed, why, risk
- Readable code > clever optimization
- Premature optimization = enemy of readable code

---

## SECTION 15: PSEUDOCODE & PROGRAMMING PROCESS

From: "Code Complete"

### Pattern 15.1: Write Pseudocode Before Code

**Problem**: Dive directly into coding, miss design  
**Solution**: Plan with pseudocode first  

```php
// PSEUDOCODE (Plan before implementing)
/*
function processOrder($order) {
    1. Validate order data
    2. Check inventory for items
    3. If items unavailable, throw error
    4. Calculate total price
    5. Apply discounts
    6. Process payment
    7. If payment fails, throw error
    8. Update inventory (decrement quantities)
    9. Create shipment record
    10. Notify customer
    11. Return confirmation
}
*/

// IMPLEMENT from pseudocode
public function processOrder(Order $order): OrderConfirmation {
    $this->validateOrder($order);
    $this->checkInventoryAvailable($order);
    
    $total = $this->calculateTotalPrice($order);
    $total = $this->applyDiscounts($total, $order);
    
    $payment = $this->processPayment($order->payment_method, $total);
    if (!$payment->isSuccessful()) {
        throw new PaymentFailedException($payment->errorMessage());
    }
    
    $this->updateInventory($order);
    $this->createShipmentRecord($order);
    $this->notifyCustomerOfOrder($order);
    
    return new OrderConfirmation($order, $payment);
}
```

**AI Decision Rule**:
- Complex functions (>30 lines): pseudocode first
- Maps 1:1 to implementation functions
- Easier to review logic before coding

---

### Pattern 15.2: Top-Down Design vs Bottom-Up

**Problem**: Unclear structure, building wrong thing  
**Solution**: Design top-down, implement bottom-up  

```php
// TOP-DOWN DESIGN: Start with public interface
interface PaymentProcessor {
    public function process(Order $order): PaymentResult;
}

// Then design main flow
public function processOrder(Order $order): OrderConfirmation {
    $paymentResult = $paymentProcessor->process($order);
    $this->handlePaymentResult($paymentResult);
    return new OrderConfirmation($order);
}

// Then implement pieces bottom-up
public function validateOrder(Order $order): void { }
public function calculateTotal(Order $order): float { }
public function applyDiscounts(float $total, Order $order): float { }
```

**AI Decision Rule**:
- Design: top-down (high-level first)
- Implement: bottom-up (utilities first, then composition)
- Test: from bottom up (utilities first)

---

## REFERENCE GUIDE FOR AI IMPLEMENTATION

### When to Apply

| Pattern | Priority | Apply For | Skip For |
|---------|----------|-----------|----------|
| DRY | HIGH | Duplicate code (3+) | First occurrence |
| Simplify | HIGH | Complex conditions | Simple logic |
| Performance | LOW | Hot paths only | Before profiling |
| Pseudocode | MEDIUM | Complex functions | Simple functions |

### AI Proposal Format

```
[SECTION X.Y] | PATTERN: [Name]
├─ Issue: [What's inefficient]
├─ Suggestion: [How to improve]
├─ Priority: HIGH|MEDIUM|LOW
└─ Focus: Code Quality
```

### Decision Logic for Review

1. **DRY**: Flag if duplicate code appears 3+ places
2. **Dead Code**: Suggest removal of commented/unreachable code
3. **Simplify**: Suggest library functions for manual loops
4. **Performance**: Only suggest if profiling shows bottleneck
5. **Pseudocode**: Suggest for complex functions (>30 lines)

---

## SUMMARY

**Focus**: Chất lượng code: DRY, Performance, Design Process  
**4 sections, 9 patterns**: DRY, Dead Code, Simplify, Performance, Pseudocode  
**For AI**: DRY audit, performance review, design planning
