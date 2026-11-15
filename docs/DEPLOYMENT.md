# AIspit — Deployment & Infrastructure Architecture

This document details the production deployment, infrastructure architecture, environment configurations, and reverse proxy topology of **AIspit** (*Sistem za elektronsku predaju radova i organizaciju onlajn ispita*).

---

## 1. Hosting Environment & Operational Topology

AIspit is deployed on the institutional server infrastructure of the **Academy of Technical and Educational Vocational Studies in Niš (ATVSS Niš)**:

- **Live Production URL:** [https://aispit.vtsappsteam.edu.rs/login](https://aispit.vtsappsteam.edu.rs/login)
- **Primary Audience:** Students, teaching assistants, professors, and academic administrators.
- **Dual Operating Modes:**
  1. **Campus-Wide Web Portal:** Accessible via HTTPS over institutional subnets and authorized academic networks for routine assignment submissions and syllabus access.
  2. **Isolated Laboratory Examination Subnet:** Engineered to run fully functional within self-contained examination laboratories with zero public internet connectivity (all CSS, scripts, and vendor icons bundled locally).

```
+-------------------------------------------------------------------------+
|                              PUBLIC INGRESS                             |
|          Client Browsers (Desktop Workstations, Lab Terminals)          |
+-------------------------------------------------------------------------+
                                    |
                           HTTPS (Port 443) / TLS
                                    |
                                    v
+-------------------------------------------------------------------------+
|                  REVERSE PROXY & WEB SERVER (Nginx)                     |
|  - TLS/SSL Termination                                                  |
|  - Request Rate Limiting & Header Sanitization                          |
|  - Direct Delivery of Static Assets: /static/ -> app/static/            |
|  - EXCLUSION: /uploads/ is explicitly NOT mapped to any public URL!     |
+-------------------------------------------------------------------------+
                                    |
                         Unix Domain Socket / TCP
                            WSGI Protocol (HTTP)
                                    |
                                    v
+-------------------------------------------------------------------------+
|                  WSGI APPLICATION SERVER (Gunicorn)                     |
|  - Python 3.10+ Execution Environment                                   |
|  - Pre-fork Worker Model (gunicorn -w 4 -b 127.0.0.1:8000 run:app)      |
|  - Flask Core Engine (Application Factory create_app)                   |
|  - Authenticated File Streaming (/uploads/)                             |
+-------------------------------------------------------------------------+
                                    |
                          TCP Connection (Port 3306)
                             mysql-connector-python
                                    |
                                    v
+-------------------------------------------------------------------------+
|                       RELATIONAL DATABASE TIER                          |
|  - MySQL 8.0 Community Server (InnoDB Storage Engine)                   |
|  - Timezone Session Synchronization (Europe/Belgrade)                   |
|  - UTF-8 Multi-byte Encoding (utf8mb4_unicode_ci)                       |
+-------------------------------------------------------------------------+
```

---

## 2. Web Server & Reverse Proxy Hardening (Nginx)

The production deployment employs **Nginx** as a high-performance reverse proxy positioned in front of the Python WSGI application server (Gunicorn).

### 2.1 Critical Security Rules: Upload Directory Isolation
As identified during the security audit documented in the thesis, student submissions are stored in an absolute filesystem directory (`/uploads/`) located **completely outside** the web server document root:

```nginx
# Static assets: served directly by Nginx for high throughput
location /static/ {
    alias /var/www/aispit/app/static/;
    expires 30d;
    add_header Cache-Control "public, no-transform";
}

# CRITICAL SECURITY DIRECTIVE:
# The /uploads/ folder has NO location block in Nginx.
# Any direct request like https://aispit.vtsappsteam.edu.rs/uploads/file.zip
# returns HTTP 404 Not Found from Nginx.
# Files can ONLY be downloaded via authenticated Flask streaming routes:
# /profesor/preuzmi/<predaja_id> or /student/preuzmi/<predaja_id>

location / {
    proxy_pass http://127.0.0.1:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Maximum payload size (aligned with Flask MAX_CONTENT_LENGTH)
    client_max_body_size 16M;
}
```

### 2.2 Security Headers
Production HTTP responses incorporate defensive headers to mitigate clickjacking and MIME confusion:
- `X-Frame-Options: SAMEORIGIN` (prevents malicious iframe framing).
- `X-Content-Type-Options: nosniff` (forces browsers to respect declared Content-Type).
- `Referrer-Policy: strict-origin-when-cross-origin`.

---

## 3. Environment Configuration & Secrets Management

The application runtime is controlled via a centralized `config.py` module backed by operating system environment variables:

| Environment Variable | Description | Security Considerations |
| :--- | :--- | :--- |
| `FLASK_ENV` | Environment identifier (`production` vs `testing`) | Must be set to `production` in live environments to disable debug modes. |
| `SECRET_KEY` | Cryptographic secret for signing client session cookies | High-entropy random string (at least 32 bytes). Never committed to version control. |
| `HMAC_KEY` | Secret key used to generate submission integrity authentication codes (MAC) | High-entropy key. Protected with version tracking (`KeyVerzija`) for rotation. |
| `DB_HOST` | MySQL database host / socket path | Typically `127.0.0.1` or internal private VPC IP. |
| `DB_USER` | Dedicated MySQL database user | Least-privilege account granted rights exclusively on `aispit_db`. |
| `DB_PASSWORD` | Database user password | Strong alphanumeric secret. |
| `DB_NAME` | Schema name (`aispit_db`) | Production database schema. |
| `UPLOAD_FOLDER` | Filesystem path for stored student uploads | Absolute path located outside web root (e.g. `/var/data/aispit/uploads`). |
| `MAX_CONTENT_LENGTH` | Application-level upload ceiling (e.g. 16MB) | Protects application memory from denial-of-service via huge payloads. |

---

## 4. Timezone & Locale Synchronization

Accurate academic deadlines depend upon precise temporal coordination across operating systems:

```python
# app/db.py
from datetime import datetime
from zoneinfo import ZoneInfo

_BEOGRAD = ZoneInfo("Europe/Belgrade")

def beograd_sada():
    """Single Source of Truth: returns naive Belgrade wall-clock time.
    Guarantees consistent deadline comparisons regardless of host OS timezone."""
    return datetime.now(_BEOGRAD).replace(tzinfo=None)

def get_db():
    if 'db' not in g:
        g.db = mysql.connector.connect(...)
        cur = g.db.cursor()
        cur.execute("SET time_zone = %s", (_belgrade_offset(),))
        cur.close()
    return g.db
```

---

## 5. Live Production Verification vs. Thesis Claims

To ensure complete transparency, this section distinguishes between features directly verified via the live public web application and features documented in the thesis:

| Feature / Subsystem | Live Application Observation (`https://aispit.vtsappsteam.edu.rs/login`) | Final Thesis Documentation | Verification Status |
| :--- | :--- | :--- | :--- |
| **Authentication Portal** | Verified live split-screen layout, responsive mobile design, institution branding. | Detailed in Section 4.4 and Section 3.2. | **Directly Verified Live** |
| **Visual Identity & Theme** | Verified dark mode default (`data-theme="dark"`), high-contrast logos, custom SVG graphics. | Detailed in Section 2.3 and Section 3.2. | **Directly Verified Live** |
| **Client Validation & Toasts** | Verified client-side required field enforcement and dynamic animated toast feedback. | Detailed in Section 2.3. | **Directly Verified Live** |
| **Admin Dashboard & Management** | Hidden behind authentication; public access restricted. | Full schema, routes, and workflows documented in Section 4.3 and 4.4. | **Documented in Thesis** |
| **Faculty Question Bank & Test Builder** | Hidden behind authentication; public access restricted. | Documented in Section 4.5; verified via test suite. | **Documented in Thesis** |
| **Student Exam Taking & Autosave** | Hidden behind authentication; public access restricted. | Documented in Section 4.5, 4.6; verified via 27 browser tests. | **Documented in Thesis** |
| **Assignment Upload & HMAC Verification** | Hidden behind authentication; public access restricted. | Documented in Section 4.7; verified via 25 upload tests. | **Documented in Thesis** |
| **Database Schema & 23 Tables** | Direct SQL inspection not possible over public HTTP. | Verified via canonical SQL script and schema diagram. | **Documented in Thesis** |
