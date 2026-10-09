# Week 4: Money Transfer Feature — Documentation Pack

**Student:** Assylbekov Bekassyl\
**Course:** Software Testing and Debugging, KBTU 2026

## Contents

- [Requirements](#requirements)
- [1. Test plan](#1-test-plan)
- [2. Test cases](#2-test-cases)
- [3. Traceability matrix](#3-traceability-matrix)
- [4. SMS code release checklist](#4-sms-code-release-checklist)
- [5. Defect reports](#5-defect-reports)
- [6. AI appendix (Level 1)](#6-ai-appendix-level-1)

---

## Requirements

| ID    | Requirement |
|-------|-------------|
| REQ-1 | Transfer amount is 100 to 500,000 KZT per transfer, whole tenge only. |
| REQ-2 | Daily limit is 1,000,000 KZT across all transfers, reset at midnight Almaty time. |
| REQ-3 | Transfers above 100,000 KZT require a 6-digit SMS code. |
| REQ-4 | The SMS code is valid for 120 seconds. |
| REQ-5 | Three wrong code attempts cancel the transfer. |
| REQ-6 | If the amount and the daily limit are both violated, the amount error is shown. |

---

## 1. Test plan

### Scope

**In scope:**

- Amount validation: 100–500,000 KZT, whole tenge only (REQ-1)
- Daily limit and the order of errors (REQ-2, REQ-6)
- SMS code: threshold, 120-second validity, three attempts (REQ-3, REQ-4, REQ-5)
- Balance and daily total after every executed, rejected, expired and blocked transfer

**Out of scope:**

- Real SMS operator delivery (a test SMS inbox is used)
- Performance and load testing
- Security testing of the SMS code generation
- Web and API channels (mobile app only)
- Insufficient balance: no requirement exists (see [DEF-03](#def-03))

### Approach

Manual system testing on the test environment, iOS and Android.

| Requirement | Technique |
|-------------|-----------|
| Amount (REQ-1) | Equivalence partitioning + 3-value boundary value analysis |
| Amount × daily limit × SMS threshold (REQ-2, REQ-3, REQ-6) | Decision table |
| SMS code (REQ-3, REQ-4, REQ-5) | State transition, valid-transitions coverage + 3 invalid transitions |

### Entry criteria

1. The build is deployed to the test environment and its build number is recorded.
2. Smoke test passes: login, open Transfers, a 1,000 KZT transfer executes.
3. Test customers TC-001, TC-002, TC-003 exist with the balances and daily totals from [section 2](#test-data).
4. SMS codes are visible in the test SMS inbox.

### Exit criteria

1. All 12 test cases executed; each one passed, or failed with a linked defect.
2. Zero open defects of severity Critical or High.
3. All 12 three-value boundary values of the amount executed and passed.
4. All 12 items of the SMS release checklist checked.
5. Every requirement not covered is listed in the traceability matrix, and the risk is accepted in writing by the product owner.

### Top three product risks

| #  | Risk | Impact | Covered by |
|----|------|--------|------------|
| R1 | Confirm tapped twice debits the account twice | Customer loses money | TC-12, DEF-01 |
| R2 | Daily limit resets at UTC midnight instead of Almaty midnight | Customers blocked or limit bypassed for 5 hours | TC-09, DEF-02 |
| R3 | Correct code accepted after expiry or after three wrong attempts | SMS protection does not work | TC-10, TC-11 |

---

## 2. Test cases

### Test data

| Test customer | Balance, KZT | Daily total used today, KZT |
|---------------|-------------:|----------------------------:|
| TC-001        | 1,000,000    | 0                           |
| TC-002        | 2,000,000    | 500,000                     |
| TC-003        | 1,000,000    | 900,000                     |

- Test data is reset before each test case, so the test cases are independent.
- Recipient in every test: test customer TC-090, card ending 5512.
- Balance: **Accounts** screen. Daily total: **Transfers > Limits > "Used today"**.
- SMS code: test SMS inbox. A wrong code is the correct code with the last digit changed.

### TC-01. Amount 99 KZT is rejected

| Field | Value |
|-------|-------|
| ID | TC-01 |
| Traces to | REQ-1 lower limit, partition EP1, boundary value 99 |
| Preconditions | Logged in as TC-001: balance 1,000,000, daily total 0 |
| Steps | 1. Open Transfers > To another client > card ending 5512<br>2. Enter amount 99<br>3. Tap Continue |
| Expected result | Error "amount outside limits" shown. No SMS code screen; no SMS in the test SMS inbox. Balance 1,000,000. Daily total 0. |
| Postconditions | No transfer created |

### TC-02. Amount 100 KZT executes without SMS code

| Field | Value |
|-------|-------|
| ID | TC-02 |
| Traces to | REQ-1 lower limit, REQ-3, EP2, boundary value 100 |
| Preconditions | Logged in as TC-001: balance 1,000,000, daily total 0 |
| Steps | 1. Open Transfers > To another client > card ending 5512<br>2. Enter amount 100<br>3. Tap Continue |
| Expected result | Transfer executed immediately, success screen shown, no SMS code screen. Balance 999,900. Daily total 100. |
| Postconditions | One completed transfer of 100 KZT in the statement |

### TC-03. Amount 100,000 KZT executes without SMS code

| Field | Value |
|-------|-------|
| ID | TC-03 |
| Traces to | REQ-3 SMS threshold, EP2, boundary value 100,000 |
| Preconditions | Logged in as TC-001: balance 1,000,000, daily total 0 |
| Steps | 1. Open Transfers > To another client > card ending 5512<br>2. Enter amount 100000<br>3. Tap Continue |
| Expected result | Transfer executed immediately, no SMS code screen, no SMS in the test SMS inbox. Balance 900,000. Daily total 100,000. |
| Postconditions | One completed transfer of 100,000 KZT |

### TC-04. Amount 100,001 KZT requires an SMS code

| Field | Value |
|-------|-------|
| ID | TC-04 |
| Traces to | REQ-3 SMS threshold, EP3, boundary value 100,001 |
| Preconditions | Logged in as TC-001: balance 1,000,000, daily total 0 |
| Steps | 1. Open Transfers > To another client > card ending 5512<br>2. Enter amount 100001<br>3. Tap Continue |
| Expected result | SMS code screen shown. One SMS with a 6-digit code in the test SMS inbox. Balance still 1,000,000. Daily total still 0. |
| Postconditions | Transfer pending confirmation (state W1). Close the screen without entering a code. |

### TC-05. Amount 500,000 KZT is accepted and executes after the code

| Field | Value |
|-------|-------|
| ID | TC-05 |
| Traces to | REQ-1 upper limit, REQ-3, EP3, boundary value 500,000 |
| Preconditions | Logged in as TC-001: balance 1,000,000, daily total 0 |
| Steps | 1. Open Transfers > To another client > card ending 5512<br>2. Enter amount 500000<br>3. Tap Continue<br>4. Enter the correct code from the test SMS inbox and tap Confirm<br>5. Open the Accounts screen and Transfers > Limits |
| Expected result | After step 3: SMS code screen shown (500,000 is accepted, not rejected).<br>After step 4: success screen.<br>Step 5: balance 500,000. Daily total 500,000. |
| Postconditions | One completed transfer of 500,000 KZT |

### TC-06. Amount 500,001 KZT is rejected

| Field | Value |
|-------|-------|
| ID | TC-06 |
| Traces to | REQ-1 upper limit, EP4, boundary value 500,001 |
| Preconditions | Logged in as TC-001: balance 1,000,000, daily total 0 |
| Steps | 1. Open Transfers > To another client > card ending 5512<br>2. Enter amount 500001<br>3. Tap Continue |
| Expected result | Error "amount outside limits" shown. No SMS code screen, no SMS in the test SMS inbox. Balance 1,000,000. Daily total 0. |
| Postconditions | No transfer created |

### TC-07. Amount with tiyn (150.50) is not accepted

| Field | Value |
|-------|-------|
| ID | TC-07 |
| Traces to | REQ-1 whole tenge only, EP5 |
| Preconditions | Logged in as TC-001: balance 1,000,000, daily total 0 |
| Steps | 1. Open Transfers > To another client > card ending 5512<br>2. Type 150.50 into the amount field (also try the comma form 150,50)<br>3. Tap Continue |
| Expected result | Either the field does not accept the decimal separator, or an error "whole tenge only" is shown. No transfer is created at any amount: not 150, not 151, not 150.50. Balance 1,000,000. Daily total 0. |
| Postconditions | No transfer created |

### TC-08. Amount and daily limit both violated: amount error wins

| Field | Value |
|-------|-------|
| ID | TC-08 |
| Traces to | REQ-6, REQ-1, REQ-2 |
| Preconditions | Logged in as TC-002: balance 2,000,000, daily total 500,000 |
| Steps | 1. Open Transfers > To another client > card ending 5512<br>2. Enter amount 600000<br>3. Tap Continue |
| Expected result | Error "amount outside limits" shown, not the daily limit error. No SMS code. Balance 2,000,000. Daily total 500,000. |
| Postconditions | No transfer created |

### TC-09. Daily limit exceeded with a valid amount is rejected

| Field | Value |
|-------|-------|
| ID | TC-09 |
| Traces to | REQ-2 daily limit, REQ-3 |
| Preconditions | Logged in as TC-003: balance 1,000,000, daily total 900,000 |
| Steps | 1. Open Transfers > To another client > card ending 5512<br>2. Enter amount 200000<br>3. Tap Continue |
| Expected result | Error "daily limit exceeded" shown. No SMS code screen and no SMS in the test SMS inbox. Balance 1,000,000. Daily total 900,000. |
| Postconditions | No transfer created |

### TC-10. Three wrong codes block the transfer; a correct code afterwards is rejected

| Field | Value |
|-------|-------|
| ID | TC-10 |
| Traces to | REQ-5, REQ-3 |
| Preconditions | Logged in as TC-001: balance 1,000,000, daily total 0 |
| Steps | 1. Start a transfer of 150000 to card ending 5512, tap Continue<br>2. Enter a wrong code, tap Confirm<br>3. Enter a wrong code, tap Confirm<br>4. Enter a wrong code, tap Confirm<br>5. If the code field is still available, enter the correct code and tap Confirm |
| Expected result | Steps 2–3: "wrong code" message, code screen stays.<br>Step 4: "transfer cancelled" message, code entry closed.<br>Step 5: code entry not possible, or code rejected.<br>Final: balance 1,000,000, daily total 0, no completed transfer in the statement. |
| Postconditions | Transfer cancelled (state BLK) |

### TC-11. Code expires after 120 s; a correct code afterwards is rejected

| Field | Value |
|-------|-------|
| ID | TC-11 |
| Traces to | REQ-4, REQ-3 |
| Preconditions | Logged in as TC-001: balance 1,000,000, daily total 0. A stopwatch. |
| Steps | 1. Start a transfer of 150000 to card ending 5512, tap Continue; start the stopwatch when the SMS code screen appears<br>2. Do nothing for 130 seconds<br>3. Enter the correct code from the test SMS inbox and tap Confirm |
| Expected result | After step 2: "code expired / transfer cancelled" message.<br>Step 3: code rejected (or entry not possible). Balance 1,000,000. Daily total 0. No completed transfer in the statement. |
| Postconditions | Transfer cancelled (state EXP) |

### TC-12. Confirm tapped twice debits only once

| Field | Value |
|-------|-------|
| ID | TC-12 |
| Traces to | REQ-3, product risk R1 (double debit) |
| Preconditions | Logged in as TC-001: balance 1,000,000, daily total 0 |
| Steps | 1. Start a transfer of 150000 to card ending 5512, tap Continue<br>2. Enter the correct code from the test SMS inbox<br>3. Tap Confirm twice quickly (double tap)<br>4. Open the Accounts screen and the statement |
| Expected result | One success screen. Balance 850,000 (not 700,000). Daily total 150,000. Exactly one transfer of 150,000 in the statement. |
| Postconditions | One completed transfer of 150,000 KZT |

---

## 3. Traceability matrix

| Requirement | Test cases | Technique | Status |
|-------------|------------|-----------|--------|
| REQ-1 Amount 100–500,000, whole tenge | TC-01, TC-02, TC-05, TC-06, TC-07 | Partitions + 3-value BVA | Covered |
| REQ-2 Daily limit 1,000,000 | TC-08, TC-09 | Decision table | Partly covered (midnight reset not covered) |
| REQ-3 SMS code above 100,000 | TC-02, TC-03, TC-04, TC-05, TC-12 | BVA + state transition | Covered |
| REQ-4 Code valid 120 s | TC-11 | State transition | Covered |
| REQ-5 Three wrong attempts | TC-10 | State transition | Covered |
| REQ-6 Amount error shown first | TC-08 | Decision table | Covered |
| Insufficient balance (no requirement) | – | – | **NOT COVERED** |

### Not covered, and why

| Item | Why | What is needed |
|------|-----|----------------|
| Insufficient balance | There is no requirement, so there is no expected result to test against. | A rule from the analyst: where the balance check comes relative to the other checks and the SMS code. |
| REQ-2: reset at midnight Almaty time | The server clock cannot be changed in the test environment. | Clock control in the test environment, then tests at 23:59:59 and 00:00:00. |

---

## 4. SMS code release checklist

- [ ] 100,000 KZT executes without a code; 100,001 KZT asks for a code
- [ ] The code has 6 digits
- [ ] Nothing is debited while the code screen is open
- [ ] Correct code on the first attempt executes the transfer
- [ ] Correct code on the third attempt (wrong, wrong, correct) executes the transfer
- [ ] Third wrong code cancels the transfer
- [ ] After three wrong codes, a correct code is not accepted
- [ ] Code expires after 120 seconds and the transfer is cancelled
- [ ] Correct code entered after expiry is rejected
- [ ] Confirm tapped twice debits only once
- [ ] Cancelled or expired transfer does not change balance or daily total
- [ ] Daily total increases by the amount after confirmation

---

## 5. Defect reports

*Mock defect reports, based on gaps and risks found in the week 3 work.*

### DEF-01

**Transfers: balance debited twice when Confirm is tapped twice on the SMS code screen**

| Field | Value |
|-------|-------|
| ID | DEF-01 |
| Environment | Android 14, Samsung Galaxy A54, app 4.11.0 build 2291, test environment 2, 2026-10-05 19:40 Almaty |
| Preconditions | Logged in as test customer TC-001: balance 1,000,000 KZT, daily total 0 |
| Steps to reproduce | 1. Open Transfers > To another client > card ending 5512<br>2. Enter 150000 and tap Continue<br>3. Enter the correct code from the test SMS inbox<br>4. Double-tap Confirm<br>5. Open the Accounts screen |
| Expected result | One transfer of 150,000 KZT. Balance 850,000. Daily total 150,000. (REQ-3; state table: CONF + correct code must be rejected) |
| Actual result | Two success screens appear one after another. Balance 700,000. The statement shows two transfers of 150,000 KZT to card ending 5512 with the same timestamp. Daily total 300,000. |
| Reproducibility | 4 of 5 attempts (one attempt: the second tap was too slow and only one transfer was made) |
| Severity | Critical: customer loses money |
| Priority | High (proposed; to be confirmed by the product owner) |
| Evidence | Screen recording of steps 3–5; statement screenshot; request IDs of both confirm requests |
| Traces to | REQ-3, TC-12, risk R1 |

### DEF-02

**Transfers: daily limit resets at 05:00 Almaty time instead of midnight**

| Field | Value |
|-------|-------|
| ID | DEF-02 |
| Environment | iOS 18.2, iPhone 13, app 4.11.0 build 2291, test environment 2 with server clock moved to the test times below |
| Preconditions | Logged in as test customer TC-003: balance 1,000,000 KZT, daily total 900,000 made on 2026-10-05. Server clock set to 2026-10-06 00:05 Almaty time (2026-10-05 19:05 UTC). |
| Steps to reproduce | 1. Open Transfers > Limits, read "Used today"<br>2. Open Transfers > To another client > card ending 5512<br>3. Enter 200000 and tap Continue |
| Expected result | After midnight Almaty time the daily total is 0, so the 200,000 KZT transfer proceeds to the SMS code screen. (REQ-2: reset at midnight Almaty time) |
| Actual result | "Used today" still shows 900,000. Error "daily limit exceeded". Repeating the steps with the server clock at 04:59 Almaty gives the same error; at 05:01 Almaty the limit shows 0 and the transfer proceeds. |
| Reproducibility | 3 of 3 attempts |
| Severity | High: customers cannot transfer for 5 hours after midnight; also allows extra spending between 00:00 and 05:00 counted against the previous day |
| Priority | Medium (proposed) |
| Evidence | Screenshots of the Limits screen at 00:05, 04:59 and 05:01; server clock settings |
| Traces to | REQ-2 |

### DEF-03

**Transfers: customer with insufficient balance is asked for the SMS code, then gets a generic error (requirement gap)**

| Field | Value |
|-------|-------|
| ID | DEF-03 |
| Environment | Android 14, Samsung Galaxy A54, app 4.11.0 build 2291, test environment 2 |
| Preconditions | Logged in as a test customer with balance 200,000 KZT, daily total 0 |
| Steps to reproduce | 1. Open Transfers > To another client > card ending 5512<br>2. Enter 300000 and tap Continue<br>3. Enter the correct code from the test SMS inbox and tap Confirm |
| Expected result | Not defined. The brief has no rule for insufficient balance. Reasonable expectation: an "insufficient funds" message shown at step 2, before an SMS code is sent. |
| Actual result | Step 2: SMS code screen shown and an SMS sent. Step 3: message "Something went wrong. Try again later". Balance 200,000. No transfer in the statement. |
| Reproducibility | 5 of 5 attempts |
| Severity | Medium: no money lost, but an SMS is wasted and the message does not tell the customer what went wrong |
| Priority | Medium (proposed) |
| Evidence | Screen recording of steps 2–3; screenshot of the test SMS inbox |
| Traces to | No requirement: gap in the brief |

---

## 6. AI appendix (Level 1)

**Tool:** Claude (Anthropic). The AI turned my week 3 test design into this documentation. The test design itself (partitions, boundary values, decision table, state table) is my own work from week 3.

### Prompts

1. **Study the lectures:** "Summarise these lecture slides: key concepts, techniques and exam terms (ISTQB CTFL v4.0). Do not add facts that are not in the slides."
2. **Documentation pack:** "Using only my brief and my test design, write a one-page test plan, 12 test cases, a traceability matrix and a release checklist (max 12 items). Do not invent behaviour; mark every assumption."
3. **Defect report:** "Turn my test notes into a defect report with the standard fields. If a field is not in the notes, write MISSING. One action per step. Mark severity and priority as proposals."
4. **Requirement gaps:** "List every ambiguity and missing rule in this brief, the risk of each, and a question for the analyst."
5. **Self-review:** "Act as a lecturer who will follow one test case and one defect report exactly as written. List every line a colleague could not execute without asking me."

### Raw output

The first AI version was a 13-page document in Arial with coloured headings and shaded tables. It contained: a contents table; a test plan with scope, approach, environment and data, entry and exit criteria, a risk table with likelihood, and project risks; 12 test cases with technique, notes and risk rows; a traceability matrix with a supporting-tests column and a state-model coverage table; a release checklist table with iOS/Android columns; three mock defect reports with status and request fields; and an AI appendix with the prompts in Russian.

### What I changed and why

| # | What the AI wrote | What I changed | Why |
|---|-------------------|----------------|-----|
| 1 | Arial font, blue headings, shaded table headers | Times New Roman only, black text, plain black table borders | Formal academic formatting; colour adds nothing to the content |
| 2 | Test plan with extra sections: environment and data, project risks, version header, likelihood column in the risk table | Kept only scope, approach, entry and exit criteria, top three risks | The assignment asks for exactly these parts on one page |
| 3 | Test cases with extra rows: technique, notes, risk, week 3 IDs | Kept ID, traces to, preconditions, steps, expected result, postconditions | Standard test case fields; the extra rows made each case harder to follow |
| 4 | Traceability matrix with a supporting-tests column and a separate state-model coverage table | One matrix (requirement, test cases, technique, status) plus a short "not covered" table | Same format as the lecture; the gap (insufficient balance) is easier to see |
| 5 | Release checklist as a table with iOS/Android columns, trace links and two items not based on the brief (30 s SMS delivery, KZ/RU/EN texts) | Plain list of 12 checks, all based on the brief | A checklist says what to verify; items not in the requirements were invented |
| 6 | Defect reports with status, request and analyst-question fields | Kept the standard defect report fields only | Matches the ISTQB field list from the lecture |
| 7 | Prompts listed in Russian | Rewrote them as short English prompts | The whole document is in English |
