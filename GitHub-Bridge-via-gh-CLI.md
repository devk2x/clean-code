# GitHub Bridge qua `gh` CLI — Không cần CI/CD

**Mục tiêu:** Tự động đưa workflow output lên GitHub bằng `gh` CLI làm tool trong Claude Code  
**Không cần:** GitHub Actions, CI/CD server, webhook, WSL, jq  
**Cần:** `gh` CLI đã auth, Node.js 18+, Claude Code Max $100  
**Môi trường:** Windows-first (chạy y hệt trên Mac/Linux/CI sau này)

---

## I. Ý tưởng cốt lõi (Bạn nghĩ đúng)

```
Claude Code (local, Windows)
    │
    │ chạy: gh (lệnh đơn) hoặc node github-review-loop.mjs (loop)
    ▼
gh CLI  ←──────────►  GitHub (Issues, PRs, Comments)
    │
    │ trả kết quả JSON về
    ▼
Claude Code parse → quyết định fix/không fix → reply
```

`gh` CLI là "tool" trong AI workflow. Claude Code gọi `gh` trực tiếp cho thao tác đơn (create, comment), và gọi script Node `github-review-loop.mjs` cho vòng lặp polling (fetch → so sánh state → output structured). Không cần MCP phức tạp, không cần API key riêng (gh tự quản auth).

**Vì sao Node thay vì bash:** team dev trên Windows — `jq`, `comm`, `grep` không có sẵn; script `.mjs` chạy thuần Node từ PowerShell/cmd, và tái dùng nguyên vẹn khi có CI Linux sau này (nhất quán với nguyên tắc dual-layer của pipeline).

Best practice xác nhận: "GitHub operations via gh CLI: issues, PRs, CI runs, code review, API queries. Use when creating/commenting on issues, listing/filtering PRs or issues."

---

## II. `gh` CLI làm được gì (Verified 2026)

### Issues

```bash
# Tạo issue từ file markdown
gh issue create --title "Design: Cache Layer" --body-file human/03-design-doc.md

# Comment lên issue
gh issue comment 123 --body "AI review: ..."
gh issue comment 123 --body-file review-comment.md

# Lấy comments của issue
gh issue view 123 --json comments

# List issues
gh issue list --state open --label "phase/03-design"
```

### Pull Requests

```bash
# Tạo PR
gh pr create --title "feat: cache layer" --body-file plan.md

# Comment tổng quát lên PR
gh pr comment 45 --body "AI review complete"

# Review với verdict
gh pr review 45 --comment -b "feedback"
gh pr review 45 --request-changes -b "needs work"
gh pr review 45 --approve

# Lấy PR comments + review threads
gh pr view 45 --comments

# Lấy review comments (inline, chi tiết) qua gh api
gh api repos/OWNER/REPO/pulls/45/comments
```

### Inline Comments (line-specific, cần gh api)

Skill "gh-get-review-comments" dùng gh api để retrieve tất cả review comments trên PR, output structured (id, path, line, body, user, in_reply_to_id) cho automation.

```bash
# Post inline comment vào line cụ thể
gh api repos/OWNER/REPO/pulls/45/comments \
  -f body="Consider error handling" \
  -f path="src/User.php" \
  -f commit_id="abc123..." \
  -F line=42 \
  -f side="RIGHT"

# Lấy tất cả inline comments (structured)
gh api repos/OWNER/REPO/pulls/45/comments \
  --jq '.[] | {id, path, line, body, user: .user.login, in_reply_to_id}'
```

---

## III. Workflow 1: Detail Design → GitHub Issue

### Flow

```
Phase 03 Design done (local)
    ↓
human/03-design-doc.md created
    ↓
Claude Code chạy: gh issue create --body-file
    ↓
Issue #123 created on GitHub
    ↓
Team review, comment trên issue
    ↓
Claude Code chạy: gh issue view --json comments
    ↓
Parse comments → fix/không fix
    ↓
Claude Code chạy: gh issue comment (reply)
```

### Skill Template

