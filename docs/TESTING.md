# AIspit — Quality Assurance, Testing Strategy & Verification

This document details the software quality assurance methodology, automated test suites, fault injection philosophy, verified test counts, and testing limitations of **AIspit** (*Sistem za elektronsku predaju radova i organizaciju onlajn ispita*).

---

## 1. QA Philosophy: The "Negative Proof of Test Validity" Principle

In safety-critical and academic evaluation systems, a test suite that produces 100% green checkmarks can create a dangerous illusion of security. An assertion that solely evaluates an HTTP status code (`200 OK` or `302 Found`) frequently passes even when underlying security controls are disabled.

To counteract this, the engineering methodology for AIspit adopted the **Negative Proof of Test Validity** (Mutation Testing / Fault Injection) principle:

> **Core QA Axiom:** *"No test is deemed valid until it has been proven capable of failing when the targeted business rule or security guard is intentionally disabled or corrupted."*

### 1.1 Methodology
For every automated check added to the test suite:
1. The developer intentionally introduces a fault into the domain code (e.g., removing the MIME magic-byte check, disabling ownership validation, or allowing unlimited submission attempts).
2. The test suite is executed against the faulted system.
3. If the test continues to pass, the test is rejected as a **false positive** (non-measuring assertion).
4. Only when the test fails specifically on the induced defect is the test accepted into the permanent regression suite.

---

## 2. Documented Automated Test Counts & Classification

The thesis documents a verified total of **574 automated checks**, divided into two complementary execution environments:

> **Testing Transparency Notice:**
> The 574 automated tests represent the verified historical test suite documented and evaluated in the final academic thesis. These tests were **not independently re-executed** during this portfolio documentation preparation phase.

```
+-------------------------------------------------------------------------+
|                  TOTAL AUTOMATED CHECKS: 574                            |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+  |
|  | 547 IN-PROCESS FLASK CLIENT TESTS (26 Functional Suites)          |  |
|  | - Direct WSGI invocation without network latency                  |  |
|  | - Evaluates database state, session cookies, and rendered DOM     |  |
|  | - Execution duration: ~3 to 5 seconds                             |  |
|  +-------------------------------------------------------------------+  |
|                                                                         |
|  +-------------------------------------------------------------------+  |
|  | 27 REAL-BROWSER END-TO-END TESTS (Headless Automation)            |  |
|  | - Headless browser running against a live test server             |  |
|  | - Evaluates JavaScript execution, CSS rendering, focus & modals   |  |
|  | - Fullscreen API events, countdown clock synchronization          |  |
|  +-------------------------------------------------------------------+  |
+-------------------------------------------------------------------------+
```

---

## 3. In-Process Test Suites Breakdown (547 Tests across 26 Suites)

In-process tests utilize Flask's built-in test client (`app.test_client()`). They execute within the same process memory space against a dedicated MySQL test database, allowing complete transaction inspection without network overhead:

| # | Test Suite / Category | Scope & Invariants Verified |
| :---: | :--- | :--- |
| **1** | **Pristup i prijavljivanje** (Access & Auth) | Anonymous visitor redirection, invalid role access rejection, session cookie signature validation, uniform error messages. |
| **2** | **Dostupnost ruta** (Route Availability) | Asserts that every registered blueprint route returns expected HTTP responses for authorized user roles. |
| **3** | **Prikaz i konvencija** (UI Conventions) | Validates across all rendered tables: column count alignment, status badge in column 1, actions in last column, date-status coherence. |
| **4** | **Izbor pitanja** (Question Selector) | Two-pass question selection: quota fulfillment, zero duplicate questions across sections, clear error emission on topic starvation. |
| **5** | **Uvoz korisnika** (Batch User Import) | 24 tests: Multi-encoding CSV parsing (UTF-8, Windows-1250 Cyrillic), preview dry-run, in-memory password generation, hash-only persistence. |
| **6** | **Polaganje** (Exam Execution) | Single-attempt enforcement, dynamic question order shuffling, answer choice shuffling, DOM scrubbing of correct answers. |
| **7** | **Vreme** (Time Boundaries) | Server-authoritative countdown calculation, deadline expiration materialization, closing of abandoned attempts. |
| **8** | **Ocenjivanje** (Automated Grading) | Exact matching for multi-choice items, zero points for partial/unanswered items, open-ended question queuing. |
| **9** | **Komisijski upis** (Manual Evaluation) | Point range enforcement, atomic batch score commit, distinguishing explicit zero from unreviewed (NULL) responses. |
| **10** | **Praćenje napuštanja** (Violation Tracking) | Server-side violation counter incrementation, warning trigger, automatic test termination at threshold 3. |
| **11** | **Zaključavanje** (Session Locking) | Immutability of test parameters and session rules once an exam is active or has recorded attempts. |
| **12** | **Poništavanje pokušaja** (Attempt Reset) | Audited cancellation by faculty/admin, audit trail preservation, restoration of single attempt for student. |
| **13** | **Otpremanje radova** (Assignment Upload) | 25 tests: Magic-byte MIME validation, size caps, UUID filename sanitization, HMAC integrity verification, atomic code redemption. |
| **14-26**| **Supporting Domain & Model Suites** | Course enrollment constraints, soft-delete filtering (`DeletedAt IS NULL`), transaction rollbacks, timezone alignment. |

---

## 4. Real-Browser End-to-End Tests (27 Tests)

The 27 browser-based tests were instituted following the discovery that in-process WSGI tests cannot detect client-side behavioral failures. They validate:

