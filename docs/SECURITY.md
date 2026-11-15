# AIspit — Security Architecture & Threat Mitigation

This document details the defensive security architecture, authentication mechanisms, authorization boundaries, cryptographic controls, and threat mitigations implemented in **AIspit** (*Sistem za elektronsku predaju radova i organizaciju onlajn ispita*).

---

## 1. Threat Model & Security Posture

High-stakes academic platforms operate in an environment characterized by unique adversarial incentives:
- **Student Incentive:** Attempting to alter submission timestamps, bypass deadlines, bypass single-attempt constraints, extract correct answers from the browser DOM, manipulate JavaScript timers, impersonate peers, brute-force activation tokens, or upload malicious payloads.
- **Faculty / Administrative Incentive:** Requiring provable, tamper-evident submission verification, strict data ownership segregation, auditable attempt cancellations, and tamper-resistant academic records.

AIspit implements **Defense-in-Depth**, operating under the foundational premise that **the client browser is completely untrusted**.

---

## 2. Multi-Tier Access Control Architecture

Authorization is enforced hierarchically across three discrete layers. No endpoint executes business logic without passing all applicable gates:

```
+-------------------------------------------------------------------------+
| TIER 1: AUTHENTICATION ENFORCEMENT (@login_required)                    |
| - Verifies presence and signature of session cookie                     |
| - Unauthenticated requests redirected to /login with flash notification |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| TIER 2: ROLE-BASED ACCESS CONTROL (@role_required(*roles))              |
| - Validates caller against allowed roles: 'admin', 'profesor',          |
|   'asistent', 'student'                                                 |
| - Cross-role violations return HTTP 403 / redirect                      |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| TIER 3: RESOURCE OWNERSHIP & TENANT BOUNDARIES (In-Route Guard)          |
| - Validates that the requested entity ID belongs to the caller          |
| - Prevents Insecure Direct Object References (IDOR)                     |
| - Intersects client form selections with authorized entity rosters      |
+-------------------------------------------------------------------------+
```

### 2.1 Implementation Code Artifacts

#### Authentication & Role Decorators (`app/routes/decorators.py`)
```python
def login_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        if 'korisnik_id' not in session:
            flash('Морате се пријавити.', 'warning')
            return redirect(url_for('auth.login'))
        return f(*args, **kwargs)
    return decorated

def role_required(*roles):
    def decorator(f):
        @wraps(f)
        def decorated(*args, **kwargs):
            if session.get('uloga') not in roles:
                flash('Немате приступ овој страници.', 'danger')
                return redirect(url_for('auth.login'))
            return f(*args, **kwargs)
        return decorated
    return decorator
```

#### Resource Ownership Enforcement (IDOR Mitigation)
Checking role access alone is insufficient: an assistant or professor could manipulate a URL ID parameter (`/profesor/prijava/<id>`) to view or mutate another faculty member's examination. In-route ownership guards resolve this:

```python
def _check_prijava_ownership(prijava_id, profesor_id):
    """Returns prijava record ONLY if owned by the authenticated professor."""
    pr = PrijavaModel.get_by_id(prijava_id)
    if pr and pr['Profesor_ID'] == profesor_id:
        return pr
    return None
```
*If ownership fails, the route immediately halts execution, logs an audit warning, and redirects.*

---

## 3. Cryptography & Credential Management

### 3.1 Password Storage & Hash Derivation
- **Algorithm:** Passwords are never stored or logged in plaintext. They are hashed using **Werkzeug's security framework** utilizing modern salted scrypt or PBKDF2:SHA256 implementations with a high work factor.
- **Salting:** Each hash incorporates an automatically generated cryptographic salt, preventing rainbow-table precomputation attacks.
- **Zero Exposure on Batch Import:** When administrators batch-import students from tabular rosters, temporary passwords are generated in-memory. Only the derived cryptographic hash is persisted to MySQL; the raw passwords are rendered once on the administrative confirmation screen and are never retrievable by any API or administrative endpoint thereafter.

### 3.2 Client Session Cryptographic Integrity
- **Signed Cookies:** Client sessions are cryptographically signed using Flask’s HMAC mechanism driven by a high-entropy `SECRET_KEY`.
- **Minimal Cookie Payload:** The session cookie contains solely the user's primary key (`korisnik_id`) and role designation (`uloga`). All permissions, profile attributes, and course assignments are queried dynamically from MySQL on every request, ensuring that immediate account suspensions or permission changes take effect instantaneously without waiting for cookie expiration.

