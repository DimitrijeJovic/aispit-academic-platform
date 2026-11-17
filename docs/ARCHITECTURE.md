# AIspit — System Architecture & Engineering Design

> **Verification note.** This document was written from the private source code, which is not published. Statements that go beyond the bachelor's thesis (for example exact limits, file names, function names, and file types) cannot be checked from this repository and should be read as unverified.

This document details the software architecture, design patterns, module organization, request life cycle, and infrastructural considerations of **AIspit** (*Sistem za elektronsku predaju radova i organizaciju onlajn ispita*).

---

## 1. Architectural Philosophy & Constraints

AIspit was engineered under strict academic and operational constraints established at the **Academy of Applied Technical and Preschool Studies Niš** (*Akademija tehničko-vaspitačkih strukovnih studija*):

1. **Laboratory Self-Containment (Zero External CDN Dependency):** The platform is designed to operate seamlessly in isolated computer examination laboratories without public internet access. All assets—including stylesheets, vendor icons (`Bootstrap Icons`), and client libraries—are bundled locally.
2. **Server-Authoritative Business Logic:** Client-side user agents are strictly treated as untrusted presentation and notification layers. All decisions regarding authentication, authorization, time boundaries, grading, file authenticity, and exam termination are executed exclusively on the server.
3. **Auditability & Historic Immutability:** Academic evaluation requires tamper-evident verification. Once an exam is initiated or an assignment is submitted, the system captures an immutable snapshot of the evaluation context. Changes made by instructors to the master question bank or syllabus never mutate historically completed evaluations.
4. **Predictable Data Access Without Leaky Abstractions:** The system deliberately avoids Object-Relational Mapping (ORM) frameworks to guarantee transparent, fine-grained control over SQL queries, transaction boundaries, index utilization, and concurrency locks.

---

## 2. High-Level System Architecture

AIspit adopts a classical **3-Tier Layered Architecture** following the **Model-View-Controller (MVC)** architectural pattern within a **Multi-Page Application (MPA)** paradigm:

```
+-------------------------------------------------------------------------+
|                              CLIENT TIER                                |
|   Modern Web Browser (Desktop / Lab Workstation)                        |
|   HTML5 Semantic DOM | Hand-crafted CSS3 (Variables) | Vanilla ES6 JS   |
+-------------------------------------------------------------------------+
                                    |
                            HTTP/S Requests
                            (Cookies / Form / Fetch)
                                    |
                                    v
+-------------------------------------------------------------------------+
|                             APPLICATION TIER                            |
|  Flask Core Framework (Application Factory: create_app)                 |
|                                                                         |
|  +-------------------------------------------------------------------+  |
|  | CONTROLLERS: Flask Blueprints (106 Endpoints)                     |  |
|  | - auth_bp (4)    | admin_bp (39)                                  |  |
|  | - profesor_bp (43) | student_bp (20)                              |  |
|  +-------------------------------------------------------------------+  |
|  | ACCESS CONTROL GUARDS                                             |  |
|  | - @login_required                                                 |  |
|  | - @role_required('admin' | 'profesor' | 'asistent' | 'student')   |  |
|  | - Granular In-Route Resource Ownership Verification               |  |
|  +-------------------------------------------------------------------+  |
|  | DOMAIN SERVICES & TIME RECONCILIATION                             |  |
|  | - beograd_sada() Single Source of Truth (Wall-Clock Europe/Belgrade)  |
|  | - HMAC-SHA256 Submission Integrity Engine                        |  |
|  | - Magic-byte MIME Validator & Two-Tier Quota Enforcer            |  |
|  +-------------------------------------------------------------------+  |
|  | VIEW PRESENTATION LAYER                                           |  |
|  | - Jinja2 Template Hierarchy (62 Templates, Base Inheritance)     |  |
|  | - Reusable Macro Components (Status Badges, Tables, Pagination)   |  |
|  +-------------------------------------------------------------------+  |
|  | DATA ACCESS MODEL (Pure SQL, Zero ORM)                            |  |
|  | - db.py Connection Lifecycle (1 Connection per HTTP Request in g) |  |
|  | - Explicit Transactions (autocommit=False)                        |  |
|  | - Parameterized Prepared Statements (%s)                             |  |
|  +-------------------------------------------------------------------+  |
+-------------------------------------------------------------------------+
                                    |
                          TCP/IP Socket (3306)
                          mysql-connector-python
                                    |
                                    v
+-------------------------------------------------------------------------+
|                               DATA TIER                                 |
|   MySQL 8.0 Community Server (InnoDB Engine, utf8mb4)                   |
|   23 Normalized Tables | 39 Foreign Keys | 16 Unique Constraints        |
|   46 Secondary Performance Indexes | Soft-Delete Auditing (DeletedAt)   |
+-------------------------------------------------------------------------+
```