```markdown
# skills/phase03-to-github.md

# Phase 03: Push Design to GitHub Issue

## Step 1: Create Issue from Design

Sau khi human approve design-doc.md, chạy:

```bash
ISSUE_URL=$(gh issue create \
  --title "Design: $(jq -r '.feature_name' context/03-design.json)" \
  --body-file human/03-design-doc.md \
  --label "phase/03-design,ai-generated")

# Lưu issue number vào context
ISSUE_NUM=$(echo $ISSUE_URL | grep -oP '\d+$')
jq ".github_issue = $ISSUE_NUM" context/03-design.json > tmp.json && mv tmp.json context/03-design.json

echo "Created issue #$ISSUE_NUM: $ISSUE_URL"
```

## Step 2: Report Back to User

Claude thông báo: "Đã tạo Issue #$ISSUE_NUM. Team có thể review tại $ISSUE_URL"
```

---

## IV. Workflow 2: AI Review Loop (Comment ↔ Fix ↔ Reply)

Đây là phần cốt lõi bạn muốn. Chi tiết:

### Flow đầy đủ

```
1. AI review local → comment lên issue/PR
2. Human/reviewer comment phản hồi
3. Tool phát hiện comment mới
4. Tool lấy comment về
5. AI phân tích: fix hay không fix?
6. Nếu fix: sửa code + reply "đã fix"
7. Nếu không fix: reply "không fix vì [lý do]"
```

### Skill Template — Review Loop

```markdown
# skills/github-review-loop.md

# GitHub Review Loop — Fetch Comments, Fix, Reply

## Tool: gh CLI qua bash

### Step 1: AI Post Review Comment (Local → GitHub)

Sau khi AI review code local xong:

```bash
# Đọc AI review từ context
REVIEW=$(jq -r '.ai_review_summary' context/04-implement.json)

# Post lên issue
gh issue comment $ISSUE_NUM --body "$REVIEW"
```

### Step 2: Detect New Comments (Polling — Cross-platform Node.js)

Vì không có webhook, dùng polling — check comment mới. **Dùng Node.js script** (chạy y hệt trên Windows/Mac/Linux, không cần `jq`, `comm`, `grep`):

```bash
# Chạy script (Windows PowerShell, cmd, hay bash đều được)
node .claude/scripts/github-review-loop.mjs --issue 123
```

Output là JSON structured cho Claude đọc:

```json
{
  "status": "NEW_COMMENTS_FOUND",
  "count": 2,
  "target": "issue-123",
  "comments": [
    {
      "id": "IC_abc",
      "author": "reviewer1",
      "body": "Nên thêm try-catch cho Redis connection",
      "kind": "issue-comment"
    }
  ],
  "instructions": [
    "Với mỗi comment: đọc code context, quyết định FIX hay NO_FIX...",
    "Reply: gh issue comment 123 --body \"...\"",
    "Sau khi reply: node github-review-loop.mjs --mark-done --issue 123 --ids IC_abc"
  ]
}
```

Nếu không có comment mới, script in `NO_NEW_COMMENTS` — Claude dừng, không làm gì.

### Step 3: Fetch & Analyze Comment

Claude đọc JSON output ở Step 2 — mỗi comment đã có đủ `id`, `author`, `body` (và `path`, `line` nếu là PR inline comment). Không cần lệnh riêng để fetch từng comment.

### Step 4: AI Decides Fix / No-Fix

Claude phân tích comment và quyết định:

```
Đọc comment: "Nên thêm try-catch cho Redis connection"

Claude phân tích:
- Comment hợp lý? → CÓ (Redis có thể fail)
- Trong scope? → CÓ
- Quyết định: FIX

Đọc comment: "Đổi hết sang Memcached"

Claude phân tích:
- Contradicts Phase 01 fact (chỉ có Redis)? → CÓ
- Quyết định: KHÔNG FIX
- Lý do: "Memcached không có trên prod (Phase 01 đã xác nhận)"
```

### Step 5: Apply Fix (nếu FIX)

```bash
# Claude sửa code
# ... edit files ...

# Verify
php -l src/Cache/UserCache.php

