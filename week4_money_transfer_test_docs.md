# Money Transfer Feature — Test Documentation Pack

Feature under test: Transfers → Between my accounts (mobile app).
Built on my Week 3 test design (Tasks A, B, C). Test case IDs match Week 3.

## Requirements

| ID | Requirement |
|---|---|
| REQ-3.1 | Amount must be 100 – 500,000 KZT, whole tenge only |
| REQ-3.2 | Daily total + amount must not exceed 1,000,000 KZT |
| REQ-3.3 | Transfers above 100,000 KZT require an SMS code: valid 120 s, 3 attempts, then blocked |
| REQ-3.4 | Transfer must be rejected if balance is insufficient |
| REQ-3.5 | If amount is invalid, the amount error is shown even if the daily limit is also broken |

## Open questions for the analyst

Q1–Q7 are from Week 3; Q8–Q9 were added after the AI review (Appendix, Prompt 2).

| # | Question | Blocks |
|---|---|---|
| Q1 | Is a daily total of exactly 1,000,000 allowed? | REQ-3.2 boundary test |
| Q2 | Is the daily limit checked before sending the SMS or after the code is entered? | B3 expected result |
| Q3 | Do cancelled or expired transfers count in the daily total? | REQ-3.2 |
| Q4 | Is the limit per user, per account or per card? | REQ-3.2 |
| Q5 | Midnight by server time or phone time? What if midnight passes during the 120 s? | Daily reset tests |
| Q6 | What happens if the balance is not enough? Which error comes first? | REQ-3.4 (all tests) |
| Q7 | Does "above 100,000" mean 100,000 itself does not need SMS? | A3 expected result |
| Q8 | Does the 120 s start when the SMS is sent or when the code screen opens? | C4, DEF-302 |
| Q9 | Can the user request a new code? Does it reset the timer and attempts? | REQ-3.3 |

---

## 1. Test Plan

**Scope — in:** amount validation (REQ-3.1), daily limit (REQ-3.2), SMS code flow (REQ-3.3), error priority (REQ-3.5), balance and daily total after a transfer.

**Scope — out:**
- Insufficient balance (REQ-3.4): no test cases until the analyst answers Q6 — behaviour is undefined, so no expected result can be written. Explored only; see DEF-303.
- Transfers to other customers or banks, currency conversion, card transfers, SMS gateway delivery internals, performance testing, web version.

**Approach:** manual system testing on the mobile app. Techniques from Week 3: equivalence partitioning and BVA for amount, decision table for approval logic, state transition for the SMS code. 60-minute exploratory session on the SMS flow focused on the three dangerous invalid transitions. Confirmation and regression testing for every fixed defect.

**Environment:** test env 2, iOS 18.2 and Android 15, latest test build.

