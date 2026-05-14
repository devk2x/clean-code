# Code Reference Suite

Hướng dẫn hoàn chỉnh về viết code, thiết kế kiến trúc, và duy trì chất lượng với tự động hóa CI/CD.

---

## 📚 4 File Reference

### Tầng Hướng Dẫn: Từ Advisory đến Enforcement

```
                    ┌─────────────────────────────┐
                    │   coding-rules.md           │
                    │   (BẮT BUỘC - CI/CD)        │
                    │   Enforce tiêu chuẩn        │
                    └────────────┬────────────────┘
                                 │
                    ┌────────────▼────────────────┐
                    │   code-quality.md           │
                    │   (OPTIMIZATION)            │
                    │   Khi nào cần cải thiện     │
                    └────────────┬────────────────┘
                                 │
                    ┌────────────▼────────────────┐
                    │   clean-code.md             │
                    │   (BEST PRACTICE)           │
                    │   Cách thiết kế tốt         │
                    └────────────┬────────────────┘
                                 │
                    ┌────────────▼────────────────┐
                    │   readable-code.md          │
                    │   (HƯỚNG DẪN)               │
                    │   Cách viết rõ ràng         │
                    └─────────────────────────────┘
```

---

## 🎯 Nhanh Chóng: File Nào Cho Tình Huống Nào?

### Theo Vai Trò

```
┌─────────────────┬──────────────────┬─────────────────────────┐
│ VAI TRÒ         │ FILE CHÍNH        │ KHI NÀO SỬ DỤNG         │
├─────────────────┼──────────────────┼─────────────────────────┤
│ Lập trình viên  │ readable-code    │ Viết code mới           │
│ Code reviewer   │ clean-code       │ Review thiết kế         │
│ Kiến trúc sư    │ code-quality     │ Lập kế hoạch hiệu năng  │
│ CI/CD Pipeline  │ coding-rules     │ Tự động kiểm tra        │
└─────────────────┴──────────────────┴─────────────────────────┘
```

### Theo Tình Huống

```
┌────────────────────────┬────────────────────────────────────┐
│ TÌNH HUỐNG             │ FILE CẦN KIỂM TRA                  │
├────────────────────────┼────────────────────────────────────┤
│ "Mình đặt tên sao?"    │ readable-code.md (Pattern 1)       │
│ "Thiết kế đúng chưa?"  │ clean-code.md (Sections 3-4)       │
│ "Làm sao cho nhanh?"   │ code-quality.md (Section 2)        │
│ "CI/CD pass được?"     │ coding-rules.md (Section 1-7)      │
│ "Best practice là gì?" │ clean-code.md + code-quality.md    │
└────────────────────────┴────────────────────────────────────┘
```

---

## 📖 Chi Tiết Từng File

### 1️⃣ readable-code.md
**Mục đích**: Cách viết code **dễ đọc, dễ hiểu, rõ ràng**

```
┌─────────────────────────────────────────┐
│ READABLE CODE REFERENCE                 │
├─────────────────────────────────────────┤
│ 7 Sections | 15 Patterns | 609 lines   │
├─────────────────────────────────────────┤
│ ✓ Quy tắc đặt tên biến/hàm/class      │
│ ✓ Cách viết comment hiệu quả           │
│ ✓ Control flow và tính rõ ràng          │
│ ✓ Phân tích biểu thức phức tạp          │
│ ✓ Quản lý phạm vi biến                  │
│ ✓ Định dạng code & thẩm mỹ              │
│ ✓ Cấu trúc test tốt                    │
├─────────────────────────────────────────┤
│ Tone: HƯỚNG DẪN (được gợi ý)           │
│ Khi: Code review, audit tính rõ ràng   │
│ Enforcement: Không                     │
└─────────────────────────────────────────┘
```

**Ví dụ Patterns**:
- Pattern 1.1: Đặt tên biến → `$userAge` vs `$u`
- Pattern 2.1: Intent của comment → "Tại sao" chứ không "Cái gì"
- Pattern 4.1: Guard clauses → Reduce nesting với early returns

---

### 2️⃣ clean-code.md
**Mục đích**: Cách **thiết kế code sạch, dễ bảo trì, kiến trúc tốt**