# Commit
git add src/Cache/UserCache.php
git commit -m "fix: add Redis error handling (addressing comment #$COMMENT_ID)"
```

### Step 6: Reply to Comment

```bash
# Nếu ĐÃ FIX
gh issue comment $ISSUE_NUM --body "✅ **Đã fix** (comment của @$COMMENT_AUTHOR)

Đã thêm try-catch cho Redis connection tại src/Cache/UserCache.php:45.
Fallback về direct DB nếu Redis unavailable.

Commit: $(git rev-parse --short HEAD)"

# Nếu KHÔNG FIX
gh issue comment $ISSUE_NUM --body "❌ **Không fix** (comment của @$COMMENT_AUTHOR)

**Lý do:** Memcached không khả dụng trên production. 
Phase 01 Investigation đã xác nhận chỉ có Redis được cài đặt.
Giữ nguyên Redis theo design đã approve.

Tham khảo: context/semantic-facts.json (fact: 'Only Redis available')"

# Mark comment as processed (cross-platform)
node .claude/scripts/github-review-loop.mjs --mark-done --issue $ISSUE_NUM --ids $COMMENT_ID
```

## Complete Loop Script — Node.js (Cross-platform)

Script hoàn chỉnh: `.claude/scripts/github-review-loop.mjs` (file riêng đã cung cấp — `github-review-loop.mjs`).

**Yêu cầu:** Node.js 18+ và `gh` CLI. Không cần `jq` (dùng `gh --jq` built-in), không cần `comm`/`grep`/`sort`.

**2 modes:**

```bash
# Mode 1: Check comment mới (issue hoặc PR)
node .claude/scripts/github-review-loop.mjs --issue 123
node .claude/scripts/github-review-loop.mjs --pr 45

# Mode 2: Mark đã xử lý (sau khi Claude reply xong)
node .claude/scripts/github-review-loop.mjs --mark-done --issue 123 --ids IC_abc,IC_def
```

**Kỹ thuật quan trọng trong script:**
- `execFileSync("gh", args)` thay vì shell string — tránh lỗi quoting/escaping trên Windows cmd
- `gh --jq` flag — jq nhúng sẵn trong gh, không cần cài riêng
- State lưu `.claude/github-state.json` — cả processed IDs lẫn decision log (FIX/NO_FIX + lý do + commit)
- Comments bắt đầu bằng "🤖" tự động bỏ qua — tránh Claude tự trả lời comment của chính mình
- PR mode fetch cả general comments (`gh pr view`) lẫn inline comments (`gh api pulls/N/comments`) trong 1 lần chạy
```

---

## V. Workflow 3: Pull Request Review Loop

Tương tự issue nhưng cho PR với inline comments.

### Skill Template — PR Review

```markdown
# skills/github-pr-loop.md

# GitHub PR Review Loop

## Step 1: Create PR from Implementation

```bash
# Từ branch feature
gh pr create \
  --title "feat: $(jq -r '.feature_name' context/04-implement.json)" \
  --body-file human/04-implementation-plan.md \
  --label "phase/04-implement"

PR_NUM=$(gh pr view --json number --jq '.number')
```

## Step 2: AI Posts Review (Inline Comments)

AI review code, post inline comments vào từng line:

```bash
# Lấy commit SHA mới nhất
COMMIT_SHA=$(gh pr view $PR_NUM --json commits --jq '.commits[-1].oid')

# Post inline comment (line-specific)
gh api repos/{owner}/{repo}/pulls/$PR_NUM/comments \
  -f body="⚠️ AI: Function này có complexity cao (8), nên tách nhỏ" \
  -f path="src/Cache/UserCache.php" \
  -f commit_id="$COMMIT_SHA" \
  -F line=42 \
  -f side="RIGHT"
```

## Step 3: Fetch All Review Comments (Structured)

gh pr view chỉ lấy main review body, KHÔNG lấy line comments. Phải dùng gh api để lấy inline comments.

```bash
# Lấy inline review comments (structured)
gh api repos/{owner}/{repo}/pulls/$PR_NUM/comments \
  --jq '.[] | {
    id: .id,
    path: .path,
    line: .line,
    body: .body,
    user: .user.login,
    in_reply_to_id: .in_reply_to_id
  }' > /tmp/pr-review-comments.jsonl