**Test data** (requested from the bank's test-data team at least 2 weeks before start):

| Customer | Balance | Daily total | Used in | Credentials |
|---|---|---|---|---|
| TC-014 | 1,000,000 KZT | 0 | A1–A6, C1–C7, DEF-301, DEF-302 | test-env password vault, entry "TC-014" |
| TC-021 | 1,000,000 KZT | 950,000 KZT | B3, B4 | test-env password vault, entry "TC-021" |
| TC-033 | 120,000 KZT | 0 | DEF-303 | test-env password vault, entry "TC-033" |

Plus access to the SMS test inbox for these customers.

**Where to check results:**
- Balance: Accounts screen, current account.
- Statement entries: Accounts → current account → Statement.
- Daily total: Transfers → Limits → "Used today".
- SMS: SMS test inbox.

**Entry criteria:**
- Build deployed to test env 2; smoke test (login, one 1,000 KZT transfer) passes
- TC-014, TC-021 and TC-033 loaded with the balances above
- SMS test inbox accessible
- Analyst has answered Q1, Q2 and Q7

**Exit criteria:**
- All 12 documented test cases executed
- Zero open critical or high defects
- Boundary tests at 99/100, 100,000/100,001 and 500,000/500,001 passed (A2, A1, A3, A4, C1, A5)
- 6 of 9 valid SMS transitions passed via C1, C3, C4, C7; the other 3 (Waiting 2 → Confirmed, Waiting 2 → Expired, Waiting 3 → Expired) run from Week 3 C2, C5, C6 and logged
- The three dangerous invalid transitions tested in the exploratory session
- Release checklist (section 4) fully ticked
- Uncovered requirements (REQ-3.4) and residual risks accepted in writing by the product owner

**Top three product risks** (from Week 3 invalid transitions):
1. Confirmed + correct code: double tap or network retry sends the money twice.
2. Expired + correct code: a late SMS still works, money sent without a valid code.
3. Confirmed + 120 s pass: the timer cancels a transfer that was already sent, leaving balance and statement inconsistent.

---

## 2. Test Cases

Common for all cases: test env 2, latest test build, recipient = own deposit ending 4417. Before each case, reset the customer to the balance and daily total in its preconditions (test-data team reset tool), so cases do not depend on run order.

### A1 · REQ-3.1 · Minimum amount 100

**Preconditions:** logged in as TC-014, balance 1,000,000 KZT, daily total 0

**Steps:**
1. Open Transfers → Between my accounts
2. Select deposit ending 4417
3. Enter 100
4. Tap Continue
5. Tap Confirm

**Expected:** no SMS code screen; success screen; balance 999,900 KZT

**Postcondition:** daily total (Transfers → Limits) 100

### A2 · REQ-3.1 · Below minimum, 99

**Preconditions:** logged in as TC-014, balance 1,000,000 KZT, daily total 0

**Steps:**
1. Open Transfers → Between my accounts
2. Select deposit ending 4417
3. Enter 99
4. Tap Continue

**Expected:** amount error shown; screen does not proceed; no SMS in test inbox

**Postcondition:** balance 1,000,000; daily total (Transfers → Limits) 0

### A3 · REQ-3.3 · Exactly 100,000, no SMS

**Preconditions:** logged in as TC-014, balance 1,000,000 KZT, daily total 0

**Steps:**
1. Open Transfers → Between my accounts
2. Select deposit ending 4417
3. Enter 100,000
4. Tap Continue
5. Tap Confirm

**Expected:** no SMS code screen, no SMS in test inbox; success screen; balance 900,000 KZT

**Postcondition:** daily total (Transfers → Limits) 100,000

**Note:** expected result depends on Q7.

### A4 · REQ-3.3 · 100,001 requires SMS

**Preconditions:** logged in as TC-014, balance 1,000,000 KZT, daily total 0

**Steps:**
1. Start a screen recording
2. Open Transfers → Between my accounts
3. Select deposit ending 4417
4. Enter 100,001
5. Tap Continue
6. Tap Back to cancel the transfer

**Expected:** after step 5, SMS code screen within 3 s (timed on the screen recording); SMS in test inbox within 30 s

**Postcondition:** no money sent; balance 1,000,000; daily total (Transfers → Limits) 0

### A5 · REQ-3.1 · Above maximum, 500,001

**Preconditions:** logged in as TC-014, balance 1,000,000 KZT, daily total 0

**Steps:**
1. Open Transfers → Between my accounts
2. Select deposit ending 4417
3. Enter 500,001
4. Tap Continue

**Expected:** amount error; screen does not proceed; no SMS in test inbox

**Postcondition:** balance 1,000,000; daily total (Transfers → Limits) 0

### A6 · REQ-3.1 · Not whole tenge, 150.50

**Preconditions:** logged in as TC-014, balance 1,000,000 KZT, daily total 0

**Steps:**
1. Open Transfers → Between my accounts
2. Select deposit ending 4417
3. Enter 150.50
4. Tap Continue

**Expected:** format error; screen does not proceed; no SMS in test inbox

**Postcondition:** balance 1,000,000; daily total (Transfers → Limits) 0

### B3 · REQ-3.2 · Daily limit exceeded (decision table columns 3–4)

**Preconditions:** logged in as TC-021, balance 1,000,000 KZT, daily total 950,000 KZT

**Steps:**
1. Open Transfers → Between my accounts
2. Select deposit ending 4417
3. Enter 60,000
4. Tap Continue

**Expected:** daily limit error; screen does not proceed; no SMS in test inbox

**Postcondition:** balance 1,000,000; daily total (Transfers → Limits) 950,000

**Notes:** balance is kept high on purpose so insufficient balance cannot be the cause of rejection. "No SMS" assumes the limit is checked before the SMS (Q2).

### B4 · REQ-3.5 · Invalid amount and broken limit together (columns 5–8)

**Preconditions:** logged in as TC-021, balance 1,000,000 KZT, daily total 950,000 KZT

**Steps:**
1. Open Transfers → Between my accounts
2. Select deposit ending 4417
3. Enter 600,000
4. Tap Continue

**Expected:** amount error (not daily limit error); screen does not proceed; no SMS in test inbox

**Postcondition:** balance 1,000,000; daily total (Transfers → Limits) 950,000

### C1 · REQ-3.3, REQ-3.1 · Correct code on first attempt, maximum amount 500,000

**Preconditions:** logged in as TC-014, balance 1,000,000 KZT, daily total 0

**Steps:**
1. Open Transfers → Between my accounts
2. Select deposit ending 4417
3. Enter 500,000
4. Tap Continue
5. Enter the code from the test inbox
6. Tap Confirm

**Expected:** amount accepted (upper boundary); success screen; balance 500,000 KZT; exactly one statement entry of 500,000 (Accounts → current account → Statement)

**Postcondition:** daily total (Transfers → Limits) 500,000

### C3 · REQ-3.3 · Correct code on third attempt

**Preconditions:** logged in as TC-014, balance 1,000,000 KZT, daily total 0

**Steps:**
1. Open Transfers → Between my accounts
2. Select deposit ending 4417
3. Enter 150,000
4. Tap Continue
5. Take the code from the test inbox, change its last digit, enter it, tap Confirm
6. Repeat step 5
7. Enter the real code from the test inbox, tap Confirm

**Expected:** after steps 5 and 6, wrong code message and the code screen stays; after step 7, success screen; balance 850,000 KZT; exactly one statement entry of 150,000

**Postcondition:** daily total (Transfers → Limits) 150,000

### C4 · REQ-3.3 · Code expires after 120 s

**Preconditions:** logged in as TC-014, balance 1,000,000 KZT, daily total 0

**Steps:**
1. Open Transfers → Between my accounts
2. Select deposit ending 4417
3. Enter 150,000
4. Tap Continue
5. Wait 125 seconds (device clock) without entering a code

**Expected:** message that the code expired; transfer cancelled

**Postcondition:** balance 1,000,000; daily total (Transfers → Limits) 0; no new statement entry

**Note:** timer start point depends on Q8.

### C7 · REQ-3.3 · Three wrong codes block the transfer

**Preconditions:** logged in as TC-014, balance 1,000,000 KZT, daily total 0

**Steps:**
1. Open Transfers → Between my accounts
2. Select deposit ending 4417
3. Enter 150,000
4. Tap Continue
5. Take the code from the test inbox, change its last digit, enter it, tap Confirm
6. Repeat step 5
7. Repeat step 5

**Expected:** after step 7, message that the transfer is blocked; transfer cancelled; user returned to the Transfers screen

**Postcondition:** balance 1,000,000; daily total (Transfers → Limits) 0; no new statement entry

---

## 3. Traceability Matrix

| Requirement | Test cases (documented) | Also designed in Week 3 | Technique | Coverage | Why not fully covered |
|---|---|---|---|---|---|
| REQ-3.1 Amount 100 – 500,000, whole tenge | A1, A2, A5, A6, C1 | 3-value neighbours (98, 101, 499,999, 500,002); "abc"; empty | Partitions + BVA | Partially covered | 2-value BVA documented at both ends. 3-value neighbours and non-numeric / empty partitions designed in Week 3 but not documented (12-case limit) |
| REQ-3.2 Daily limit 1,000,000 | B3 | B1, B2 | Decision table | Partially covered | Exactly 1,000,000 not tested (Q1). Daily reset not tested (Q5) and test env has no time control. Q3 and Q4 unanswered |
| REQ-3.3 SMS code | A3, A4, C1, C3, C4, C7 | C2, C5, C6 | BVA + state transition | Partially covered | 6 of 9 valid transitions in documented cases; Waiting 2 → Confirmed, Waiting 2 → Expired, Waiting 3 → Expired only in Week 3 C2, C5, C6. Invalid transitions covered only by exploratory testing and checklist. A3 depends on Q7 |
| REQ-3.4 Insufficient balance | — | — | — | NOT COVERED | Behaviour undefined (Q6): no error text, no order relative to the limit check or SMS. Cannot write an expected result. Explored in DEF-303 |
| REQ-3.5 Amount error first | B4 | — | Decision table | Covered | — |

---

## 4. Release Checklist — SMS Code Flow

- [ ] SMS code is asked for 100,001 KZT and not for 100,000 KZT
- [ ] Code arrives within 30 seconds
- [ ] Code expires after 120 seconds
- [ ] After expiry, any code (correct or wrong) is rejected and does not restart the flow
- [ ] Wrong code with attempts left shows an error and keeps the code screen
- [ ] Correct code on the third attempt completes the transfer
- [ ] Third wrong code blocks the transfer; any later code is rejected; no money sent
- [ ] Confirm tapped twice sends money once
- [ ] Transfer stays sent if 120 s pass after Confirmed
- [ ] No SMS is sent when the daily limit is already broken
- [ ] SMS text correct in KZ, RU and EN
- [ ] Balance and daily total update after Confirmed, not before

---

## 5. Defect Reports

Mock reports, based on the dangerous invalid transitions and requirement gaps from Week 3.

### DEF-301 — SMS confirmation: money sent twice when Confirm is tapped twice

- **Environment:** Android 15, app 4.11.0 build 2291, test env 2
- **Preconditions:** logged in as TC-014, balance 1,000,000 KZT, daily total 0
- **Steps to reproduce:**
  1. Open Transfers → Between my accounts
  2. Select deposit ending 4417
  3. Enter 150,000, tap Continue
  4. Enter the code from the SMS test inbox
  5. Double-tap Confirm rapidly with one finger (two taps within 0.5 s)
  6. Open Accounts → current account → Statement
- **Expected result:** one transfer; balance 850,000; daily total 150,000
- **Actual result:** two statement entries of 150,000; balance 700,000; daily total 300,000
- **Reproducibility:** 3 of 5 attempts. If not reproduced, reset TC-014 and repeat from step 1, up to 5 times.
- **Severity / priority:** Critical / Critical (proposed)
- **Evidence:** screen recording, statement screenshot, both request IDs
- **Traces to:** REQ-3.3 / C1; invalid transition "Confirmed + correct code"

### DEF-302 — SMS confirmation: code accepted after 120 seconds

- **Environment:** iOS 18.2, app 4.11.0 build 2291, test env 2
- **Preconditions:** logged in as TC-014, balance 1,000,000 KZT, daily total 0
- **Steps to reproduce:**
  1. Open Transfers → Between my accounts
  2. Select deposit ending 4417
  3. Enter 150,000, tap Continue
  4. Note the code from the SMS test inbox
  5. Press Home and wait 130 s (device clock)
  6. Return to the app, enter the code, tap Confirm
- **Expected result:** code rejected as expired; no money sent
- **Actual result:** success screen; balance 850,000
- **Reproducibility:** 5 of 5 attempts
- **Severity / priority:** High / High (proposed)
- **Evidence:** screen recording with device clock visible, request ID
- **Traces to:** REQ-3.3 / C4; invalid transition "Expired + correct code"

### DEF-303 — Transfers: insufficient balance shown only after SMS code entry, with generic error

Raised from a requirement gap (REQ-3.4, Q6).

- **Environment:** iOS 18.2, app 4.11.0 build 2291, test env 2
- **Preconditions:** logged in as TC-033, balance 120,000 KZT, daily total 0
- **Steps to reproduce:**
  1. Open Transfers → Between my accounts
  2. Select deposit ending 4417
  3. Enter 120,001, tap Continue
  4. Enter the code from the SMS test inbox, tap Confirm
- **Expected result:** not defined in requirements. Proposed: insufficient funds error after step 3, before the SMS is sent
- **Actual result:** SMS sent after step 3; after step 4, message "Something went wrong"; balance unchanged
- **Reproducibility:** 5 of 5 attempts
- **Severity / priority:** Medium / Medium (proposed)
- **Evidence:** screenshots after steps 3 and 4, request ID
- **Traces to:** REQ-3.4 (not covered) — clarification requested from analyst (Q6)

---

## 6. AI Appendix (Level 1)

**Tool:** Claude (Anthropic).

### 6.1 Review

**Prompt 1 — test cases against the four properties**

```
Here are 12 test cases I wrote for a money transfer feature: [paste].
Check each against four properties: self-contained preconditions,
concrete data, observable expected result, independence from other tests.
Only list problems. Do not rewrite the test cases.
```

**Raw output (excerpt):**
- Steps 1–2 defined once at the top, not inside each case — cases are not fully self-contained.
- A1, A3, C1, C3 change TC-014's balance and daily total, but no reset step — later cases depend on run order.
- A2, A5, A6, B3, B4: "amount error" / "format error" / "daily limit error" have no exact message text — not fully observable.
- "No SMS sent" does not say where to check (test inbox).
- A3 expected result depends on unanswered analyst Q7.
- C4: "message that the code expired" — no exact text.

**What I changed and why:** Added a reset rule before each case so tests do not depend on run order. Changed "no SMS sent" to "no SMS in test inbox" so the check is observable. Kept generic error names: exact texts are not in the requirements.

**Prompt 2 — gaps in requirements**

```
Requirements: [paste REQ-3.1 – 3.5].
List situations where expected behaviour is not defined.
Do not propose answers, only questions for the analyst.
```

**Raw output (excerpt):**
- Is exactly 1,000,000 daily total allowed?
- Does exactly 100,000 need an SMS ("above" = > or ≥)?
- In what order are amount, limit, balance and SMS checked?
- Do cancelled, expired or blocked transfers count toward the daily total?
- When does the daily total reset, and by which clock/timezone?
- What error and when for insufficient balance?
- Can the user request a new code? Does it reset the timer and attempts?
- Does 120 s start when the code is sent or when the screen opens?
- After Blocked, can the user start a new transfer right away?
- What happens to empty, non-numeric, or leading-zero input?
- What happens if the app is closed during code entry?

**Comparison with my 7 Week 3 analyst questions:** AI repeated my questions on the limit boundary, check order, SMS at 100,000, cancelled transfers, daily reset and insufficient balance. New from AI: code resend, when the 120 s starts, retry after Blocked, app closed during code entry, input format. I added the timer-start and resend questions to my analyst list (Q8, Q9).

**Prompt 3 — notes to defect report**

```
Here are my raw notes from a test session: [raw notes — see notes/late-code-session.md].
Turn them into a defect report with exactly these fields: title,
environment, preconditions, steps to reproduce, expected result,
actual result, reproducibility, severity, priority, evidence.
Rules:
- Use only facts present in my notes. If a field is missing, write MISSING.
- Steps: numbered, one action each, from a known state.
- Title format: [screen]: [what happens] [under what condition].
- Mark severity and priority as proposals.
```

**Raw output (excerpt)** (notes from the late-code session, became DEF-302):

- **Title:** SMS confirmation: code accepted after 120 seconds
- **Environment:** iOS 18.2, test env 2, app build MISSING
- **Preconditions:** TC-014, balance 1,000,000, daily total 0
- **Steps:** 1. Open Transfers → Between my accounts 2. Select deposit 4417 3. Enter 150,000, tap Continue 4. Note code 5. Background app 130 s 6. Return, enter code, tap Confirm
- **Expected:** code rejected as expired, no money sent
- **Actual:** success screen, balance 850,000
- **Reproducibility:** 5/5
- **Severity / priority:** High / High (proposed)
- **Evidence:** MISSING

**Note:** the original raw notes were lost. `notes/late-code-session.md` is a reconstruction generated with Claude afterwards (Prompt 8), not the input actually used for this prompt.

**MISSING fields and how I filled them:** App build — taken from the app's About screen (4.11.0 build 2291). Evidence — added screen recording with device clock visible and the request ID.

**Prompt 4 — checklist against state table**

```
Here is my SMS code state table: [paste]. Here is my release checklist: [paste].
Which states or transitions from the table are not reflected in the checklist?
```

**Raw output (excerpt):**
- Wrong code with attempts left (shows message, stays in Awaiting code) — no item.
- Correct code on 2nd or 3rd attempt → Confirmed — no item.
- Expired + wrong code — no item.
- Code resend (if in the table) and its effect on timer/attempts — no item.
- User cancels while Awaiting code — no item.

**What I changed and why:** Added three checklist items: wrong code with attempts left, correct code on third attempt, wrong code after expiry. Did not add resend or user cancel: they are not in my state table.

**Prompt 5 — grader's view**

```
A grader will pick one test case and one defect report at random and try
to follow them exactly as written. Here they are: [paste A4 and DEF-301].
Where would the grader get stuck or have to guess?
```

**Raw output (excerpt):**

A4:
- Must scroll to the top to find what steps 1–2 are.
- No login credentials for TC-014.
- Does not say where to see the daily total to check the postcondition.
- "Within 3 s" — no way to measure stated.
- Transfer is left in Awaiting code — unclear whether to cancel it.

DEF-301:
- Preconditions do not say "logged in".
- Tapping twice within 0.5 s is hard to do reliably by hand; with 3/5 reproducibility the grader may not see the bug.
- Does not say where to see statement entries and daily total.

**What I changed and why:** A4 — wrote steps inline, added "logged in", timing by screen recording, and a cancel step to leave a clean state. DEF-301 — added "logged in" and a step to open the statement. Kept the 0.5 s double tap: it is the real trigger, and the screen recording shows it.

**Prompt 6 — final check against the assignment brief**

```
Here is my assignment (Google Doc), my Week 3 test design, and the Week 4
slides with the assignment task. Assess the assignment against the task,
tell me what to fix, then fix it and return Markdown.
```

**Raw output (excerpt, summary of findings):** checklist had 15 items (limit 12); steps 1–2 inline only in A4; "logged in" missing in most preconditions and in DEF-302/303; no credentials or place to check daily total and statement; DEF-303 used an unnamed customer; DEF-301 had no instruction for repeated attempts; REQ-3.1 marked "Covered" with no test at 500,000, while the exit criteria required one; exit criteria claimed all 9 SMS transitions although only 6 are in documented cases; Q1–Q9 referenced but never listed; REQ-3.4 missing from scope; placeholder text in Prompt 3; "000000" as a wrong code could match a real code. The AI then produced a corrected version of this document.

**What I changed and why:** Accepted all fixes after checking them against Week 3. Notable decisions: changed C1 from 150,000 to 500,000 so the upper valid boundary is tested without exceeding the 12-case limit; downgraded REQ-3.1 and REQ-3.3 to "Partially covered" to match what is actually documented; merged checklist items to fit 12 instead of dropping any state-table transition.

**Prompt 7 — final clean-up**

```
Clean up all the mess and return the final Markdown.
```

**Raw output (excerpt):** the final version of this document: "(excerpt)" labels added to raw outputs that were shortened in this appendix, consistent formatting across all sections, and this prompt recorded.

**What I changed and why:** Reviewed the final file against the assignment brief and my Week 3 design before submitting; no content changes beyond the labels, so the appendix states plainly which outputs are shortened.


### 6.2 Decisions that stayed mine

- Severity and priority in all defect reports: proposed by me; AI only formatted them as proposals. Final priority is the product owner's.
- AI suggestions I rejected: exact error message texts (not defined in requirements); resend and cancel checklist items (not in my state table).