```
┌─────────────────────────────────────────┐
│ CLEAN CODE REFERENCE                    │
├─────────────────────────────────────────┤
│ 7 Sections | 18 Patterns | 1020 lines  │
├─────────────────────────────────────────┤
│ ✓ Thiết kế hàm & trách nhiệm            │
│ ✓ Chiến lược xử lý lỗi                  │
│ ✓ Thiết kế class & OOP patterns         │
│ ✓ SOLID Principles (5 patterns)         │
│ ✓ Phương pháp table-driven              │
│ ✓ Code smells & refactoring             │
│ ✓ Boundaries & integration              │
├─────────────────────────────────────────┤
│ Tone: BEST PRACTICE (được khuyến cáo)  │
│ Khi: Design review, architecture      │
│ Enforcement: Không                     │
└─────────────────────────────────────────┘
```

**Ví dụ Patterns**:
- Pattern 3.1: Single Responsibility → 1 hàm = 1 lý do thay đổi
- Pattern 10.1: Open/Closed Principle → Extend, đừng sửa
- Pattern 11.1: Code Smells → Extract duplicate code

---

### 3️⃣ code-quality.md
**Mục đích**: Cách **duy trì chất lượng qua DRY, hiệu năng, thiết kế**

```
┌─────────────────────────────────────────┐
│ CODE QUALITY REFERENCE                  │
├─────────────────────────────────────────┤
│ 3 Sections | 8 Patterns | 395 lines    │
├─────────────────────────────────────────┤
│ ✓ Nguyên tắc DRY & loại bỏ trùng lặp   │
│ ✓ Tuning code & hiệu năng               │
│ ✓ Pseudocode & quy trình thiết kế       │
├─────────────────────────────────────────┤
│ Tone: OPTIMIZATION (khi cần thiết)      │
│ Khi: Lập kế hoạch hiệu năng, review    │
│ Enforcement: Không                     │
└─────────────────────────────────────────┘
```

**Ví dụ Patterns**:
- Pattern 1.1: DRY → Extract nếu logic lặp 3+ lần
- Pattern 2.1: Optimize → Đo lường trước, sau đó optimize hot paths
- Pattern 3.1: Pseudocode → Kế hoạch trước khi code

---

### 4️⃣ coding-rules.md
**Mục đích**: **Tiêu chuẩn bắt buộc được enforce bởi CI/CD**

```
┌─────────────────────────────────────────┐
│ CODING RULES & STANDARDS                │
├─────────────────────────────────────────┤
│ 7 Sections | 21 Rules | 678 lines      │
├─────────────────────────────────────────┤
│ ✓ Quy tắc đặt tên (auto-enforce)       │
│ ✓ Giới hạn cấu trúc code                │
│ ✓ Yêu cầu xử lý lỗi                    │
│ ✓ Mục tiêu test coverage tối thiểu     │
│ ✓ Yêu cầu documentation                 │
│ ✓ Enforcement chất lượng                │
│ ✓ Quy tắc bảo mật                      │
├─────────────────────────────────────────┤
│ Tone: BẮT BUỘC (auto-fail nếu vi phạm) │
│ Khi: Kiểm tra CI/CD pipeline            │
│ Enforcement: CÓ - block merge           │
└─────────────────────────────────────────┘
```

**Ví dụ Rules**:
- Rule 1.1: Biến PHẢI là `$camelCase` (auto-fail nếu không)
- Rule 4.1: Min 70% test coverage (auto-fail nếu thấp hơn)
- Rule 7.2: Input PHẢI được validate (auto-fail nếu không)

---

## 🔄 Cách Chúng Hoạt Động Cùng Nhau

### Ví Dụ: Thêm Feature Mới

