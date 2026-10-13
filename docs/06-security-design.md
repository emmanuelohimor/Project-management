# Security Design

**Project:** FimeBag Internal Task Management API
**Deliverable:** 6 of 9, Week 1
**Depends on:** `01-requirements-analysis.md`, `03-database-design.md`, `04-api-design.md`, `05-backend-architecture.md`

No security code is written this week. This document says what must be protected, what could go wrong, and how each risk will be handled when the system is built.

---

## 1. Summary

| Topic | Approach |
|---|---|
| **Password storage** | BCrypt hashes. Never stored, returned or logged in plain text |
| **Authentication** | Email and password login. Generic error messages. Failed-login limit |
| **JWT** | Signed, short-lived (1 hour) access token. Secret kept outside the code |
| **Authorisation** | Two checks on every request: the role, then the data |
| **Roles** | Administrator, Project Manager, Staff. Least privilege. Deny by default |
| **Access control** | Every record is checked against the user, so guessing an ID gives nothing |
| **Input validation** | Validate every request. Use DTOs so clients cannot set fields they should not |
| **Sensitive information** | Hashes and secrets never leave the system. Responses use DTOs |
| **Audit logging** | Important actions recorded in the same transaction. Append-only |
| **Error handling** | One standard format. No stack traces. Same message for wrong email or wrong password |
| **Secrets and configuration** | Environment variables. Nothing secret in Git |
| **Database security** | Least-privilege database user. Audit table cannot be changed. Encrypted connection |

---

## 2. What We Protect and What Could Go Wrong

### What we protect

| Asset | Why it matters |
|---|---|
| User accounts and passwords | If stolen, an attacker can act as that user |
| Project, task and worklog data | Internal company information |
| The audit log | It is the record of who did what. It must be trustworthy |
| The JWT signing secret and database password | If leaked, an attacker can create tokens or read the whole database |

### Main threats and the answer to each

| Threat | Example | Defence |
|---|---|---|
| Guessing passwords | Repeated login attempts | Failed-login limit, BCrypt, password rules |
| Stolen database | Attacker reads the users table | Passwords are BCrypt hashes, not readable |
| Stolen or reused token | Token copied from a device | Short expiry, HTTPS only, user re-checked on every request |
| Privilege escalation | Staff member tries to approve a task or change a role | Role check plus data check on the server |
| Guessing record IDs | Changing a task ID in the URL | Policy check on every object, UUIDs |
| Bad input | SQL injection, extra JSON fields | Validation, parameterised queries, DTOs |
| Data leaking in responses | Password hash returned in JSON | DTOs only, never entities |
| Tampering with the audit trail | Deleting evidence | Append-only, database permissions |
| Secrets in Git | Password committed by mistake | Environment variables, `.gitignore`, secret scanning |
| Revealing internals | Stack trace shown to the user | Global exception handler |

---

## 3. Password Storage

- Passwords are hashed with **BCrypt**, a slow hash made for passwords, with a work factor of 12. The salt is built into each BCrypt hash.
- **Plain-text passwords are never saved, returned, or written to logs.** The column is `users.password_hash`, and the field is never in any response DTO.
- **Password rules** (checked when a user is created and when a password is changed): 8 to 72 characters, with an upper-case letter, a lower-case letter and a digit. The limit of 72 exists because BCrypt ignores anything beyond 72 bytes.
- When the email does not exist, the login still runs a dummy BCrypt check, so response time does not reveal whether an email is registered.
- Hashes are one-way: nobody, including administrators, can read a password.
- **Changing a password:** a user changes their own password with `PATCH /users/me/password`. They must send the current password and a new one that follows the same rules and differs from the current one. The change is recorded in the audit log as `PASSWORD_CHANGED`.

---

## 4. Authentication

1. The client sends `POST /api/v1/auth/login` with `email` and `password`.
2. The server finds the user and checks the password against the hash.
3. If the password is correct but the account is deactivated (`is_active = false`), the server returns **403**. This check happens *after* the password is verified, so it does not reveal which emails exist.
4. If everything is correct, the server returns a JWT.

