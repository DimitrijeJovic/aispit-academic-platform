# AIspit — AI-Assisted Software Engineering & Empirical Analysis

This document details the methodology, tool ecosystem, empirical version-control statistics, iterative failure cases, and verification framework governing the use of **Generative Artificial Intelligence (AI)** in the engineering of **AIspit** (*Sistem za elektronsku predaju radova i organizaciju onlajn ispita*).

---

## 1. Foundational Demarcation: AI in Development vs. AI in Runtime

A fundamental principle of this academic project is complete honesty regarding the role of Artificial Intelligence:

> **CRITICAL ARCHITECTURAL DISTINCTION:**
> - **AI IN DEVELOPMENT (YES):** Generative AI tools were utilized extensively as interactive design sounding boards, implementation copilots, code generators, and test-authoring assistants.
> - **AI IN RUNTIME (NO):** The deployed application contains **zero Artificial Intelligence models**. It does not perform automated essay grading via Large Language Models, does not make non-deterministic evaluation decisions, and relies strictly on deterministic relational algebra, cryptographic hashes, and server-side business rules.

Presenting AI as a runtime feature of AIspit would be technically inaccurate; its presence is strictly confined to the software engineering development process.

---

## 2. Multi-Tool AI Ecosystem & Assigned Responsibilities

Four distinct AI systems were deployed during development, each matched to specific engineering disciplines:

| Tool | Deployment Paradigm | Engineering Scope & Assigned Responsibilities |
| :--- | :--- | :--- |
| **Claude (Conversational)** | Web Interface / Dialogic Consultation | **System Architecture & Trade-Off Analysis:** Conceptual deliberation prior to code authoring. Analyzed trade-offs of omitting ORM layers, adopting Belgrade wall-clock single source of time, designing the immutable snapshot denormalization pattern, and designing the 3-tier access control model. |
| **Claude Code** | Terminal / In-Environment Agent | **Implementation, Refactoring & Test Automation:** Direct access to repository files and system console. Generated Flask blueprints, domain models, parameterized SQL queries, Jinja2 templates, and automated test suites. Executed test runners and iterated based on compiler and test failure outputs. |
| **Gemini** | Web Interface / Multimodal Studio | **Visual Identity & Logo Engineering:** Conceptualized the "AIspit" shield-and-book emblem across multiple iterations. Adapted Cyrillic typography and generated high-contrast monochrome variants specifically engineered for the application's dark theme. |
| **ChatGPT / GPT** | Web Interface / Vision Processing | **Asset Post-Processing & Palette Calibration:** Image background removal, alpha transparency mask extraction, resolution scaling, and color value alignment with the application's hand-crafted CSS custom properties. |

---

## 3. The 6-Step Code Verification Pipeline

AI tools frequently generate code that *appears* completely correct—it is neatly formatted, syntactically clean, and richly commented—yet conceals subtle logical bugs or security vulnerabilities.

To prevent flawed AI output from compromising the platform, every AI-suggested change had to pass a mandatory **6-Step Verification Pipeline**:

```
[ AI Code Proposal ]
        |
        v
+-------------------------------------------------------------------------+
| STEP 1: COMPREHENSIVE HUMAN UNDERSTANDING                               |
| The author must be able to explain every line, construct, and decision. |
| If it cannot be defended during thesis examination, it is rejected.    |
+-------------------------------------------------------------------------+
        |
        v
+-------------------------------------------------------------------------+
| STEP 2: PROJECT CONVENTION AUDIT                                        |
| Checks adherence to foundational project invariants:                    |
| - Soft deletion: Does every query include 'WHERE DeletedAt IS NULL'?    |
| - Time source: Does it call beograd_sada() instead of datetime.now()?   |
+-------------------------------------------------------------------------+
        |
        v
+-------------------------------------------------------------------------+
| STEP 3: ARCHITECTURAL BOUNDARY VERIFICATION                             |
| - Do models remain pure (no import flask.request)?                      |
| - Do templates remain strictly presentational (no business logic)?      |
+-------------------------------------------------------------------------+
        |
        v
+-------------------------------------------------------------------------+
| STEP 4: DEFENSIVE SECURITY INSPECTION                                   |
| - Are SQL queries parameterized (%s)?                              |
| - Is access guarded by authentication AND role AND ownership checks?    |
+-------------------------------------------------------------------------+
        |
        v
+-------------------------------------------------------------------------+
| STEP 5: FAULT-INJECTION TESTING ("Negative Proof")                      |
| Is the test suite proven capable of failing if the rule is corrupted?   |
+-------------------------------------------------------------------------+
        |
        v
+-------------------------------------------------------------------------+
| STEP 6: FULL REGRESSION RE-EXECUTION                                    |
| Full execution of 547 in-process and 27 browser checks on test DB.      |
| Zero testing on live production data.                                   |
+-------------------------------------------------------------------------+
        |
        v
[ Git Commit Accepted ]
```

---

## 4. Iterative Failure Cases & Engineering Discoveries

The limitations of generative AI were documented through real iterative failures encountered during development:

### Case 1: The Administrative Session Modification Bug
- **Task:** Enable system administrators to edit examination sessions.
- **AI Proposal:** The in-environment agent created a dedicated administrative edit form. However, this form only exposed basic metadata (title, dates) and omitted advanced fields (bound test, duration, obligation type). The controller called the update model passing 9 arguments, while the underlying function expected 14 arguments. The remaining 5 parameters were silently populated with default values.
- **The Failure:** When an administrator corrected a typo in a colloquium title and clicked Save, the bound test was unlinked, the activation code requirement vanished, and the obligation type was reset—without any warning.
- **Root Cause & Resolution:** The AI treated the immediate request by duplicating code rather than considering the holistic domain model. The duplicate administrative form was permanently removed, and administrators were routed to the unified editing workflow used by faculty.