```
Bước 1: THIẾT KẾ (Trước khi code)
└─→ Kiểm tra clean-code.md
    • Section 3: Cấu trúc class mới như thế nào?
    • Section 10: Áp dụng SOLID principles?
    • Section 6: Cần table-driven data không?

Bước 2: CODE (Viết feature)
└─→ Kiểm tra readable-code.md
    • Section 1: Đặt tên biến/hàm như thế nào?
    • Section 4: Control flow rõ ràng?
    • Section 7: Cách viết test?

Bước 3: OPTIMIZE (Nếu cần hiệu năng)
└─→ Kiểm tra code-quality.md
    • Section 2: Profile trước, sau đó optimize?
    • Section 1: Có trùng lặp để xóa không (DRY)?

Bước 4: SUBMIT (Trước khi tạo PR)
└─→ Kiểm tra coding-rules.md
    • Rule 1: Đặt tên pass validation?
    • Rule 4: Tests đạt coverage tối thiểu?
    • Rule 5: Có docblocks?
    • Rule 7: Không hardcode secret?
    
    ✅ Nếu tất cả rules pass → PR merge tự động
    ❌ Nếu rule fail → CI/CD block merge
```

---

## 📊 Phân Tích Nội Dung

### Ma Trận Coverage

```
CHỦ ĐỀ                   │ readable │ clean │ quality │ rules
─────────────────────────────────────────────────────────────
Quy tắc đặt tên         │    ✓     │   ✓   │    ✓    │   ✓
Comments & Docs         │    ✓     │   ✓   │    ✓    │   ✓
Cấu trúc code           │    ✓     │   -   │    -    │   ✓
Functions & Methods     │    -     │   ✓   │    -    │   ✓
Classes & Design        │    -     │   ✓   │    -    │   -
Error Handling          │    ✓     │   ✓   │    ✓    │   ✓
Testing                 │    ✓     │   ✓   │    -    │   ✓
Code Quality/DRY        │    ✓     │   ✓   │    ✓    │   ✓
Performance             │    -     │   -   │    ✓    │   -
Security                │    -     │   -   │    -    │   ✓
```

---

## 🎓 Con Đường Học Tập

### Cho Lập Trình Viên Mới

```
Tuần 1: readable-code.md
├─ Section 1: Đặt tên (cách đặt tên)
├─ Section 4: Control Flow (viết logic rõ ràng)
└─ Section 7: Testing (viết test tốt)

Tuần 2: clean-code.md Sections 3-4
├─ Pattern 3.1: Single Responsibility
├─ Pattern 9.2: Encapsulation
└─ Pattern 10.1-10.5: SOLID Principles

Tuần 3: code-quality.md
├─ Section 1: DRY (tránh trùng lặp)
└─ Section 2: Performance (khi nào optimize)

Tuần 4: coding-rules.md
└─ Tất cả sections (hiểu CI/CD checks)
```

### Cho Code Reviewer

```
Focus vào: clean-code.md + readable-code.md

Trước khi approve PR, kiểm tra:
✓ Thiết kế (clean-code Sections 3-4)
✓ Code style (readable-code Section 1-4)
✓ Error handling (clean-code Section 8)
✓ Testing (readable-code Section 7)
```

### Cho Kiến Trúc Sư

```
Focus vào: clean-code.md + code-quality.md

Thảo luận thiết kế:
✓ SOLID principles (clean-code Section 10)
✓ Architecture patterns (clean-code Sections 3-12)
✓ Performance strategy (code-quality Section 2)
✓ DRY principle (code-quality Section 1)
```

---

## 🚀 Tích Hợp CI/CD

### Những Gì Được Enforce

```
coding-rules.md thực thi:

1. Naming Check
   └─ Tất cả biến/hàm/class match convention
      ✅ $camelCase, camelCase(), PascalCase, UPPER_SNAKE_CASE
      ❌ Fail nếu $snake_case, methodName_old(), v.v.

2. Code Structure Check
   └─ File < 300 lines, Method < 30 lines, Nesting ≤ 3
      ✅ Auto-pass nếu trong giới hạn
      ❌ Fail nếu vượt quá

3. Error Handling Check
   └─ Không generic exceptions, tất cả có context
      ✅ throw new InvalidEmailException("reason")
      ❌ Fail: throw new Exception() hoặc generic

4. Test Coverage Check
   └─ Min 70% code coverage, method mới phải có test
      ✅ Coverage ≥ 70%
      ❌ Fail nếu < 60%

5. Documentation Check
   └─ Tất cả class/public method có docblock
      ✅ /** @param @return @throws */
      ❌ Fail nếu thiếu

6. Quality Gate Check
   └─ Không duplication > 5%, magic number, commented code
      ✅ DRY enforce, dùng constant
      ❌ Fail nếu vi phạm

7. Security Check
   └─ Không hardcode secret, input validate, no SQL injection
      ✅ Dùng env() hoặc config()
      ❌ Fail: hardcode API key
```

