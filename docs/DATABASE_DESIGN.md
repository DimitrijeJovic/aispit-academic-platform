# AIspit — Database Architecture & Relational Design

This document provides a comprehensive technical breakdown of the database schema for **AIspit** (*Sistem za elektronsku predaju radova i organizaciju onlajn ispita*), implemented on **MySQL 8.0** using the **InnoDB** storage engine.

---

## 1. Design Principles & Schema Metrics

The database schema is engineered for strict relational integrity, historical auditability, and deterministic concurrency control:

- **Storage Engine:** MySQL 8.0 InnoDB (ACID compliant, foreign keys, row-level locking).
- **Character Set & Collation:** `utf8mb4` with `utf8mb4_unicode_ci` (full Serbian Latin and Cyrillic support, emojis, extended Unicode).
- **Entities & Tables:** 23 operational tables (plus 3 legacy read-only views).
- **Constraints & Indexes:**
  - **39 Foreign Keys** enforcing referential integrity across all relationships.
  - **16 Unique Constraints** preventing logical duplication (duplicate index numbers, duplicate enrollments, duplicate single-attempt exams).
  - **46 Secondary Indexes** optimized for query lookups, foreign key traversal, and range filtering.
- **Transaction Model:** Explicit transactions (`autocommit=False`) with row-level locks for critical operations such as activation code reservation and exam initialization.
- **Zero ORM Overhead:** Direct interaction via `mysql-connector-python` using parameterized SQL prepared statements.

---

## 2. Entity-Relationship Overview

The schema is categorized into four functional domains:

```
+-----------------------------------------------------------------------------------+
| 1. CORE ACADEMIC ENTITIES                                                         |
|    korisnik (User accounts, auth hashes, roles)                                   |
|    smer (Study programs / majors)                                                 |
|    predmet (Academic subjects / courses)                                          |
|    smerpredmet (Subject-to-Program mappings with Year & Semester attributes)      |
|    profesor (Faculty profiles linked to korisnik)                                 |
|    student (Student profiles with index numbers, linked to korisnik)              |
|    izvodjenjepredmeta (Faculty-to-Subject-to-Program teaching instances)          |
|    studentpredmet (Student course enrollments)                                    |
+-----------------------------------------------------------------------------------+
                                      |
                                      v
+-----------------------------------------------------------------------------------+
| 2. SESSIONS & ASSIGNMENT SUBMISSIONS                                              |
|    prijava (Core session: exam, project submission, or composite)                 |
|    aktivacionikod (Single-use & shared activation codes with expiry)              |
|    predaja (Submitted files, HMAC-SHA256 integrity verification, grades, teacher notes)       |
|    pokusajpredaje (Audit logging of all submission / code validation attempts)    |
+-----------------------------------------------------------------------------------+
                                      |
                                      v
+-----------------------------------------------------------------------------------+
| 3. QUESTION BANK & TEST TEMPLATES                                                 |
|    oblast (Subject topics / modules owned by faculty)                             |
|    oblastsmer (Scoping of question topics to specific study programs)             |
|    pitanje (Questions: text, point value, image path, choice vs. free text)       |
|    ponudjeniodgovor (Answer choices with correct answer flag and display order)   |
|    test (Test template specification: durations, randomization flags)             |
|    teststavka (Test sections referencing topics, quotas, selection mode)         |
|    teststavkapitanje (Explicitly assigned questions for manual section mode)      |
+-----------------------------------------------------------------------------------+
                                      |
                                      v
+-----------------------------------------------------------------------------------+
| 4. EXAM EXECUTION & FROZEN SNAPSHOT AUDIT                                         |
|    studenttest (Individual student exam attempt, timings, violation counters)    |
|    studenttestpitanje (Frozen deep-copy of questions presented to candidate)      |
|    studenttestodgovor (Frozen deep-copy of answer options in candidate order)     |
|    studentodgovor (Candidate responses, submitted selections, assigned scores)   |
+-----------------------------------------------------------------------------------+
```

---

## 3. Schema Normalization & Justified Denormalization