---

## 3. Directory Layout & Package Structure

The project codebase follows strict modular separation:

```
projekat/
|-- run.py                     # Entry point for development and WSGI invocation
|-- config.py                  # Environment-specific configuration classes
|-- requirements.txt           # Minimized dependencies (flask, mysql-connector, etc.)
|-- schema.sql                 # Canonical MySQL 8 database schema definition
|-- app/
|   |-- __init__.py            # Application Factory: create_app()
|   |-- db.py                  # Database connection lifecycle & timezone synchronization
|   |-- models/                # Domain models encapsulating pure SQL queries
|   |   |-- aktivacionikod.py  # Activation code lifecycle & atomic consumption
|   |   |-- korisnik.py        # User authentication & role management
|   |   |-- pitanje.py         # Question bank & answer option operations
|   |   |-- predaja.py         # Assignment submissions & HMAC verification
|   |   |-- predmet.py         # Academic courses & curriculum mapping
|   |   |-- prijava.py         # Examination sessions & assignment deadlines
|   |   |-- smer.py            # Study programs & degree tracks
|   |   |-- student.py         # Student records & academic enrollments
|   |   |-- studenttest.py     # Exam session execution, snapshots & grading
|   |   `-- test.py            # Section-based test specifications
|   |-- routes/                # Controller layer: Flask Blueprints
|   |   |-- auth.py            # Login, logout, session cookie lifecycle
|   |   |-- admin.py           # Institutional administration & batch user import
|   |   |-- profesor.py        # Question bank, exam creation, submissions, grading
|   |   |-- student.py         # Student dashboard, active exams, assignment uploads
|   |   `-- decorators.py      # Authentication & role authorization decorators
|   |-- templates/             # Jinja2 template hierarchy (62 templates)
|   |   |-- base.html          # Global layout (sidebar, topbar, theme switcher)
|   |   |-- base_polaganje.html# Distraction-free layout for exam taking (no nav)
|   |   |-- macros/            # UI components (status_badge, tables, action links)
|   |   |-- admin/             # Admin management views
|   |   |-- auth/              # Authentication split-screen views
|   |   |-- profesor/          # Professor dashboards, builders, evaluation views
|   |   `-- student/           # Student submission and exam interfaces
|   `-- static/                # Static public assets
|       |-- css/style.css      # Consolidated hand-crafted stylesheet (~1,200 lines)
|       |-- img/               # Brand logos, themes, vector emblems
|       `-- vendor/            # Offline vendor assets (Bootstrap Icons)
|-- uploads/                   # Secure storage for student uploads (OUTSIDE static/)
|-- tests/                     # Automated test suites (547 in-process, 27 browser)
`-- docs/                      # Architectural and technical documentation
```

---

## 4. Architectural Patterns & Core Decisions

### 4.1 Application Factory Pattern (`create_app()`)
Rather than relying on a global Flask application instance, the core application is instantiated via a factory function:
- **Environment Isolation:** Allows dynamic injection of configuration profiles (`ProductionConfig`, `TestingConfig`).
- **Test Automation:** Enables the automated test harness to spin up isolated application contexts within the same process without requiring network binding.
- **Blueprint & Filter Registration:** All routing blueprints, Jinja2 environment filters (e.g., custom Serbian date-time formatters), and error handlers (`403`, `404`, `500`) are bound programmatically.

### 4.2 Multi-Page Application (MPA) vs. Single-Page Application (SPA)
The decision to utilize an MPA architecture with server-rendered HTML (Jinja2) was deliberate:
- **Security Boundaries:** Business rules governing academic evaluation cannot be inspected, bypassed, or tampered with by client-side JavaScript execution.
- **No Complex Client Toolchain:** Zero client-side packaging step (no Webpack, Vite, Node runtime, or npm vulnerabilities in production).
- **Tabular & Form-Centric Domain:** The vast majority of views are structured academic forms, administrative listings, and tabular rosters—scenarios where SSR provides optimal performance and accessibility.
- **Progressive Enhancement:** JavaScript is utilized selectively as an enhancement layer (e.g., auto-saving answers via `fetch()`, debouncing keystrokes, clipboard utilities, fullscreen toggles, and dynamic modal notifications) rather than as a prerequisite for core form processing.