### Trước & Sau

```
TRƯỚC CI/CD (Manual Review):
Lập trình viên submit PR
  ↓
Reviewer đọc code (30 phút)
  ↓
Feedback: "Dùng camelCase", "Add test", "Extract DRY"
  ↓
Lập trình viên fix (1 giờ)
  ↓
Re-review (15 phút)
  ↓
Approve & merge
Tổng: ~2 giờ, human bottleneck

SAU CI/CD (Automated):
Lập trình viên submit PR
  ↓
CI/CD chạy 7 kiểm tra (30 giây)
  ├─ ❌ Naming check FAILS
  ├─ ❌ Test coverage FAILS
  └─ ❌ Docblock FAILS
  ↓
PR block tự động
  ↓
Lập trình viên chạy auto-fix tools (2 phút)
  ↓
Resubmit PR
  ↓
CI/CD chạy kiểm tra (30 giây)
  ├─ ✅ Tất cả pass
  ↓
Human reviewer approve design (10 phút)
  ↓
Auto-merge
Tổng: ~15 phút, scale đến 100s PR
```

---

## 📋 Quick Lookup

### "Mình có thể..."

```
...đặt tên biến sao?
→ readable-code.md Pattern 1.1

...viết test sao?
→ readable-code.md Section 7

...thiết kế class sao?
→ clean-code.md Section 9

...loại bỏ trùng lặp sao?
→ code-quality.md Section 1

...xử lý lỗi đúng cách?
→ clean-code.md Section 8 + coding-rules.md Rule 3

...optimize hiệu năng sao?
→ code-quality.md Section 2

...đảm bảo pass CI/CD?
→ coding-rules.md Sections 1-7

...viết comment tốt?
→ readable-code.md Section 2

...áp dụng SOLID?
→ clean-code.md Section 10

...viết code bảo trì tốt?
→ clean-code.md Sections 3-9
```

---

## 📈 Thống Kê

### Tất Cả Files Combined

```
Tổng Nội Dung:
├─ 4 file reference
├─ 24 sections
├─ 62 patterns/rules
├─ 2,699 dòng nội dung
└─ 100% cover code practices

Phân Tích Theo File:
├─ readable-code.md:   609 lines | 7 sections | 15 patterns
├─ clean-code.md:     1020 lines | 7 sections | 18 patterns
├─ code-quality.md:    395 lines | 3 sections |  8 patterns
└─ coding-rules.md:    678 lines | 7 sections | 21 rules
```

---

## 🎯 Bước Tiếp Theo

### Cách Sử Dụng Files

1. **Bookmark lại**: Thêm vào IDE/docs của bạn
2. **Configure CI/CD**: Tích hợp coding-rules.md checks
3. **Chia sẻ team**: Dùng README.md này làm onboarding guide
4. **Reference trong PRs**: Link cụ thể patterns khi code review

### Mở Rộng

- Thêm team-specific rules vào coding-rules.md
- Thêm project-specific patterns vào readable-code.md
- Document thêm optimization strategies trong code-quality.md

---

## 📞 Hỗ Trợ

### Tổ Chức Files

```
/references/
├─ README.md                  ← Bạn đang ở đây
├─ readable-code.md          ← Cách viết
├─ clean-code.md             ← Cách thiết kế
├─ code-quality.md           ← Cách optimize
└─ coding-rules.md           ← Phải enforce
```

### Cách Dùng

- **Reference riêng lẻ**: Đọc file cụ thể cho topic
- **PR validation**: CI/CD chạy coding-rules.md checks
- **Code review**: Reference patterns từ clean-code.md
- **Learning**: Follow con đường học tập từ "Con Đường Học Tập" section

---

**Tất cả files đã sẵn sàng production và tối ưu hóa cho Forge agents.** ✅