### 3.1 Third Normal Form (3NF) Base Architecture
The majority of the schema adheres strictly to Third Normal Form (3NF):
- **Curriculum Decoupling:** `Godina` (Year) and `Semestar` (Semester) attributes are stored in `smerpredmet` rather than in `predmet`. The identical course (e.g., *Web Programming*) can be taught during Year 2 in one study program and Year 3 in another; hence, placement is an attribute of the program-course relationship.
- **Account-Persona Separation:** Authentication credentials (`Email`, `PasswordHash`, `Uloga`, `Status`) reside exclusively in `korisnik`. Biographical details (`Ime`, `Prezime`, `BrojIndeksa`) reside in `student` or `profesor`. Accounts can be suspended or roles transitioned without mutating historical profile records.

### 3.2 Deliberate Denormalization: Immutable Exam Snapshots
A core architectural requirement of high-stakes academic testing is **tamper-evident historical integrity and temporal immutability**. 

In a purely normalized database, student attempts would link directly via foreign keys to `pitanje` and `ponudjeniodgovor`. However:
- If a professor corrects a typographical error or revises an ambiguous answer option after the exam has begun, the historical context of already-submitted or in-progress tests would be mutated.
- If a question is retired or soft-deleted, historical student score justifications become corrupted.

**The Solution:** When a student initializes an exam, AIspit performs an **atomic deep snapshot**:
1. Selected questions are copied from `pitanje` into `studenttestpitanje` (copying question text, point weighting, and image references).
2. Answer choices are copied from `ponudjeniodgovor` into `studenttestodgovor` (copying choice text, correctness flags, and randomized order).
3. Student responses (`studentodgovor`) reference the snapshot tables, not the mutable question bank.

*This denormalization represents an intentional engineering decision ensuring that completed exams remain preserved exactly as viewed by the student, indefinitely.*

---

## 4. Comprehensive Table Specifications

### 4.1 Core Academic Entities

#### `korisnik` — User Authentication & Identity
| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `ID` | INT UNSIGNED | PK, AUTO_INCREMENT | Unique user identifier |
| `Email` | VARCHAR(255) | NOT NULL, UNIQUE | University institutional email address |
| `PasswordHash` | VARCHAR(255) | NOT NULL | Werkzeug-generated password hash (salt + scrypt/pbkdf2) |
| `Uloga` | ENUM | NOT NULL | Role: `'admin'`, `'profesor'`, `'asistent'`, `'student'` |
| `Status` | ENUM | NOT NULL, DEFAULT `'aktivan'` | Account status: `'aktivan'`, `'neaktivan'`, `'blokiran'` |
| `CreatedAt` | DATETIME | NOT NULL | Record creation timestamp |
| `UpdatedAt` | DATETIME | NULL | Timestamp of last record update |
| `DeletedAt` | DATETIME | NULL | Soft-deletion timestamp |

#### `student` — Student Biographical Data
| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `ID` | INT UNSIGNED | PK, AUTO_INCREMENT | Unique student record identifier |
| `Korisnik_ID` | INT UNSIGNED | NOT NULL, FK(`korisnik.ID`) | Associated user account |
| `Ime` | VARCHAR(100) | NOT NULL | First name |
| `Prezime` | VARCHAR(100) | NOT NULL | Last name |
| `BrojIndeksa` | VARCHAR(20) | NOT NULL, UNIQUE | Student academic registration index number |
| `Smer_ID` | INT UNSIGNED | NOT NULL, FK(`smer.ID`) | Enrolled study program |
| `GodinaStudija` | TINYINT | NOT NULL | Current study year (1-4) |
| `Status` | ENUM | NOT NULL | `'aktivan'`, `'diplomirao'`, `'ispisan'` |
| `StatusFinansiranja` | ENUM | NOT NULL | `'budzet'`, `'samofinansiranje'` |

#### `profesor` — Faculty & Assistant Profiles
| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `ID` | INT UNSIGNED | PK, AUTO_INCREMENT | Faculty profile identifier |
| `Korisnik_ID` | INT UNSIGNED | NOT NULL, FK(`korisnik.ID`) | Associated user account (holds role 'profesor' or 'asistent') |
| `Ime` | VARCHAR(100) | NOT NULL | First name |
| `Prezime` | VARCHAR(100) | NOT NULL | Last name |
| `Status` | ENUM | NOT NULL, DEFAULT `'aktivan'` | Faculty status |

---