1. **Anti-Cheating Violation Modal Interaction:** Verifies that when a student triggers a `visibilitychange` or `blur` event, the violation dialog appears, is interactive, is not obscured by CSS blur filters, and the "Настави" (Continue) button restores focus.
2. **Debounced Event Handling:** Ensures that pressing F5 or reloading the browser does not emit duplicate violation events that immediately abort the test.
3. **Autosave Interactivity:** Validates that selecting a radio button dispatches a background `fetch()` call and displays the green *"Сачувано"* badge without page flicker.
4. **Debounced Textarea Saving:** Verifies that typing free-text answers schedules a background save 1.5 seconds after keystrokes cease, and saves immediately upon `blur`.
5. **Theme Switching:** Validates that toggling Light/Dark theme instantly mutates the `data-theme` attribute on `<html>` and updates CSS custom properties without visual flash.
6. **Distraction-Free Mode:** Verifies that navigation bars, sidebars, and outbound hyperlinks are completely absent from the exam viewport.

---

## 5. False Positives Discovered via Mutation Testing

The "Negative Proof" methodology successfully uncovered five critical testing defects that previously yielded false passes:

### Case 1: Silent Section Deletion Masked as Success
- **Defect:** An administrative test was sending incorrect HTML form field names. The controller parsed empty inputs and silently deleted all test sections.
- **Why the Test Passed:** The test only asserted HTTP `302 Redirect`. Because the controller redirected to the success dashboard, the test reported green while corrupting test data.
- **Resolution:** Re-engineered the assertion to query the database before and after the request, verifying that existing `TestStavka` records were preserved.

### Case 2: Model Testing vs. Template Decision Divergence
- **Defect:** A test verified that an assignment session permitted uploads by asserting the model function `prijava.ima_fajl()`. However, the view template utilized an independent, outdated condition to render the upload widget.
- **Why the Test Passed:** The model test was green, yet students saw no upload button on the live screen.
- **Resolution:** Replaced raw model unit testing with full HTML response parsing asserting the presence of the upload input element.

### Case 3: CSS Computed Blur Property Trap
- **Defect:** A test asserted that the violation warning modal had no CSS `filter: blur()`. 
- **Why the Test Passed:** The modal element itself had no blur rule, but its parent container did. In real browsers, child elements inherit the visual blur of their parent even though `getComputedStyle(child).filter` returns `'none'`.
- **Resolution:** Moved the test to the real-browser suite, asserting element visibility and clickability via pointer hit-testing.

### Case 4: Identical Redirect Codes on Authorization Failure
- **Defect:** A network-restriction test checked whether access from an unauthorized IP address was rejected.
- **Why the Test Passed:** Both the authorized path and the rejected path issued an HTTP `302 Redirect` (one to the dashboard, one to the login error page). The test simply asserted `response.status_code == 302`.
- **Resolution:** Updated assertions to follow the redirect and inspect the resulting flash message and destination URL.

### Case 5: Validator Precedence Masking Feature Locking
- **Defect:** A test verifying that an active exam's time limit cannot be modified submitted a payload with invalid date formats.
- **Why the Test Passed:** The request was rejected with an error, which the test interpreted as the locking mechanism working. In reality, the locking guard was never reached because the date validator rejected the request first.
- **Resolution:** Sanitized the test fixture payload to ensure valid syntax so that only the locking guard could trigger rejection.

---

## 6. Upload Testing Nuances & Discoveries

Testing file uploads required 25 rigorous assertions. Four specific test design traps were identified and resolved:

1. **Pre-existing Submission Collision:** A test designed to verify that executable files are rejected failed because the mock student already had an existing submission from a prior test. The request was aborted by the single-attempt constraint before reaching the MIME validator. *Resolution: Each upload test now spins up a dedicated, isolated session fixture.*
2. **Trivial Unique Name Verification:** A test asserting that files uploaded by different students receive distinct server filenames initially uploaded files with different original names (e.g., `a.pdf` and `b.pdf`). The test passed vacuously. *Resolution: Updated the test to upload identical filenames (`test.pdf`), proving that UUID generation enforces disk isolation.*
3. **Resubmission Record Overwriting:** A resubmission test counted the total rows in `predaja`. Because `(Student_ID, Prijava_ID)` has a unique constraint, an approved resubmission updates the existing row rather than inserting a new one. Counting rows failed to detect whether the file had actually been replaced. *Resolution: Asserted changes in `SacuvaniNazivFajla` and `SubmissionKey`.*
4. **Static Directory Isolation False Pass:** A test verifying that uploaded files cannot be downloaded via `/static/uploads/` picked a submission whose file did not exist on the mock disk. The HTTP `404 Not Found` was mistaken for directory isolation. *Resolution: Ensured the file exists on disk, proving that HTTP 404 occurs because the static web server route is disconnected.*

---

## 7. Known Testing Limitations & Untested Scenarios

To maintain complete transparency, the following limitations of the test harness are documented:

1. **Activation Code Rate Limiting Exclusion:** The security control enforcing a maximum of 5 code entry attempts per minute per student is **excluded** from the automated test harness. Because the test runner executes dozens of requests within milliseconds, the rate limiter would trip and abort unrelated test suites. In testing fixtures, attempt counters in `pokusajpredaje` are reset prior to invocation.
2. **High-Concurrency Load Testing:** The test suite does not simulate concurrent load scenarios (e.g., 500 students simultaneously submitting answers at the final second of an exam). Concurrency safety is enforced structurally via InnoDB row-level locking and atomic SQL queries, but synthetic multi-threaded stress benchmarks were not executed.
3. **No 100% Code Coverage Claim:** Line coverage is deliberately not claimed as 100%. The testing strategy prioritizes high-risk security boundaries, access controls, data integrity constraints, and fault injection over trivial template boilerplate coverage.