### 3.3 Enumeration Attack Prevention
Authentication failures return uniform, calibrated error messaging (`"Неисправни приступни подаци"`) regardless of whether the submitted email exists or the password was incorrect. This prevents adversaries from enumerating registered institutional email addresses.

---

## 4. SQL Injection Mitigation & Transactional Integrity

### 4.1 Strict Parameterized Statements
AIspit avoids dynamic SQL string concatenation for user-supplied data. Across domain models, database queries utilize parameterized prepared statements through the official `mysql-connector-python` driver:

```python
# Parameterized Query with %s Placeholder
cur.execute("""
    SELECT ID, Naziv, TrajanjeMin 
    FROM Test 
    WHERE Profesor_ID = %s AND DeletedAt IS NULL
""", (profesor_id,))
```
*User-supplied data is delivered out-of-band to the MySQL server protocol, mitigating SQL injection by preventing inputs from being interpreted as SQL commands.*

### 4.2 Explicit Multi-Statement Transactions (`autocommit=False`)
Automatic committing is disabled on all database sessions. Multi-step operations (such as processing an assignment submission, marking an activation code as consumed, and recording the audit log) are executed within atomic transactions:
- If any operation fails or constraint is violated, `g.db.rollback()` is invoked.
- Prevents orphaned records or partial state mutations.

---

## 5. File Submission Security & Keyed Integrity Verification

### 5.1 Content-Based MIME Magic-Byte Validation
Relying on client-supplied file extensions (`.pdf`, `.zip`) is a critical security vulnerability. Adversaries can rename an executable (`payload.exe` or `malware.sh`) to `document.pdf`.

AIspit validates file authenticity by inspecting the **initial magic bytes** of the file stream against known binary signatures:
- **PDF:** Must begin with `%PDF-` (`0x25 0x50 0x44 0x46`).
- **ZIP / DOCX:** Must begin with `PK\x03\x04` (`0x50 0x4B 0x03 0x04`).

Files failing magic-byte inspection are rejected before persistence.

### 5.2 Path Traversal & File System Sanitization
User-controlled filenames are stripped and discarded before writing to disk:
1. The server generates a collision-proof hexadecimal identifier: `uuid.uuid4().hex`.
2. The file is saved under an isolated, non-executable directory structure partitioned by session ID:
   ```
   uploads/<prijava_id>/<uuid_token>.<validated_extension>
   ```
3. The student's original human-readable filename is stored safely in MySQL for presentation and downloading headers (`Content-Disposition: attachment; filename="..."`).

### 5.3 The "Public Upload Directory" Vulnerability Case Study
During development, a serious vulnerability was uncovered: uploaded student assignment files were initially stored under `app/static/uploads/`.
- **The Vulnerability:** Because the `static/` directory was directly served by the web server without application routing, any unauthenticated visitor who obtained or guessed the direct URL could download another student’s confidential assignment. While UUID filenames made brute-force guessing difficult, URLs could leak via browser histories, shared links, or proxy logs.
- **The Remediation:**
  1. The upload root was moved entirely **outside** the web server static root to `/uploads/`.
  2. Direct web server static mapping was eliminated.
  3. All file downloads were routed through an authenticated controller endpoint (`/profesor/preuzmi/<predaja_id>` and `/student/preuzmi/<predaja_id>`).
  4. The controller asserts `@login_required`, validates ownership (`_check_prijava_ownership`), and streams the file using Flask’s `send_from_directory()`.

### 5.4 HMAC-SHA256 Submission Integrity Verification (MAC)
To detect any unauthorized replacement, backdating, or modification of student files and submission records after the deadline, each submission generates a keyed HMAC-SHA256 message authentication code (MAC):

$$\text{MAC} = \text{HMAC-SHA256}\Big(K_{\text{secret}}, \text{StudentID} \parallel \text{PrijavaID} \parallel \text{AktivacioniKodID} \parallel \text{ServerTimestamp} \parallel \text{FileSHA256}\Big)$$

- The calculated MAC is stored in `predaja.SubmissionKey`.
- Faculty can click **"Провери потпис"** (the UI integrity check button) on any submission to recompute the HMAC-SHA256 code using the server's secret key and verify that neither the record nor the file has been modified. Any direct database tampering (e.g., modifying the timestamp from 03:08 to 02:00, or replacing the file on disk) causes the verification check to fail.
- Includes key version tracking (`KeyVerzija`, `KeyID`) to facilitate future key rotation without invalidating historical verification.

