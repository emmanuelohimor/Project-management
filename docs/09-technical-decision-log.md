# Technical Decision Log

**Project:** FimeBag Internal Task Management API
**Deliverable:** 9 of 9, Week 1
**Date:** October 2026

This log records the important decisions made during the design, and **why** each one was made. Each entry also says what was rejected. The assumptions and open questions at the end are the points that still need confirmation from FimeBag.

Document references: 01 Requirements, 02 Use cases, 03 Database, 04 API, 05 Architecture, 06 Security, 08 Development plan.

---

## 1. Technology

| # | Decision | Choice | Reason |
|---|---|---|---|
| T-1 | Language | Java 21 (LTS) | Project requirement. A long-term-support version gets fixes for years, and Spring Boot 3 needs Java 17 or newer |
| T-2 | Framework | Spring Boot 3.x | Project requirement. It has ready-made support for web APIs, database access and security, so less code to write and fewer mistakes |
| T-3 | Database | PostgreSQL | The data is highly related (users, projects, tasks, worklogs), so a relational database fits. It enforces foreign keys, `CHECK` rules and unique values, which protect the data. *Rejected:* a NoSQL database, because it gives weaker guarantees for related data |
| T-4 | Data access | Spring Data JPA (Hibernate) | Tables map to Java classes, which removes a lot of repetitive SQL. *Rejected:* plain JDBC, which is more code for the same result |
| T-5 | Schema changes | Flyway migrations, with Hibernate set to *validate* only | Every database change is a reviewed script in Git, so the schema is versioned and repeatable. *Rejected:* letting Hibernate create tables, because changes would be invisible and unrepeatable |
| T-6 | Build tool | Maven | Standard for Spring Boot, with a simple fixed structure that is easy to learn. *Rejected:* Gradle, which is more flexible but has more to learn |

---

## 2. Data Design (Doc 03)

| # | Decision | Choice | Reason |
|---|---|---|---|
| D-1 | Primary keys | UUIDs | They cannot be guessed by counting upward, so `/tasks/{id}` cannot be enumerated. The cost is slightly larger indexes |
| D-2 | Roles | A `roles` table, one role per user (`users.role_id`) | The brief lists Role as an entity. A table lets roles change without code changes. One role per user is enough for the requirements and avoids a join table |
| D-3 | Project Managers | Junction table `project_managers` with key `(project_id, user_id)` | A project can have several managers and a manager can run several projects (many-to-many). A single column cannot express that |
| D-4 | Statuses | `VARCHAR` with a `CHECK` constraint | There are only a few fixed values. The database blocks bad values, while the order of changes is enforced in code. *Rejected:* lookup tables, which add joins for no benefit |
| D-5 | Deleting | Never delete. Users have `is_active`, projects end as `COMPLETED`. All foreign keys use `ON DELETE RESTRICT` | History, worklogs and the audit trail must survive |
| D-6 | Audit log reference | `audit_logs.entity_id` has no foreign key | One column points at different tables (users, projects, tasks), so a normal foreign key is impossible |
| D-7 | Who did the work | `worklogs.user_id` is stored on each worklog | If a task is reassigned, past hours must still show who did them |
| D-8 | Hours | `NUMERIC(4,2)` | Decimal numbers do not suffer the rounding errors of `FLOAT` |
| D-9 | Review feedback | A `review_comment` column on `tasks`, holding only the latest comment | Staff must be able to see why work was rejected. The audit log is Administrator-only, so the feedback has to be on the task. Each decision replaces the previous comment, and the audit log keeps the full history |

---

## 3. API Design (Doc 04)

| # | Decision | Choice | Reason |
|---|---|---|---|
| A-1 | Style and versioning | REST, with the version in the URL (`/api/v1`) | REST is the requirement, and users, projects and tasks map naturally to resources. A new version can be added later without breaking existing clients |
| A-2 | Task status changes | Separate action endpoints: `status` (start), `submit-review`, `approve`, `reject`. No general "set status" | A free status field would let Staff send `DONE` and skip the review. Separate endpoints make each rule enforceable |
| A-3 | Deactivation | `PATCH .../status` instead of `DELETE` | Records are kept (D-5), and a user can be reactivated |
| A-4 | Error codes | 400 malformed input, 422 valid input that breaks a business rule, 409 a clash with the current state (duplicate, or an invalid status change) | Clients can tell the difference between a typing mistake and a rule being refused |
| A-5 | Error format | One format for every error, with a stable `error` code | Predictable for clients. Spring's default errors would differ by case |
| A-6 | Pagination | `page`, `size` (maximum 100) and `sort` on all lists. Sort fields must be on a whitelist | Prevents huge responses, and blocks injection through the sort parameter |
| A-7 | Project creation | `managerIds` is required, so a project is never created without a manager | Rule 5: every project has at least one Project Manager |