**Login protections**
- **One generic message** for a wrong email or a wrong password: "Invalid email or password" (**401**).
- **Failed-login limit:** after 5 failed attempts for the same email within 15 minutes, further attempts are refused for 15 minutes with **429 Too Many Requests**.
- Every failed login is written to the audit log (see section 9).
- Every other endpoint requires a valid token. `POST /auth/login` is the only public endpoint.

---

## 5. JWT (JSON Web Token)

| Setting | Decision | Reason |
|---|---|---|
| **Signing** | HS256 (HMAC with a secret key) | One service signs and checks its own tokens. A shared secret is enough |
| **Secret** | At least 32 random bytes, from an environment variable | A weak or committed secret lets anyone forge a token |
| **Lifetime** | 60 minutes (`expiresIn: 3600`) | Limits the damage if a token is stolen |
| **Contents** | User ID (`sub`), issued-at, expiry. Nothing sensitive | A token can be read by anyone who holds it |
| **Sent in** | `Authorization: Bearer <token>` header | Not stored in cookies |
| **Checked** | Signature, expiry and algorithm on every request. Tokens signed with `none` or another algorithm are rejected | Prevents forged tokens |
| **Transport** | HTTPS only | Tokens must not travel in plain text |

**The database is the source of truth, not the token.** On every request the security filter loads the user from the database and takes the role and `is_active` flag from there, not from the token (decision 6.3 in the architecture). So a deactivated user or a changed role takes effect immediately, even though the token itself is still valid.

**Not in this version:** logout and refresh tokens. Tokens simply expire after an hour. To invalidate all tokens at once, change the signing secret.

---

## 6. Roles, Authorisation and Access Control

### 6.1 Principles
- **Deny by default.** Any endpoint not explicitly allowed requires authentication and a matching role.
- **Least privilege.** Each role gets only what it needs.
- **The server decides.** The front end can hide buttons, but every rule is enforced on the server.

### 6.2 Two checks on every request

1. **Role check:** Spring Security method security (`@PreAuthorize`) confirms the user's role may call the endpoint. Failure: **403**.
2. **Data check:** a policy class in the service confirms the user may touch *this record*, for example "this Project Manager is assigned to this task's project". Failure: **403**.

### 6.3 Permission matrix

| Action | Administrator | Project Manager | Staff |
|---|---|---|---|
| Create, update, deactivate users; assign roles | ✔ | | |
| Create projects, change project details and status | ✔ | | |
| Assign or remove Project Managers on a project | ✔ | | |
| View projects | All | Assigned projects | Projects containing their tasks |
| Create, edit and assign tasks | | Assigned projects only | |
| View tasks | All (read-only) | Tasks in assigned projects | Their own tasks |
| Start a task, submit it for review | | | Their own tasks |
| Approve or reject a submitted task | | Assigned projects only | |
| Record a worklog | | | Their own tasks |
| View worklogs | ✔ | Assigned projects | Their own tasks |
| View the audit log | ✔ | | |
| Change their own password | ✔ | ✔ | ✔ |

### 6.4 Protecting individual records
Changing an ID in the URL must never expose someone else's data. Every request that uses an ID loads the record and checks it against the policy before doing anything. IDs are UUIDs, so they cannot be guessed by counting upward, but the policy check is the real protection.

### 6.5 Preventing privilege escalation
- Staff cannot mark a task `DONE`. Only the approve action does that, and only for an assigned Project Manager (rule 17).
- `PATCH /tasks/{id}` does not accept a status. Role changes are only possible through the Administrator-only user endpoints.
- Completed tasks cannot be moved backwards by any endpoint.

---

## 7. Input Validation

Every request is validated **before** it reaches the business logic.