### Case 2: The Section Question Quota Starvation Bug
- **Task:** Dynamically generate an exam from topic sections.
- **AI Proposal:** The agent wrote an algorithm that sampled questions from each topic independently and subsequently discarded duplicates.
- **The Failure:** Tests consistently generated fewer questions than requested (e.g., requesting 8 questions yielded 7; testing two sections over a 5-question pool yielded fluctuating counts of 5, 4, or 3). Two students taking the identical exam received differing question volumes.
- **Root Cause & Resolution:** The AI sampled without excluding previously selected IDs, dropped collisions without replacement, and asserted capacity against the whole topic rather than remaining unpicked items. The algorithm was refactored into a strict **Two-Pass Algorithm**: Pass 1 fulfills explicit assignments; Pass 2 draws random items exclusively from the remaining unselected pool with explicit capacity assertions.

### Case 3: The Trapped Violation Modal Bug
- **Task:** Implement anti-cheating viewport monitoring.
- **AI Proposal:** The agent constructed a modal dialog alerting students when leaving the exam screen.
- **The Failure:** In automated WSGI unit tests, the feature passed with flying colors. But in a live browser, the exam was rendered completely unrecoverable on the very first violation. The modal was nested inside the DOM element targeted by CSS `filter: blur(8px)` and `pointer-events: none`, causing the modal to inherit blur and click-disabling. The "Настави" (Continue) button could not be clicked or focused via keyboard. Furthermore, browser reload events fired multiple un-debounced blur events, immediately hitting the 3-violation threshold and aborting the exam.
- **Root Cause & Resolution:** In-process HTTP tests cannot evaluate computed visual styles or browser event timing. The modal was extracted outside the blurred container, browser events were debounced with a temporal window, and 27 real-browser E2E tests were instituted.

### Case 4: The Public Upload Directory Vulnerability
- **Task:** Implement student file upload functionality.
- **AI Proposal:** The agent configured student uploads to be stored inside `app/static/uploads/`.
- **The Failure:** Discovered months later during test suite expansion. The web server served the `static/` folder directly without entering the Flask application. Anyone possessing the URL could download confidential submissions without logging in.
- **Root Cause & Resolution:** The AI followed common quick-start tutorial patterns rather than enterprise security principles. The storage was relocated outside the static web root to `/uploads/`, and an authenticated streaming endpoint was engineered.

### The "Symptom Treatment" Pattern
Across the project, AI consistently displayed a tendency to **treat superficial symptoms rather than root causes**:
- When an input form suffered from excessive vertical spacing, the AI generated three consecutive commits over three weeks shaving off margins and padding, instead of identifying and removing an unnecessary wrapper element causing the overflow.
- When two different views showed conflicting submission counts, the AI attempted to patch one template's logic rather than unifying the calculation into a single authoritative model function.

---

## 5. Empirical Git Repository Analytics

The development of AIspit spanned **6 weeks** and accumulated **110 version-controlled Git commits**. An empirical analysis of commit history reveals the operational dynamics of AI-assisted software engineering:

### 5.1 Commit Categorization
```
+-------------------------------------------------------------------------+
| TOTAL COMMITS: 110                                                      |
+-------------------------------------------------------------------------+
| New Features:          42 commits (38.2%)                               |
| Refinements:           32 commits (29.1%)                               |
| Bug Fixes / Hotfixes:  19 commits (17.3%)                               |
| Documentation & Tests: 17 commits (15.5%)                               |
+-------------------------------------------------------------------------+
```
*Nearly 1 in every 6 commits (17.3%) was a direct bug fix addressing issues introduced in earlier iterations, underscoring that AI code generation is inherently non-linear and requires substantial corrective cycles.*

### 5.2 Development Velocity Distribution
The temporal distribution of commits demonstrates a distinctive pattern:
- **Concentrated Bursts:** Over **50% of all commits** were created in the **final 5 days** of active development.
- **Peak Velocity:** A single calendar day recorded **22 commits**, primarily comprising table convention harmonization across 45 templates (out of 62 total application templates) and test suite expansion.
- **Mid-Development Hiatus:** A multi-week interval elapsed between late July and early August with zero commits.

**Analytical Insight:** Generative AI dramatically accelerates the *typing and syntax generation phase*, but does not accelerate the *decision-making, requirements definition, or architectural contemplation phases*. Work accumulates until the human engineer establishes a clear design direction; once decided, execution proceeds at unprecedented velocity.

---

## 6. Conclusions on AI in Software Engineering

The empirical findings from the development of AIspit yield three primary conclusions:

1. **Effort Relocation, Not Effort Elimination:** Generative AI does not reduce total engineering effort; it relocates it. Time saved on boilerplates, syntax lookup, and routine CRUD authoring was redirected into rigorous test authoring, code review, fault injection, and defensive auditing.
2. **Review Overhead Exceeds Traditional Authoring:** Reviewing AI-generated code requires *greater* cognitive scrutiny than reading code written by oneself. Code written by hand reflects an existing mental model; code provided by AI arrives clean, confident, and plausible, demanding deliberate skepticism to detect subtle logical divergences.
3. **The Indispensability of the Human Engineer:** Generative AI proposed solutions, but the human engineer made all definitive architectural choices: rejecting unnecessary ORM layers, rejecting unnecessary Many-to-Many course complexities, eliminating duplicate administrative forms, and taking ultimate responsibility for platform security and academic fairness.