### 4.2 Sessions & Submissions

#### `prijava` — Unified Academic Deadline / Exam Session
The central coordinating entity binding course delivery, tests, and submission parameters.

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `ID` | INT UNSIGNED | PK, AUTO_INCREMENT | Unique session identifier |
| `IzvodjenjePredmeta_ID` | INT UNSIGNED | NOT NULL, FK | Links faculty, course, and study program |
| `NazivPrijave` | VARCHAR(200) | NOT NULL | Name of session (e.g., "Kolokvijum 1", "Seminarski rad") |
| `TipPrijave` | ENUM | NOT NULL | Obligation classification name |
| `TipObaveze` | ENUM | NOT NULL | Functional type: `'rad'` (upload), `'test'` (exam), `'obostrano'` |
| `DatumPocetka` | DATETIME | NOT NULL | Opening timestamp (Belgrade wall-clock) |
| `DatumZavrsetka` | DATETIME | NOT NULL | Hard deadline timestamp |
| `DozvoljeniTipoviFajlova` | VARCHAR(200) | NULL | Permitted file extensions (e.g. `.pdf,.docx,.zip`) |
| `MaksimalnaVelicinaMB` | TINYINT | NOT NULL, DEFAULT 10 | Maximum file size in megabytes |
| `DozvoljenaPonovnaPredaja`| TINYINT(1) | NOT NULL, DEFAULT 0 | Flag permitting resubmission |
| `KoristiKod` | TINYINT(1) | NOT NULL, DEFAULT 0 | Requires activation code to submit |
| `Test_ID` | INT UNSIGNED | NULL, FK(`test.ID`) | Bound test template (if exam component enabled) |
| `TrajanjeMin` | SMALLINT | NULL | Override test duration (minutes) for this session |
| `SamoIzLaboratorije` | TINYINT(1) | NOT NULL, DEFAULT 0 | Enforces lab IP subnet restriction |

#### `predaja` — Submitted Assignment Records
| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `ID` | INT UNSIGNED | PK, AUTO_INCREMENT | Unique submission identifier |
| `Student_ID` | INT UNSIGNED | NOT NULL, FK(`student.ID`) | Submitting candidate |
| `Prijava_ID` | INT UNSIGNED | NOT NULL, FK(`prijava.ID`) | Target session |
| `AktivacioniKod_ID` | INT UNSIGNED | NULL, FK(`aktivacionikod.ID`)| Code redeemed for submission |
| `SubmissionKey` | CHAR(64) | NOT NULL | Keyed HMAC-SHA256 message authentication code (MAC) for integrity verification |
| `KeyVerzija` | TINYINT | NOT NULL, DEFAULT 1 | Secret key version for HMAC verification |
| `KeyID` | CHAR(8) | NOT NULL | Key fingerprint |
| `OriginalniNazivFajla` | VARCHAR(500) | NOT NULL | User's local file name for display/export |
| `SacuvaniNazivFajla` | VARCHAR(500) | NOT NULL | System-generated sanitized UUID file name on disk |
| `PutanjaFajla` | VARCHAR(1000) | NOT NULL | Relative disk path (`<prijava_id>/<uuid>.<ext>`) |
| `Ocena` | ENUM | NULL | Qualitative evaluation mark |
| `OcenaBroj` | TINYINT | NULL | Numerical score (0-100) |
| `ProfesorKomentar` | TEXT | NULL | Evaluator feedback |
| `DatumIVremePredaje` | DATETIME | NOT NULL | Server-recorded submission timestamp |

*Constraint:* `UNIQUE KEY (Student_ID, Prijava_ID)` prevents duplicate submissions per session unless resubmission is authorized.

#### `aktivacionikod` — Submission Gate Codes
| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `ID` | INT UNSIGNED | PK, AUTO_INCREMENT | Code identifier |
| `Kod` | VARCHAR(50) | NOT NULL, UNIQUE | Alphanumeric activation token (e.g. `DEMO-CODE-XXXX`) |
| `Prijava_ID` | INT UNSIGNED | NOT NULL, FK(`prijava.ID`) | Bound session |
| `Student_ID` | INT UNSIGNED | NULL, FK(`student.ID`) | Target student (NULL for multi-use shared codes) |
| `Aktivan` | TINYINT(1) | NOT NULL, DEFAULT 1 | Administrative activation switch |
| `Iskoriscen` | TINYINT(1) | NOT NULL, DEFAULT 0 | Atomic consumption flag |
| `Deljeni` | TINYINT(1) | NOT NULL, DEFAULT 0 | Shared multi-student token flag |
| `VaziDo` | DATETIME | NOT NULL | Absolute expiration timestamp |