| Risk | Defence |
|---|---|
| Missing, wrong-typed or oversized fields | Bean Validation (`@Valid`) on every request DTO. Lengths match the database columns |
| Invalid values | Enums for statuses, priority and roles. Valid email format. Valid UUIDs and dates |
| Clients setting fields they should not (mass assignment) | Request DTOs contain only the allowed fields. A client cannot send `id`, `role`, `status` or `completed_at` and have them stored |
| SQL injection | Only parameterised queries through Spring Data JPA. No SQL built from strings |
| Injection through sort and filter parameters | Sort fields are checked against a whitelist. Filter values are parsed as enums, UUIDs or dates |
| Huge responses or requests | Page size capped at 100. Request body size limited |
| Cross-site scripting (XSS) | The API returns JSON, not HTML. Data is stored as entered, and the front end must escape it when displaying |

Business rules are validated in the service layer, for example "the assignee must be an active Staff member". These return **422** or **409**, as in the API design.

---

## 8. Sensitive Information

| Data | Handling |
|---|---|
| `password_hash` | Never returned. Never logged |
| JWT tokens | Never logged. Never stored by the server |
| JWT secret, database password | Environment variables only |
| Names, emails | Returned only to users allowed to see them |
| IP addresses (in the audit log) | Visible to Administrators only |
| Rejection reasons and comments | Visible to people who can see the task, and in the audit log to Administrators |

- Responses are built only from DTOs, so internal fields cannot leak by accident.
- Logs contain IDs and actions, not passwords, tokens or full request bodies.
- Personal data (names, emails, IP addresses) is limited to what the system needs. Nigerian data protection law may apply to it, so FimeBag should confirm its retention rules for audit data.

---

## 9. Audit Logging

**What is recorded:** who did it (user), what (action), which record (entity type and ID), details of the change, the IP address, and the time.

| Area | Actions recorded |
|---|---|
| Login | `LOGIN`, `LOGIN_FAILED` |
| Users | `USER_CREATED`, `USER_UPDATED`, `USER_ROLE_CHANGED`, `USER_DEACTIVATED`, `USER_REACTIVATED`, `PASSWORD_CHANGED` |
| Projects | `PROJECT_CREATED`, `PROJECT_UPDATED`, `PROJECT_STATUS_CHANGED`, `PROJECT_MANAGER_ADDED`, `PROJECT_MANAGER_REMOVED` |
| Tasks | `TASK_CREATED`, `TASK_UPDATED`, `TASK_ASSIGNED`, `TASK_STARTED`, `TASK_SUBMITTED`, `TASK_APPROVED`, `TASK_REJECTED` |
| Worklogs | `WORKLOG_CREATED` |

**Rules**
- **Same transaction as the action.** If the action fails, no audit row is saved. If saving the audit row fails, the action is rolled back.
- **Failed logins are the exception.** They are saved in a separate transaction so they survive, with no user attached.
- **Never recorded:** passwords, tokens and secrets.
- **Append-only.** The application only inserts into `audit_logs`, and its database account is not allowed to update or delete rows there (section 12).
- **Read-only access** for Administrators through `GET /audit-logs`. There are no endpoints to change or delete entries.
- **Audit log versus application log.** The audit log is a business record. The application log is a technical log for developers. The two are separate, and neither contains secrets.

---

## 10. Error Handling

- **One standard error format** for every error (API design, section 11).
- **No internal details** in responses: no stack traces, SQL, class names or file paths. Unexpected errors return a generic `500` message, and the details go to the server log only.
- **Correct status codes:** 401 when the user is not logged in, 403 when the user is logged in but not allowed.
- **Login errors** never say whether the email or the password was wrong.
- **Validation errors** name the field and the problem, but never repeat sensitive values back.
- Spring Security's own 401 and 403 responses are replaced with the same standard format, so they look like every other error.

---

## 11. Secrets and Configuration