---

## 4. Business Rules (Docs 01 and 02)

| # | Decision | Choice | Reason |
|---|---|---|---|
| B-1 | Roles | Administrator, Project Manager, Staff | Separates running the system, managing projects and doing the work, which supports least privilege |
| B-2 | Who marks work Done | Staff submit for review. Only the assigned Project Manager approves (Done) or rejects (back to In Progress) | Rules 17 to 22. It gives quality control and stops people approving their own work |
| B-3 | Completed tasks | Final. No endpoint moves a `DONE` task backwards | Rule 23 asks for an authorised action but does not define one. The safest choice is none for now. Reopening is listed as a later item |
| B-4 | Administrator and tasks | Read-only across the system | The requirements give the Administrator "access to information", while task management belongs to Project Managers (rules 8, 15). It keeps responsibilities separate |
| B-5 | Project status changes | Planned → Active, Active ↔ On Hold, Active → Completed | Rules 10 to 12. Completion is only allowed from Active so a project cannot be completed without having started |
| B-6 | Visibility | Administrator sees everything. A Project Manager sees their projects and those projects' tasks. Staff see their own tasks and the projects those belong to | Least privilege: people see only what they need |

---

## 5. Security (Doc 06)

| # | Decision | Choice | Reason |
|---|---|---|---|
| S-1 | Authentication | JWT, signed with HS256, valid for 60 minutes, sent as a Bearer token | Stateless, so no server sessions to manage. Short life limits the harm from a stolen token. One service signs and checks its own tokens, so a shared secret is enough |
| S-2 | Checking the user | The user is loaded from the database on every request, and the role and active status come from there, not the token | A deactivated user or changed role takes effect immediately. The cost is one quick lookup per request. *Rejected:* trusting the token until it expires |
| S-3 | Password storage | BCrypt (work factor 12), 8 to 72 characters | BCrypt is slow by design, so guessing is expensive. It ignores anything past 72 bytes, so the limit is explicit |
| S-4 | Authorisation | Two checks: the role (Spring Security method security), then the data (policy classes) | Roles alone cannot say "this manager belongs to *this* project". Policy classes keep that rule in one place |
| S-5 | Login protection | A generic error message, and a limit of 5 failed attempts in 15 minutes, answered with 429 | Stops guessing and stops attackers learning which emails exist |
| S-6 | Logout and refresh tokens | Not in this version | Short-lived tokens and the per-request check (S-2) give acceptable safety for a first release. Both are listed as later items |
| S-7 | CSRF protection | Switched off | The API uses a token in a header, not cookies, so the browser cannot be tricked into sending it automatically |
| S-8 | Secrets | Environment variables only. No default admin password. The first Administrator is created from environment values | Nothing secret ever enters Git, and no known password exists |
| S-9 | Database permissions | The application uses a limited database user. On `audit_logs` it may only insert and read. A separate user runs migrations | The audit trail cannot be changed even if the application is compromised |
| S-10 | Audit trail | Written by the service in the same transaction as the action. A failed login is saved separately | The audit log can never record something that did not happen. A failed login still needs recording even though it fails |

---

## 6. Architecture (Doc 05)