---

### 4.3 Question Bank & Examination Engine

#### `test` — Test Template Specification
| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `ID` | INT UNSIGNED | PK, AUTO_INCREMENT | Test template identifier |
| `Profesor_ID` | INT UNSIGNED | NOT NULL, FK(`profesor.ID`) | Authoring professor |
| `Predmet_ID` | INT UNSIGNED | NOT NULL, FK(`predmet.ID`) | Bound academic course |
| `Naziv` | VARCHAR(200) | NOT NULL | Test title |
| `MesajPitanja` | TINYINT(1) | NOT NULL, DEFAULT 1 | Shuffle question ordering per candidate |
| `MesajOdgovore` | TINYINT(1) | NOT NULL, DEFAULT 1 | Shuffle answer options per candidate |
| `MesajOblasti` | TINYINT(1) | NOT NULL, DEFAULT 0 | Shuffle section display order |
| `TrajanjeMin` | SMALLINT | NOT NULL | Default test time limit (minutes) |

#### `studenttest` — Candidate Exam Execution Snapshot
| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `ID` | INT UNSIGNED | PK, AUTO_INCREMENT | Attempt identifier |
| `Student_ID` | INT UNSIGNED | NOT NULL, FK(`student.ID`) | Candidate |
| `Prijava_ID` | INT UNSIGNED | NOT NULL, FK(`prijava.ID`) | Exam session |
| `Test_ID` | INT UNSIGNED | NOT NULL, FK(`test.ID`) | Instantiated test template |
| `PocetakVreme` | DATETIME | NOT NULL | Exact server timestamp when attempt was initialized |
| `KrajVreme` | DATETIME | NULL | Timestamp when exam was finished or closed |
| `Status` | ENUM | NOT NULL | `'u_toku'`, `'zavrsen'`, `'istekao'`, `'prekinut'` |
| `BrojNapustanja` | INT | NOT NULL, DEFAULT 0 | Violation counter (leaving exam viewport/fullscreen) |
| `PocetnaIP` | VARCHAR(45) | NOT NULL | Client IP address at attempt start |
| `PonistioKorisnik_ID`| INT UNSIGNED| NULL, FK(`korisnik.ID`) | Audit ID of professor/admin if attempt was invalidated |

*Constraint:* `UNIQUE KEY (Student_ID, Prijava_ID)` guarantees exactly one attempt per student per session.

---

## 5. Defensive Data Integrity Patterns

### 5.1 Universal Soft Deletion Pattern
Physical `DELETE` statements are prohibited across all operational tables. Every table includes a `DeletedAt DATETIME NULL` column.
- Queries explicitly enforce `WHERE DeletedAt IS NULL`.
- Foreign key dependencies are preserved permanently.
- Re-enrolling a previously dropped entity re-activates the existing row (`DeletedAt = NULL`) to maintain historical continuity and avoid unique key collisions.

### 5.2 Atomic State Updates (Race Condition Immunity)
Concurrency risks (such as multiple parallel submissions attempting to claim a single activation code) are eliminated through atomic conditional writes rather than multi-step read-then-write logic:

```sql
UPDATE AktivacioniKod 
SET Iskoriscen = 1 
WHERE ID = %s 
  AND Iskoriscen = 0 
  AND Aktivan = 1 
  AND DeletedAt IS NULL 
  AND VaziDo > %s;
```
*If `cursor.rowcount == 0`, the transaction is rolled back and the attempt rejected immediately.*

### 5.3 Deterministic Timezone Synchronization
To prevent timezone skew across operating systems or database configurations, each connection executes:
```sql
SET time_zone = '+01:00'; -- Adjusted dynamically for Europe/Belgrade CEST (+02:00)
```
Ensuring Python wall-clock calculations (`beograd_sada()`) and database interval evaluations match down to the second.