# Lấy general PR comments
gh pr view $PR_NUM --json comments \
  --jq '.comments[] | {id, author: .author.login, body}' \
  > /tmp/pr-general-comments.jsonl
```

## Step 4: Analyze Each Comment + Fix/No-Fix

Best practice: "First use gh pr view --comments to get all comments, then list them classified by review thread. For each comment, read the relevant code context, propose a fix plan, apply changes, then use gh pr comment to reply collectively."

```bash
# Với mỗi inline comment
while IFS= read -r comment; do
  PATH=$(echo "$comment" | jq -r '.path')
  LINE=$(echo "$comment" | jq -r '.line')
  BODY=$(echo "$comment" | jq -r '.body')
  COMMENT_ID=$(echo "$comment" | jq -r '.id')
  
  # Claude đọc code context tại path:line
  # Claude phân tích comment
  # Claude quyết định fix/no-fix
  
done < /tmp/pr-review-comments.jsonl
```

## Step 5: Reply to Inline Comment (Threaded)

```bash
# Reply vào comment cụ thể (threaded reply)
gh api repos/{owner}/{repo}/pulls/$PR_NUM/comments \
  -f body="✅ Đã fix: tách function thành 2 helper methods" \
  -f in_reply_to="$COMMENT_ID"

# Hoặc reply tổng quát
gh pr comment $PR_NUM --body "Đã xử lý tất cả review comments:
- ✅ Line 42: tách function (complexity giảm còn 3)
- ❌ Line 55: giữ nguyên (lý do: pattern chuẩn của project)
- ✅ Line 60: thêm error handling"
```

## Step 6: Submit Overall Review Verdict

```bash
# Sau khi fix xong, request re-review hoặc self-status
gh pr comment $PR_NUM --body "🤖 AI đã địa chỉ tất cả comments. Sẵn sàng re-review."
```
```

---

## VI. Vì sao dùng Polling (không webhook)

Vì bạn không có CI/CD server, không nhận được webhook. Giải pháp: **polling** — chủ động check comment mới.

### 3 cách trigger polling

**Cách 1: Manual (đơn giản nhất)**
```
Bạn nói với Claude Code:
"Check comments trên issue #123 và xử lý"
→ Claude chạy loop script
```

**Cách 2: Scheduled (Windows Task Scheduler / cron)**
```powershell
# Windows — Task Scheduler (PowerShell, chạy 1 lần để đăng ký):
$action = New-ScheduledTaskAction -Execute "node" `
  -Argument ".claude\scripts\github-review-loop.mjs --issue 123" `
  -WorkingDirectory "C:\project"
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) `
  -RepetitionInterval (New-TimeSpan -Minutes 15)