- **All secrets come from environment variables:** the JWT secret, database URL, username and password. The code and `application.yml` hold only placeholders.
- **`.gitignore` excludes** `.env` files and any local config that holds real values. Only a `.env.example` with fake values is committed.
- **Separate profiles** (`dev`, `test`, `prod`) with different secrets. A development secret must never be used in production.
- **GitHub secret scanning** (or a tool such as gitleaks) is turned on, to catch accidental commits.
- **If a secret is ever committed,** treat it as leaked: change it straight away. Deleting it from the Git history is not enough.
- **No default admin password.** The first Administrator is created by a one-time setup step that reads its credentials from environment variables. There is no password written into migration scripts or code.
- **Production hardening:** Swagger UI and debug endpoints are disabled or protected in production.

---

## 12. Database Security

| Control | Detail |
|---|---|
| **Least-privilege database user** | The application connects as its own user, not as a superuser. It gets `SELECT`, `INSERT`, `UPDATE` on the tables it needs |
| **Audit table protected** | On `audit_logs` the application user gets only `INSERT` and `SELECT`. `UPDATE` and `DELETE` are revoked |
| **Separate migration user** | Flyway runs with a different account that can change the schema. The running application cannot |
| **Not publicly reachable** | The database accepts connections only from the application, through a private network or firewall rule |
| **Encrypted connection** | The application connects to PostgreSQL over TLS |
| **Constraints as a safety net** | Unique email, foreign keys with `ON DELETE RESTRICT`, `CHECK` constraints on statuses and hours |
| **No hard deletes** | Users are deactivated and records are kept, so history and the audit trail stay intact |
| **Backups** | Regular, encrypted, stored away from the database server, and restore-tested |
| **Encryption at rest** | Use disk or database encryption from the hosting provider |

---

## 13. Other Measures

- **HTTPS everywhere.** TLS ends at a reverse proxy or load balancer. HTTP is redirected to HTTPS and an HSTS header is sent.
- **CSRF protection is switched off on purpose.** CSRF attacks rely on browsers sending cookies automatically. This API uses a token in the `Authorization` header and no session cookies, so the attack does not apply.
- **CORS** allows only FimeBag's own front-end origins, not `*`.
- **Security headers** (such as `X-Content-Type-Options`) use Spring Security's defaults.
- **Dependency checks.** Dependabot or OWASP Dependency-Check flags libraries with known vulnerabilities.
- **Overall rate limiting** on the API, not only on login, is added at the reverse proxy.

---

## 14. How the Design Will Be Tested

| Check | What it proves |
|---|---|
| Call every endpoint with no token | Returns 401 everywhere except login |
| Call each endpoint with each role | Returns 403 where the permission matrix says no |
| Staff tries to approve a task or reach someone else's task | Refused |
| Expired token, wrong signature, `none` algorithm | Rejected |
| Deactivate a user, then reuse their old token | Refused straight away |
| Send `role`, `status` or `id` in a request body | Ignored |
| Send bad values in `sort` and filters | 400 |
| Check every response and log | No password hash, token or secret |
| Trigger an unexpected error | Generic 500, no stack trace |
| Try to `UPDATE` or `DELETE` in `audit_logs` as the application user | Database refuses |

---

## 15. Limits and Open Points

- **No logout or refresh token.** A stolen token works until it expires (at most 1 hour) unless the user is deactivated.
- **No password reset.** Users can change their own password only while they know the current one. If one forgets it, the current API has no reset flow, and an Administrator cannot set a new password through user update. A reset flow (Administrator-triggered or by email) should be added later.
- **Changing a password does not cancel tokens already issued.** They expire within 1 hour.
- **No multi-factor authentication.** It could be added later for Administrators.
- **Lock-out safeguard.** The requirements do not say that an Administrator may not deactivate their own account or change their own role. Adding this rule would prevent an accidental lock-out.
- **Audit log retention** (how long entries are kept) is not decided and needs agreement with FimeBag.
- **The failed-login counter** is held in memory, which is fine for one server. Several servers would need a shared store.