### 5.5 Activation Code Abuse & Rate Limiting
- **Atomic Reservation:** Activation codes are consumed in a single atomic SQL statement preventing race conditions.
- **Brute-Force Rate Limiting:** Enforces a limit of **5 activation code validation attempts per minute per student**.
- **Audit Logging:** Every attempt (success or failure) is logged to `pokusajpredaje` recording the candidate ID, session ID, client IP address, timestamp, and failure reason.

---

## 6. Examination Integrity & Anti-Cheating Controls

AIspit approaches online testing with realistic engineering guarantees, distinguishing between **verifiable server-enforced controls** and **browser-side deterrent monitoring**. Browser-side JavaScript detects and reports viewport-related events, while the server validates and records reported violations and enforces the configured examination termination threshold:

```
+--------------------------------------------------------------------------+
| SERVER-ENFORCED CONTROLS (Authoritative & Server-Evaluated)              |
| - Server-side time boundary measurement (beograd_sada())                |
| - Zero correct answer flags in client HTML/JS                           |
| - Hard single-attempt limit via database UNIQUE constraint              |
| - Immutable snapshot freezing of exam questions and choices             |
| - Server-enforced automatic submission upon deadline expiry             |
+--------------------------------------------------------------------------+
                                    |
+--------------------------------------------------------------------------+
| BROWSER-SIDE MONITORING (Deterrent & Audit Trail)                        |
| - Distraction-free exam template (base_polaganje.html: no links/nav)     |
| - Fullscreen enforcement & Page Visibility / Blur detection             |
| - Real-time violation counter incremented on server                      |
| - Automatic test termination at violation threshold (3 events)          |
| - Debounced event handling preventing accidental refresh lockouts        |
+--------------------------------------------------------------------------+
```

> **Security & Monitoring Boundaries:**
> Browser-side viewport monitoring acts strictly as an operational deterrent and audit trail against casual tab switching or unauthorized local application navigation during exams. It does not and cannot prevent external collaboration, secondary physical devices, physical notes or reference materials, or all possible cheating methods. Authoritative examination validity rests on server-enforced snapshot freezing, question/choice shuffling, and rigid server-measured time limits.

### 6.1 Server-Authoritative Time Enforcement
- The timer displayed in the browser is **strictly informational**, calculated as:
  $$\text{RemainingSeconds} = \max\Big(0, \big(\min(\text{PocetakVreme} + \text{Trajanje}, \text{DatumZavrsetka}) - \text{beograd\_sada}()\big).\text{total\_seconds}()\Big)$$
- If a candidate pauses JavaScript execution, rewinds their local operating system clock, or disables browser scripts, the server evaluates `beograd_sada() > efektivni_kraj()` upon the next HTTP request, immediately closing the attempt and evaluating stored answers.

### 6.2 Zero Answer Leakage in DOM
When rendering the exam interface (`/student/test/<id>`), the backend queries `studenttestpitanje` and `studenttestodgovor`. The `Tacan` (is_correct) boolean column is **deliberately excluded** from the template context. The browser DOM contains only choice IDs and question text; inspecting DOM elements or network traffic reveals zero clues regarding answer correctness.

### 6.3 Anti-Cheating Viewport & Fullscreen Monitoring
- During the exam, JavaScript attaches listeners to `visibilitychange`, `blur`, and fullscreen exit events.
- Leaving the active exam tab triggers an asynchronous notification to the server, incrementing `BrojNapustanja` in `studenttest`.
- The candidate is presented with an overlay warning:
  > *"Напустили сте тест (1/3). Излазак са екрана теста се бележи. Још 2 напуштања и тест се предаје аутоматски."*
- Upon the 3rd recorded violation, the server automatically flags the test status as `'prekinut'` (aborted) and evaluates submitted answers up to that point.
- **Event Debouncing:** A temporal debounce window was engineered to ensure that a browser page reload does not emit multiple concurrent events and prematurely terminate the exam.

### 6.4 Hard Single-Attempt Rule & Audited Invalidation
- Enforced at the database level by a unique composite constraint:
  ```sql
  UNIQUE KEY (Student_ID, Prijava_ID)
  ```
- If a student encounters genuine hardware failure or power outage, only the owning professor or system administrator can reset the attempt. The cancellation leaves a permanent audit trail (`PonistioKorisnik_ID`, `UpdatedAt`).