Register-ScheduledTask -TaskName "GH-Review-Poll" -Action $action -Trigger $trigger
```
```bash
# Mac/Linux — crontab -e:
*/15 * * * * cd /project && node .claude/scripts/github-review-loop.mjs --issue 123
```

**Cách 3: Git hook (khi bạn pull/push)**
```bash
# .git/hooks/post-merge (Git tự chạy qua sh của Git for Windows — OK trên Windows)
#!/bin/sh
node .claude/scripts/github-review-loop.mjs --issue 123
```
Lưu ý: git hooks luôn chạy qua `sh` đi kèm Git for Windows nên gọi `node` bên trong là an toàn nhất — logic nằm trong `.mjs`, hook chỉ là 1 dòng trigger.

**Khuyến nghị:** Bắt đầu Cách 1 (manual) — bạn control khi nào xử lý. Sau này tự động hóa với Cách 2/3.

---

## VII. State Tracking — Tránh xử lý trùng

Vì polling, cần nhớ comment nào đã xử lý. **Tất cả state nằm trong 1 file duy nhất** — script `.mjs` tự đọc/ghi:

```
.claude/
└── github-state.json           ← Toàn bộ state (processed IDs + decisions)
```

`github-state.json` (format khớp với github-review-loop.mjs):
```json
{
  "processed": {
    "issue-123": ["IC_abc", "IC_def"],
    "pr-45": ["2345678", "2345679"]
  },
  "decisions": [
    {
      "comment_id": "IC_abc",
      "decision": "FIX",
      "action": "Added error handling",
      "commit": "abc1234"
    },
    {
      "comment_id": "IC_def",
      "decision": "NO_FIX",
      "reason": "Contradicts Phase 01 fact"
    }
  ]
}
```

**Lưu ý:** `processed` do script tự quản qua `--mark-done`. `decisions` do Claude append sau mỗi quyết định (audit trail — Phase 09 Review đọc lại được toàn bộ lịch sử fix/no-fix).

---

## VII-B. Multi-Round Loop — Khi human phản hồi lại quyết định của AI

Vòng 1 (comment → FIX/NO_FIX → reply) chưa đủ. Thực tế sẽ có:

- AI reply **NO_FIX** → human comment lại: *"Không đồng ý, phải fix theo hướng ABC"*
- AI reply **Đã fix** → human: *"Chưa đủ, rà soát thêm case XYZ"*

### Thread State Machine

Mỗi thread (bắt đầu từ 1 comment gốc) đi qua các trạng thái:

```
OPEN (comment mới)
  ↓ AI xử lý round 1
AI_REPLIED (FIX hoặc NO_FIX)
  ↓ human ack / im lặng          ↓ human phản hồi lại
RESOLVED                      REOPENED (round 2)
                                 ↓ AI xử lý theo quy tắc Human Override
                              AI_REPLIED (round 2)
                                 ↓ human vẫn chưa đồng ý
                              REOPENED (round 3)
                                 ↓
                              ESCALATED — AI dừng, tag Tech Lead quyết định
```

**Round 3 = hard stop.** Giống max-retries trong FSM Phase 04 — AI không tranh luận vô tận. Script tự flag `escalate: true` khi thread đạt round 3.

### Quy tắc Human Override (round 2+)

Human là final authority trong pipeline (human gates), nhưng có 1 ngoại lệ quan trọng:

| Tình huống | AI làm gì | Decision ghi lại |
|---|---|---|
| Human chỉ định hướng fix hợp lệ ("fix theo ABC") | Fix theo đúng hướng ABC, không tranh luận lại | `HUMAN_DIRECTED_FIX` |
| Human yêu cầu rà soát thêm | Rà soát, báo cáo kết quả (tìm thấy gì / không tìm thấy gì) | `FIX` hoặc `NO_FIX` round mới |
| Human yêu cầu **conflict với facts** (VD: "đổi sang Memcached" khi semantic-facts ghi prod chỉ có Redis) | KHÔNG fix mù. Reply nêu rõ conflict + evidence, đề nghị escalate | `CONFLICT_ESCALATED` |
| Round 3+ bất kể nội dung | Dừng. Tag Tech Lead, tóm tắt 2 quan điểm, chờ human gate | `ESCALATED` |

Điểm mấu chốt: AI **nhường quyền quyết định** nhưng **không nhường facts**. Nếu human muốn làm trái facts đã verify → đó là quyết định cần authority cao hơn (Tech Lead cập nhật semantic-facts.json trước, rồi AI fix theo).

### Thread tracking — Issue vs PR khác nhau

**PR inline comments:** GitHub có threading thật (`in_reply_to_id`) → script tự link comment mới vào thread cũ, attach sẵn `prior_decisions` + `round`.

**Issue comments:** GitHub issue comments là **flat list, không có threading**. Giải pháp 2 phần:
1. AI reply **luôn nhúng marker** `[thread:IC_abc]` (ID comment gốc) trong body
2. Script output kèm `all_decisions` — Claude đọc marker trong reply cũ + nội dung comment mới để tự match ngữ nghĩa: đây là thread mới hay phản hồi thread nào

### Decision record (mở rộng)

```json
{
  "thread_id": "IC_abc",
  "round": 2,
  "decision": "HUMAN_DIRECTED_FIX",
  "reason": "Human chỉ định dùng cache-aside pattern thay vì write-through",
  "commit": "def5678",
  "human_comment_id": "IC_xyz"
}
```

`decision` values: `FIX` | `NO_FIX` | `HUMAN_DIRECTED_FIX` | `CONFLICT_ESCALATED` | `ESCALATED`

### Ví dụ hoàn chỉnh — 3 rounds

```
Round 1:
  Human: "Nên đổi sang Memcached cho nhẹ"
  AI: "❌ Không fix [thread:IC_abc] — prod chỉ có Redis (Phase 01 verified)"
  → decision: NO_FIX, round 1