### 4.3 Post/Redirect/Get (PRG) Pattern
All mutating actions (creating an exam, submitting answers, assigning grades, uploading archives) strictly follow the Post/Redirect/Get pattern:
1. Client submits payload via `POST`.
2. Controller validates input, performs database transaction, and registers a flash notification.
3. Controller responds with an HTTP `302/303 Redirect` to a `GET` endpoint.
4. Client renders view via idempotent `GET`.
*Prevents accidental duplicate submissions or transaction retries on browser reload.*

### 4.4 The 3-Tier Access Control Pattern
Every incoming HTTP request must pass through three distinct authorization checkpoints:
1. **Tier 1 — Authentication Guard (`@login_required`):** Ensures that the incoming session cookie is cryptographically signed and contains a valid user identifier (`korisnik_id`). Unauthenticated visitors are redirected to `/login`.
2. **Tier 2 — Role Authorization Guard (`@role_required(*roles)`):** Verifies that the authenticated session holds the requisite role (`'admin'`, `'profesor'`, `'asistent'`, `'student'`). Unauthorized attempts are aborted with HTTP 403 / warning redirect.
3. **Tier 3 — Resource Ownership Verification (In-Route Guard):** Role verification alone is insufficient in multi-tenant environments. A professor must not access another professor's exam session or question bank simply by altering an integer in the URL query string (`/profesor/prijava/42`). In-route validation functions (e.g., `_check_prijava_ownership(prijava_id, profesor_id)`) verify that the requested database entity is owned by the authenticated caller.

```
       Incoming HTTP Request
                 |
                 v
   +-----------------------------+
   |  Is user authenticated?     | ---- NO ---> Redirect to /login
   +-----------------------------+
                 | YES
                 v
   +-----------------------------+
   |  Does user possess role?    | ---- NO ---> Flash 403 "Pristup odbijen"
   +-----------------------------+
                 | YES
                 v
   +-----------------------------+
   |  Does resource belong       |
   |  to authenticated user?     | ---- NO ---> Flash 403 "Pristup odbijen"
   +-----------------------------+
                 | YES
                 v
       Execute Business Logic
```

---

## 5. Subsystem Modular Breakdown

### 5.1 Authentication Subsystem (`app/routes/auth.py`, `app/models/korisnik.py`)
- **Closed Registration Policy:** Self-registration is strictly disabled. Student and faculty accounts are provisioned exclusively by administrators or imported via institutional CSV rosters.
- **Cryptographic Password Hashing:** User passwords are never persisted in plaintext. Passwords are processed using Werkzeug's password hashing with a random salt per password (the algorithm is not named in the thesis).
- **Tamper-Proof Client Session:** Session tokens are stored in cryptographically signed client cookies using the application's private `SECRET_KEY`. Session payload contains strictly minimal identity claims (`korisnik_id`, `uloga`); full entity attributes are fetched fresh from MySQL on every request to reflect immediate permission revocations or deactivations.
- **Uniform Error Messaging:** Login validation emits generic error feedback (`"Неисправни приступни подаци"`) regardless of whether the email or password failed, preventing user enumeration attacks.

### 5.2 Examination Engine (`app/routes/profesor.py`, `app/models/test.py`, `app/models/studenttest.py`)
- **Question Bank Modularization:** Questions belong to specific subject areas (`Oblast`) and support two response paradigms:
  1. *Closed-ended (Multiple Choice / Single Choice):* Automatically graded by the server.
  2. *Open-ended (Free Text Input):* Queued for manual grading by the professor.
- **Dynamic 2-Pass Section Composition:** Tests are specified via composite sections (`TestStavka`). Each section specifies a target subject area, number of questions, and selection mode (`ručno` vs `nasumično`). Test generation employs a two-pass algorithm:
  - *Pass 1:* Fulfills explicitly assigned questions.
  - *Pass 2:* Fills remaining slots randomly from the unselected candidate pool of that area, guaranteeing zero duplicate questions across sections.