| # | Decision | Choice | Reason |
|---|---|---|---|
| R-1 | Overall shape | Layered (controller, service, repository), grouped by feature | Simple, well understood and easy to test, which suits one database and one developer. Grouping by feature keeps all task code together. *Rejected:* microservices (too complex for seven tables), hexagonal architecture (extra layers with no second database or integration to justify them) |
| R-2 | Data sent to clients | DTOs, never database entities | Stops fields such as `password_hash` leaking, and lets the API change without changing the tables |
| R-3 | Business rules | Policy classes (`ProjectAccessPolicy`, `TaskWorkflow`) | Each rule is written once and tested alone, instead of being copied between services |
| R-4 | Task workflow | An enum with a map of allowed changes | There are only four states. *Rejected:* a state-machine framework, which adds more complexity than it removes |
| R-5 | Auditing | Explicit calls to an `AuditService` | Clear to read, and records business meaning such as "approved". *Rejected:* AOP or database triggers, which are hidden and cannot record the reason |
| R-6 | Visibility filtering | Done in the database query | Page counts stay correct and nothing is loaded that the user may not see |

---

## 7. Process (Docs 07 and 08)

| # | Decision | Choice | Reason |
|---|---|---|---|
| P-1 | Branching | `main` (stable), `develop` (integration), and short-lived `docs/`, `feature/` and `fix/` branches. Everything merges by pull request | No one develops directly on `main`. Each change is reviewed, and the history shows how the work progressed |
| P-2 | Commit messages | `type: description` (for example `docs: add API design`) | A readable, searchable history |
| P-3 | Documentation | Markdown files in `/docs` in the same repository, with diagrams as images or Mermaid | Documents are versioned with the code and reviewed in the same way |
| P-4 | Planning | Dependency-based order with agile sprints (2 weeks, 7 build sprints) | Dependencies fix what can be built first, and sprints give regular demos and chances to adjust. *Rejected:* waterfall and building by layer, which show nothing working until the end |
| P-5 | Testing | Many unit tests for rules and policies, controller tests, repository tests on a real PostgreSQL (Testcontainers), and a few full-flow tests | The risk is in the business rules, and these tests check them quickly. A real PostgreSQL avoids surprises from a fake database |

---

## 8. Assumptions

The requirements document lists the same assumptions (section 4). Item 15 is a planning assumption only.

| # | Assumption |
|---|---|
| 1 | The Administrator represents someone with organisation-wide authority (for example HR, the Managing Director or a system administrator) |
| 2 | Users are created by an Administrator. There is no self-registration, and the email address is the unique login |
| 3 | Each user has one role |
| 4 | A project has at least one Project Manager and may have several. A Project Manager can manage several projects |
| 5 | A task belongs to one project and has at most one assignee, who must be an active Staff member |
| 6 | Task priority is one of `LOW`, `MEDIUM` or `HIGH` |
| 7 | Project status changes: Planned → Active, Active ↔ On Hold, Active → Completed |
| 8 | The Administrator has read-only access to tasks and worklogs. Managing tasks belongs to Project Managers |
| 9 | Staff see only their own tasks, the worklogs on them and their projects. A Project Manager sees their projects, tasks and worklogs. The Administrator sees everything |
| 10 | A completed task is final. No function reopens it |
| 11 | Worklogs cannot be edited or deleted once saved |
| 12 | Only the latest review comment is kept on a task |
| 13 | Every logged-in user can change their own password. There is no password reset in the first release |
| 14 | The system serves one organisation and is deployed once |
| 15 | One developer, 2-week sprints, and about 8 planned days of work per sprint |

---

## 9. Open Questions

| # | Question | Why it matters |
|---|---|---|
| 1 | How long is the project, and what sprint length should be used? | The plan assumes 2-week sprints, so about 14 weeks after Week 1 |
| 2 | Where will the application be deployed? | Affects Sprint 7 and the production configuration |
| 3 | Can a completed task ever be reopened, and by whom? | Rule 23 is not specific. The current design says no |
| 4 | Should the Administrator be able to create or change tasks, or only view them? | The design makes the Administrator read-only on tasks |
| 5 | Can a Project Manager create projects, or only the Administrator? | The requirements give this to the Administrator |
| 6 | Can a project go from On Hold straight to Completed? | The rules do not say, and the design says no |
| 7 | Are password reset and logout needed in the first release? | Neither is in the current design |
| 8 | How long should audit entries be kept, and does data protection law apply to them? | Audit entries contain names, user IDs and IP addresses |
| 9 | Will a front end use this API, and from which address? | It decides the allowed origins (CORS) and when the API documentation is needed |
| 10 | Should worklogs be editable if someone makes a mistake? | The design says no, so a mistaken entry stays |