Round 2:
  Human: "Tôi biết, nhưng ops sắp cài Memcached tháng sau, cứ fix trước đi"
  Script: phát hiện comment mới, Claude match marker [thread:IC_abc]
          → round 2, prior_decisions có NO_FIX
  AI: "⚠️ [thread:IC_abc] Yêu cầu conflict với semantic-facts hiện tại
       ('Only Redis available'). Nếu Memcached được confirm cho tháng sau,
       đề nghị: (1) Tech Lead cập nhật facts + design, (2) sau đó tôi fix.
       Fix trước khi môi trường sẵn sàng sẽ làm code không chạy được trên prod."
  → decision: CONFLICT_ESCALATED, round 2

Round 3:
  Human: "Cứ làm đi, tôi chịu trách nhiệm"
  Script: round 3 → escalate: true
  AI: "🚨 [thread:IC_abc] Thread đạt round 3 — dừng theo protocol.
       @tech-lead cần quyết định: giữ Redis (theo facts) hay đổi Memcached
       (theo yêu cầu @reviewer, môi trường chưa sẵn sàng).
       Tóm tắt 2 quan điểm: [...]"
  → decision: ESCALATED. AI không xử lý thread này nữa cho tới khi
    Tech Lead comment quyết định + semantic-facts được cập nhật.
```

**Sau escalation:** Tech Lead comment quyết định → đó là comment mới → Claude xử lý như `HUMAN_DIRECTED_FIX` với authority đã đúng cấp (và cập nhật semantic-facts.json nếu facts thay đổi). Loop tiếp tục bình thường.

---

## VIII. Setup Ban Đầu

### 1. Prerequisites (Windows-friendly)

```bash
# gh CLI (nếu chưa có)
# Windows: winget install GitHub.cli
# Mac: brew install gh

# Node.js 18+ (team bạn đã có — MCP server dùng Node)
node --version

# KHÔNG cần: jq, WSL, Git Bash, bash — script .mjs chạy thuần Node
```

### 2. Auth gh CLI (1 lần)

```bash
gh auth login
# → Chọn GitHub.com, HTTPS, login qua browser

# Verify
gh auth status
gh repo set-default owner/your-repo
```

### 3. Test cơ bản

```bash
# Test tạo issue (chạy được từ PowerShell, cmd, hay terminal nào cũng OK)
gh issue create --title "Test" --body "Test body"

# Test comment
gh issue comment 1 --body "Test comment"

# Test lấy comments (gh --jq built-in, không cần cài jq)
gh issue view 1 --json comments --jq '.comments'

# Test loop script
node .claude/scripts/github-review-loop.mjs --issue 1
```

### 4. Tạo skill + scripts

```
.claude/
├── skills/
│   ├── phase03-to-github.md
│   ├── github-review-loop.md
│   └── github-pr-loop.md
├── scripts/
│   └── github-review-loop.mjs   ← 1 script duy nhất (issue + PR, check + mark-done)
└── github-state.json            ← Script tự tạo lần chạy đầu
```

---

## IX. Ví dụ Hoàn Chỉnh — End to End

```
# 1. Phase 03 design xong
Claude: "Design hoàn tất. Tạo issue nhé?"
Bạn: "OK"

# 2. Claude tạo issue
$ gh issue create --title "Design: Cache Layer" --body-file human/03-design-doc.md
→ Issue #123 created

# 3. AI review local, post comment
$ gh issue comment 123 --body "AI Review: Design ổn, nhưng thiếu Redis fallback"