- **Candidate Snapshot Freezing:** Upon candidate initialization (`StudentTest`), questions, randomized answer choices, and designated point values are duplicated into candidate-specific tables (`StudentTestPitanje`, `StudentTestOdgovor`). Master question bank changes never affect an active or completed attempt.
- **Client DOM Sanitization:** Correct answer flags are scrubbed before rendering HTML; candidate DOM contains strictly question text and choice IDs.
- **Asynchronous Auto-Saving:** JavaScript monitors candidate input. Choice changes trigger instant background `fetch()` requests; free text fields utilize a 1,500ms debounce after typing ceases, persisting partial state against power or network interruptions.

### 5.3 Assignment Submission Subsystem (`app/routes/student.py`, `app/models/predaja.py`, `app/models/aktivacionikod.py`)
- **Session Types:** A `Prijava` session can be configured as pure exam (`Test`), pure assignment submission (`Rad`), or unified composite (`Test + Rad`).
- **Activation Code Access Gates:** Submissions can be gated by professor-generated single-use or shared activation codes. Code consumption is executed as an atomic SQL transaction:
  ```sql
  UPDATE AktivacioniKod 
  SET Iskoriscen = 1 
  WHERE ID = %s AND Iskoriscen = 0 AND Aktivan = 1 AND VaziDo > %s;
  ```
- **Brute-Force Rate Limiting:** Enforces a maximum of 5 code validation attempts per minute per student, logging client IP, timestamp, and failure cause to `PokusajPredaje`.
- **MIME Magic-Byte Verification:** Uploaded streams are inspected for real binary signatures (e.g., `%PDF`, `PK\x03\x04`), rejecting executable files masquerading as archives or documents.
- **Keyed Submission Integrity Verification (HMAC):** Every accepted submission gets a message authentication code computed with a separate signing key (Python `hmac`). The exact fields and hash function are not documented in the thesis.
  Any direct modification of database records or file payload on the disk breaks verification.

---

## 6. Request Lifecycle

The standard request lifecycle across AIspit follows a deterministic sequence:

1. **Ingress:** The request reaches the Flask application (the production server configuration is not documented in the thesis).
2. **Context Creation:** Flask's application factory instantiates application and request contexts.
3. **Database Attachment (`get_db()`):** A dedicated MySQL connection is retrieved or opened, stored in `flask.g.db`. The session timezone is initialized to `SET time_zone = '+01:00'` (or `+02:00` CEST).
4. **Middleware & Routing:** Flask matches URL patterns against registered Blueprints.
5. **Guard Evaluation:** `@login_required` verifies session cookie signature; `@role_required` asserts role claims; in-route ownership assertions check resource tenant bounds.
6. **Domain Execution:** Controller invokes plain Python model methods. Models execute parameterized SQL queries over `g.db`.
7. **Template Rendering:** Jinja2 renders HTML with contextual parameters, resolving macro components and active UI theme.
8. **Teardown & Cleanup (`close_db()`):** On request completion, `app.teardown_appcontext` closes cursors and returns/closes the MySQL connection. If an unhandled exception occurred, rollback is triggered.
9. **Egress:** The HTTP response is returned to the client.

---

## 7. Architectural Evolution & Technical Lessons

The design of AIspit was refined through rigorous iterative problem discovery during development:

| Scenario / Defect | Initial Naive Implementation | Architectural Resolution |
| :--- | :--- | :--- |
| **Admin vs. Professor Session Editing** | Separate admin edit form with incomplete parameters silently unlinked attached tests and reset time constraints. | Consolidated editing logic into a single authoritative domain model used by both administrative and faculty routes. |
| **Section Question Pool Starvation** | Section generator sampled from whole topic and dropped duplicates, silently delivering fewer questions than requested. | Refactored into a deterministic two-pass algorithm: explicit assignments first, followed by quota fulfillment from remaining questions with explicit capacity assertions. |
| **Exam Violation Modal Trapping** | Violation warning modal was placed inside a container that inherited CSS `filter: blur()` and `pointer-events: none`, permanently locking the student out. | Extracted modal outside the blurred viewport container, debounced browser event triggers, and instituted real-browser E2E testing. |
| **Uploaded File Exposure** | Early upload directory was situated in `app/static/uploads/`, allowing unauthenticated direct downloads if URLs leaked. | Relocated upload storage completely outside static root (`/uploads/`) and enforced an authenticated streaming controller with strict ownership checks. |