# 4. Reviewer comment (trên GitHub web)
Reviewer: "Đồng ý, thêm fallback. Cũng nên dùng Memcached cho nhẹ"

# 5. Bạn nói Claude check
Bạn: "Check comments issue #123 và xử lý"

# 6. Claude fetch + analyze
$ gh issue view 123 --json comments
→ Comment 1: "thêm fallback" → FIX
→ Comment 2: "dùng Memcached" → NO_FIX (chỉ có Redis)

# 7. Claude fix + reply
$ [edit code, add fallback]
$ gh issue comment 123 --body "✅ Đã thêm Redis fallback (commit abc123)
❌ Không đổi Memcached: Phase 01 xác nhận prod chỉ có Redis"

# 8. Done — loop tiếp nếu có comment mới
```

---

## X. Ưu / Nhược điểm

### Ưu điểm

```
✅ Không cần CI/CD — chỉ gh CLI local
✅ Không cần API key riêng — gh tự quản auth
✅ Không cần webhook server — polling đủ dùng
✅ Full control — bạn quyết định khi nào sync
✅ Chi phí $0 thêm (ngoài Claude Max $100)
✅ Dùng được ngay hôm nay
```

### Nhược điểm

```
❌ Polling không real-time — có độ trễ (manual/15 phút)
❌ Inline PR comments cần gh api (verbose hơn gh pr)
❌ gh pr view không lấy line comments — phải dùng gh api
```

### Lưu ý (đặc biệt cho Windows)

```
⚠️ gh auth phải setup trước, token có scope đủ (repo, read/write)
⚠️ Rate limit: gh api có giới hạn ~5000 requests/giờ (đủ dùng)
⚠️ Comment ID format khác giữa issue (IC_) và PR inline (số)
⚠️ Luôn --mark-done để tránh reply trùng
⚠️ Test trên repo test trước khi chạy repo thật

Windows-specific:
⚠️ KHÔNG viết script bash (.sh) — jq/comm/grep không có sẵn trên Windows
⚠️ Dùng github-review-loop.mjs (Node) — chạy từ PowerShell/cmd đều OK
⚠️ Trong script, gọi gh bằng execFileSync (array args) — tránh lỗi
   quoting của cmd khi comment body chứa ký tự đặc biệt (", %, ^)
⚠️ Nếu comment body dài/nhiều dòng: dùng --body-file thay vì --body
   (paste multiline vào cmd/PowerShell dễ vỡ)
```

---

## XI. Nâng cấp sau này

```
Giai đoạn 1 (bây giờ):
  Manual polling — bạn nói Claude check comments

Giai đoạn 2 (khi quen):
  Cron job tự check mỗi 15 phút
  Git hook check khi pull

Giai đoạn 3 (khi có CI/CD):
  GitHub Actions webhook → real-time
  Không cần polling nữa
  gh CLI vẫn dùng được trong Actions
```

---

## XII. Tóm tắt

**Bạn nghĩ đúng:** dùng `gh` CLI làm tool trong Claude Code workflow.

**Cách hoạt động:**
```
Detail design.md → gh issue create → Issue trên GitHub
AI review local → gh issue comment → Comment lên issue
Reviewer comment → gh issue view --json → Claude lấy về
Claude phân tích → FIX/NO-FIX → sửa code
Claude reply → gh issue comment → "đã fix" / "không fix vì..."

Tương tự cho PR:
  gh pr create → gh api pulls/comments (inline) →
  fetch → analyze → fix → gh api reply
```

**Không cần:** CI/CD, webhook, API key riêng.  
**Chỉ cần:** `gh auth login` 1 lần, Claude Code Max $100.

**Trigger:** Polling (manual → cron → git hook).

**State:** `.claude/github-state.json` — script `--mark-done` tự quản, tránh xử lý trùng.

**Windows:** 1 script Node duy nhất (`github-review-loop.mjs`) — không bash, không jq, không WSL.

---

**Document Version:** 1.0  
**Status:** Ready — dùng được ngay hôm nay  
**Prerequisite:** gh CLI + gh auth login
