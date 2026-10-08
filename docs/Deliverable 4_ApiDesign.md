# REST API Design

**Project:** FimeBag Internal Task Management API
**Deliverable:** 4 of 9, Week 1

---

## 1. Design Principles

### Base URL and versioning

```text
/api/v1
```

Version 1 is used from the start so that future breaking changes can be introduced as `/api/v2` without breaking existing clients. All endpoint paths in this document are relative to `/api/v1`.

### Authentication

Protected endpoints require:

```http
Authorization: Bearer <JWT>
```

The login endpoint does not require authentication.

### Role permissions

| Role | Can do |
|---|---|
| **Administrator** | Manage users and roles; create and manage projects; assign Project Managers; view projects, tasks and worklogs across the system (read-only); view the audit log |
| **Project Manager** | View projects they are assigned to; create, update and assign tasks in those projects; review (approve or reject) submitted tasks; view worklogs |
| **Staff** | View their assigned tasks and the related project information; start work on, and submit, their own tasks; record worklogs on their own tasks |

Task management belongs to Project Managers (requirements rules 8 and 15). The Administrator can see tasks and worklogs but cannot create, edit, assign or review them.

### Naming conventions

- **Plural nouns** for resources: `/users`, `/projects`, `/tasks`.
- **Path parameters** for individual resources: `/projects/{projectId}`.
- **HTTP methods** indicate the operation.
- Lowercase resource names, with hyphens separating words where needed (`/audit-logs`).
- Workflow actions that are not simple updates are modelled as `POST` sub-resources (`/tasks/{taskId}/approve`).

```http
GET   /api/v1/projects
GET   /api/v1/projects/{projectId}
POST  /api/v1/projects
PATCH /api/v1/projects/{projectId}
```

---

## 2. Endpoint Summary

| Method | Endpoint | Main actor | Purpose |
|---|---|---|---|
| POST | `/auth/login` | All | Authenticate |
| POST | `/users` | Admin | Create user |
| GET | `/users` | Admin | List users |
| GET | `/users/{id}` | Admin | Get user |
| PATCH | `/users/{id}` | Admin | Update user |
| PATCH | `/users/{id}/status` | Admin | Activate/deactivate |
| POST | `/projects` | Admin | Create project |
| GET | `/projects` | All | List accessible projects |
| GET | `/projects/{id}` | All | Get project |
| PATCH | `/projects/{id}` | Admin | Update project |
| PATCH | `/projects/{id}/status` | Admin | Change project status |
| POST | `/projects/{id}/managers` | Admin | Assign manager |
| GET | `/projects/{id}/managers` | Admin/PM | View managers |
| DELETE | `/projects/{id}/managers/{userId}` | Admin | Remove manager |
| POST | `/tasks` | PM | Create task |
| GET | `/tasks` | All | List accessible tasks |
| GET | `/tasks/{id}` | All | Get task |
| PATCH | `/tasks/{id}` | PM | Update task |
| POST | `/tasks/{id}/assign` | PM | Assign task |
| PATCH | `/tasks/{id}/status` | Staff | Update work status |
| POST | `/tasks/{id}/submit-review` | Staff | Submit for review |
| POST | `/tasks/{id}/approve` | PM | Approve task |
| POST | `/tasks/{id}/reject` | PM | Reject task |
| POST | `/tasks/{id}/worklogs` | Staff | Record work |
| GET | `/tasks/{id}/worklogs` | PM/Staff/Admin | View worklogs |
| GET | `/audit-logs` | Admin | View audit trail |

---

## 3. Authentication Endpoints

### POST `/api/v1/auth/login`

**Purpose:** Authenticate a user and issue a JWT.

**Authentication:** None.

**Request:**

```json
{
  "email": "john@example.com",
  "password": "password123"
}
```

**Success: `200 OK`**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "user": {
    "id": "uuid",
    "firstName": "John",
    "lastName": "Doe",
    "role": "STAFF"
  }
}
```

**Errors:**

- `400 Bad Request`: missing or invalid input
- `401 Unauthorized`: invalid email or password
- `403 Forbidden`: account inactive

---

## 4. User Endpoints

Only the **Administrator** can manage users.

### POST `/api/v1/users`

**Purpose:** Create a user account.

**Authentication:** Administrator.

**Request:**

```json
{
  "firstName": "John",
  "lastName": "Doe",
  "email": "john@example.com",
  "password": "temporaryPassword",
  "role": "STAFF"
}
```

**Success:** `201 Created`

**Errors:**

- `400 Bad Request`: validation failure
- `401 Unauthorized`
- `403 Forbidden`
- `409 Conflict`: email already exists

### GET `/api/v1/users`

**Purpose:** Retrieve users.

**Authentication:** Administrator.

**Query parameters:**

```text
?page=0&size=20&role=STAFF&active=true
```

**Success: `200 OK`**

```json
{
  "content": [
    {
      "id": "uuid",
      "firstName": "John",
      "lastName": "Doe",
      "email": "john@example.com",
      "role": "STAFF",
      "active": true
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 45,
  "totalPages": 3
}
```

**Errors:**

- `400 Bad Request`: invalid pagination or filter
- `401 Unauthorized`
- `403 Forbidden`

### GET `/api/v1/users/{userId}`

**Purpose:** Retrieve a specific user.

**Authentication:** Administrator.

**Success:** `200 OK`

**Errors:**

- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`

### PATCH `/api/v1/users/{userId}`

**Purpose:** Update user information or role.

**Authentication:** Administrator.

**Request:**

```json
{
  "firstName": "John",
  "lastName": "Smith",
  "role": "PROJECT_MANAGER"
}
```

**Success:** `200 OK`

**Errors:**

- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `409 Conflict`: email already exists

### PATCH `/api/v1/users/{userId}/status`

**Purpose:** Activate or deactivate a user account.

**Authentication:** Administrator.

**Request:**

```json
{
  "active": false
}
```

**Success:** `200 OK`

**Errors:**

- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`

There is no `DELETE /users/{id}`. Users are deactivated rather than permanently deleted, which preserves historical records and audit information.

---

## 5. Project Endpoints

### POST `/api/v1/projects`

**Purpose:** Create a project.

**Authentication:** Administrator.

**Request:**

```json
{
  "name": "Company Website Redesign",
  "description": "Redesign the company website.",
  "objectives": "Improve performance, usability and SEO.",
  "startDate": "2026-10-10",
  "endDate": "2026-12-20",
  "managerIds": ["uuid"]
}
```

A project must have at least one Project Manager (rule 5), so `managerIds` is required. Every ID must belong to an active user with the `PROJECT_MANAGER` role. The initial status is `PLANNED`.

**Success:** `201 Created`

**Errors:**

- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `422 Unprocessable Entity`: invalid date relationship, or a manager that does not exist or is not an active Project Manager

### GET `/api/v1/projects`

**Purpose:** Retrieve projects accessible to the authenticated user.

**Authentication:** Required.

**Query:**

```text
?page=0&size=20&status=ACTIVE
```

Visibility depends on role:

- Administrator: all projects
- Project Manager: projects they manage
- Staff: projects containing their assigned tasks

**Success:** `200 OK`

**Errors:**

- `400 Bad Request`
- `401 Unauthorized`

### GET `/api/v1/projects/{projectId}`

**Purpose:** Retrieve a specific project.

**Authentication:** Required.

**Success: `200 OK`**

```json
{
  "id": "uuid",
  "name": "Company Website Redesign",
  "description": "Redesign the company website.",
  "objectives": "Improve performance and usability.",
  "startDate": "2026-10-10",
  "endDate": "2026-12-20",
  "status": "ACTIVE",
  "projectManagers": [
    {
      "id": "uuid",
      "name": "Sarah Doe"
    }
  ]
}
```

**Errors:**

- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`

### PATCH `/api/v1/projects/{projectId}`

**Purpose:** Update project information.

**Authentication:** Administrator.

**Request:**

```json
{
  "name": "Updated Project Name",
  "description": "Updated description",
  "objectives": "Updated objectives",
  "endDate": "2026-12-31"
}
```

**Success:** `200 OK`

**Errors:**

- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `422 Unprocessable Entity`

### PATCH `/api/v1/projects/{projectId}/status`

**Purpose:** Change project status.

**Authentication:** Administrator.

**Request:**

```json
{
  "status": "ON_HOLD"
}
```

Valid statuses: `PLANNED`, `ACTIVE`, `ON_HOLD`, `COMPLETED`.

Allowed transitions (rules 10 to 12):

```text
PLANNED → ACTIVE
ACTIVE  → ON_HOLD → ACTIVE
ACTIVE  → COMPLETED
```

**Success:** `200 OK`

**Errors:**

- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `409 Conflict`: invalid status transition

---

## 6. Project Manager Assignment

### POST `/api/v1/projects/{projectId}/managers`

**Purpose:** Assign a Project Manager to a project.

**Authentication:** Administrator.

**Request:**

```json
{
  "userId": "uuid"
}
```

The API verifies that the selected user is an active user with the `PROJECT_MANAGER` role.

**Success:** `201 Created`

**Errors:**

- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `409 Conflict`: manager already assigned
- `422 Unprocessable Entity`: user is not an active Project Manager

### GET `/api/v1/projects/{projectId}/managers`

**Purpose:** View the Project Managers assigned to a project.

**Authentication:** Administrator, or a Project Manager assigned to that project.

**Success:** `200 OK`

**Errors:**

- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`

### DELETE `/api/v1/projects/{projectId}/managers/{userId}`

**Purpose:** Remove a Project Manager from a project.

**Authentication:** Administrator.

**Success:** `204 No Content`

**Errors:**

- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `409 Conflict`: attempting to remove the project's last manager (a project must always have at least one Project Manager)

---

## 7. Task Endpoints

### POST `/api/v1/tasks`

**Purpose:** Create a task within a project.

**Authentication:** Project Manager assigned to that project.

**Request:**

```json
{
  "projectId": "uuid",
  "title": "Create login page",
  "description": "Build the frontend login page.",
  "assigneeId": "uuid",
  "priority": "HIGH",
  "dueDate": "2026-10-20"
}
```

The initial status is `PENDING`.

**Success:** `201 Created`

**Errors:**

- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `422 Unprocessable Entity`: assignee does not exist, is inactive, or is not a Staff member

### GET `/api/v1/tasks`

**Purpose:** Retrieve tasks accessible to the authenticated user.

**Authentication:** Required.

Visibility depends on role:

- Administrator: all tasks (read-only)
- Project Manager: tasks in the projects they manage
- Staff: tasks assigned to them

**Query parameters:**

```text
?page=0&size=20
&projectId=uuid
&status=IN_PROGRESS
&assigneeId=uuid
&priority=HIGH
```

**Success:** `200 OK`

**Errors:**

- `400 Bad Request`: invalid pagination or filter
- `401 Unauthorized`

### GET `/api/v1/tasks/{taskId}`

**Purpose:** Retrieve a specific task.

**Authentication:** Required.

**Success:** `200 OK`

**Errors:**

- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`

### PATCH `/api/v1/tasks/{taskId}`

**Purpose:** Update task information.

**Authentication:** Project Manager assigned to the task's project.

**Request:**

```json
{
  "title": "Updated task title",
  "description": "Updated description",
  "priority": "MEDIUM",
  "dueDate": "2026-10-25"
}
```

**Success:** `200 OK`

**Errors:**

- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`

### POST `/api/v1/tasks/{taskId}/assign`

**Purpose:** Assign or reassign a task to a Staff member.

**Authentication:** Project Manager assigned to the task's project.

**Request:**

```json
{
  "userId": "uuid"
}
```

The API verifies that the selected user is an active Staff member (rule 14).

**Success:** `200 OK`

**Errors:**

- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `422 Unprocessable Entity`: user does not exist, is inactive, or is not a Staff member

---

## 8. Task Status Workflow

Clients cannot set an arbitrary status. Each step of the workflow is its own operation, so that business rules can be enforced:

```text
PENDING → IN_PROGRESS → UNDER_REVIEW → DONE
                ↑              │
                └── (reject) ──┘
```

A Staff member cannot send `{ "status": "DONE" }` to bypass the Project Manager's review (rule 17).

**Completed tasks are final (rule 23).** No endpoint moves a `DONE` task backwards, and `PATCH /tasks/{id}` does not accept a status. If reopening is needed later, it would be added as a dedicated, authorised action with a mandatory reason.

**Comments and reasons.** The `tasks` table has no column for review feedback, so the `comment` sent with `submit-review` and `approve`, and the `reason` sent with `reject`, are recorded in the audit log entry for that action (`audit_logs.details`), where the Administrator can read them.

### PATCH `/api/v1/tasks/{taskId}/status`

**Purpose:** Allow an assigned Staff member to start working on a task.

**Authentication:** Assigned Staff member.

**Request:**

```json
{
  "status": "IN_PROGRESS"
}
```

Allowed transition: `PENDING → IN_PROGRESS`.

**Success:** `200 OK`

**Errors:**

- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `409 Conflict`: invalid status transition

### POST `/api/v1/tasks/{taskId}/submit-review`

**Purpose:** The Staff member indicates that their work is finished and submits the task for Project Manager review.

**Authentication:** Assigned Staff member.

**Request:**

```json
{
  "comment": "Completed the login page and tested the validation."
}
```

**Success:** `200 OK`. Task becomes `IN_PROGRESS → UNDER_REVIEW`.

**Errors:**

- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `409 Conflict`: task is not currently in progress

### POST `/api/v1/tasks/{taskId}/approve`

**Purpose:** Approve a task submitted for review.

**Authentication:** Project Manager assigned to the task's project.

**Request:**

```json
{
  "comment": "Reviewed and approved."
}
```

**Success:** `200 OK`. Task becomes `UNDER_REVIEW → DONE`, and `completed_at` is set.

**Errors:**

- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `409 Conflict`: task is not Under Review

### POST `/api/v1/tasks/{taskId}/reject`

**Purpose:** Reject a submitted task and return it to the Staff member for further work.

**Authentication:** Project Manager assigned to the task's project.

**Request:**

```json
{
  "reason": "The form validation is incomplete."
}
```

**Success:** `200 OK`. Task becomes `UNDER_REVIEW → IN_PROGRESS`.

**Errors:**

- `400 Bad Request`: rejection reason missing
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `409 Conflict`: task is not Under Review

---

## 9. Worklog Endpoints

### POST `/api/v1/tasks/{taskId}/worklogs`

**Purpose:** Record work performed against a task.

**Authentication:** Assigned Staff member only.

**Request:**

```json
{
  "description": "Implemented and tested the login form.",
  "hoursSpent": 3.5,
  "workDate": "2026-10-08"
}
```

**Success:** `201 Created`

**Errors:**

- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `422 Unprocessable Entity`: invalid hours or date

### GET `/api/v1/tasks/{taskId}/worklogs`

**Purpose:** View worklogs associated with a task.

**Authentication:** Administrator, a Project Manager assigned to the project, or the Staff member assigned to the task.

**Success:** `200 OK`

**Errors:**

- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`

---

## 10. Audit Log Endpoints

### GET `/api/v1/audit-logs`

**Purpose:** Allow the Administrator to review important system activity.

**Authentication:** Administrator only.

**Query parameters:**

```text
?page=0
&size=20
&userId=uuid
&action=TASK_APPROVED
&entityType=TASK
&from=2026-10-01
&to=2026-10-08
```

**Success:** `200 OK`

**Errors:**

- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`

There are deliberately no `POST`, `PUT` or `DELETE` endpoints for audit logs. Audit records are generated by the application when important actions happen, and cannot be altered through the API.

---

## 11. Standard Error Response

All errors use one consistent format.

```json
{
  "timestamp": "2026-10-08T12:30:00Z",
  "status": 400,
  "error": "VALIDATION_ERROR",
  "message": "Request validation failed.",
  "path": "/api/v1/tasks",
  "details": [
    {
      "field": "title",
      "message": "Title is required"
    },
    {
      "field": "dueDate",
      "message": "Due date cannot be in the past"
    }
  ]
}
```

---

## 12. HTTP Status Codes

| Status | Meaning |
|---|---|
| `200 OK` | Successful GET, update or action |
| `201 Created` | Resource successfully created |
| `204 No Content` | Successful operation with no response body |
| `400 Bad Request` | Invalid request or validation failure |
| `401 Unauthorized` | Not authenticated, or invalid token |
| `403 Forbidden` | Authenticated but lacks permission |
| `404 Not Found` | Resource does not exist |
| `409 Conflict` | Operation conflicts with the current state |
| `422 Unprocessable Entity` | Valid request structure, but business validation fails |
| `500 Internal Server Error` | Unexpected server error |

---

## 13. Pagination

Collection endpoints (`GET /users`, `/projects`, `/tasks`, `/audit-logs`) support:

```text
?page=0&size=20
```

Optional sorting:

```text
?sort=createdAt,desc
```

Example:

```http
GET /api/v1/tasks?page=0&size=20&sort=createdAt,desc
```

A maximum page size is enforced (`size ≤ 100`) so that a client cannot request thousands of records in one call.

---

## 14. Validation

Requests are validated before reaching the service layer.

**User**

- Email required, valid format, and unique
- Password must meet minimum requirements
- Role must be one of the supported roles

**Project**

- Name required
- At least one Project Manager required on creation
- Start date cannot be after end date
- Status must be one of the supported statuses, and changes must follow the allowed transitions

**Task**

- Title required
- Project must exist
- Assignee must exist and be an active Staff member
- Due date must be valid
- Priority must be `LOW`, `MEDIUM` or `HIGH`
- Status transitions must follow the defined workflow

**Worklog**

- Task must exist
- Description required
- Hours must be greater than zero, with at most 2 decimal places
- Work date must be valid

---

## 15. Business-Rule Enforcement

The API never relies on the frontend to enforce permissions. For example, for `POST /api/v1/tasks/123/approve` the backend verifies that:

1. The user is authenticated.
2. The user is a Project Manager.
3. That Project Manager is assigned to the project containing task 123.
4. Task 123 exists.
5. Task 123 is currently `UNDER_REVIEW`.

Only then is the task approved.

Likewise, a Staff member cannot bypass the workflow by sending `PATCH /api/v1/tasks/123` with `{ "status": "DONE" }`. The backend rejects it, because `PATCH /tasks/{id}` does not accept a status.

---

## 16. Use Case Coverage

| Actor | Use case (from the use case diagram) | Endpoint(s) |
|---|---|---|
| All | Authenticate / Login | `POST /auth/login` |
| Administrator | Manage Users | `POST/GET /users`, `GET/PATCH /users/{id}`, `PATCH /users/{id}/status` |
| Administrator | Create Project | `POST /projects` |
| Administrator | Manage Projects | `PATCH /projects/{id}`, `PATCH /projects/{id}/status` |
| Administrator | Assign Project Managers | `POST/GET/DELETE /projects/{id}/managers` |
| Administrator | View Audit Log | `GET /audit-logs` |
| Project Manager | View Assigned Projects | `GET /projects`, `GET /projects/{id}` |
| Project Manager | Manage Tasks | `POST /tasks`, `PATCH /tasks/{id}` |
| Project Manager | Assign Tasks | `POST /tasks/{id}/assign` |
| Project Manager | Review Submitted Tasks | `POST /tasks/{id}/approve`, `POST /tasks/{id}/reject` |
| Project Manager | View Worklogs | `GET /tasks/{id}/worklogs` |
| Staff | View Assigned Tasks | `GET /tasks`, `GET /tasks/{id}` |
| Staff | Update Task Status | `PATCH /tasks/{id}/status` |
| Staff | Submit Task for Review | `POST /tasks/{id}/submit-review` |
| Staff | Record Worklog | `POST /tasks/{id}/worklogs` |
| Staff | View Task/Project Information | `GET /tasks`, `GET /projects` (limited to their own tasks) |

---

## 17. API Structure

```text
/api/v1
│
├── /auth
│   └── POST /login
│
├── /users
│   ├── POST /
│   ├── GET /
│   ├── GET /{id}
│   ├── PATCH /{id}
│   └── PATCH /{id}/status
│
├── /projects
│   ├── POST /
│   ├── GET /
│   ├── GET /{id}
│   ├── PATCH /{id}
│   ├── PATCH /{id}/status
│   ├── POST /{id}/managers
│   ├── GET /{id}/managers
│   └── DELETE /{id}/managers/{userId}
│
├── /tasks
│   ├── POST /
│   ├── GET /
│   ├── GET /{id}
│   ├── PATCH /{id}
│   ├── POST /{id}/assign
│   ├── PATCH /{id}/status
│   ├── POST /{id}/submit-review
│   ├── POST /{id}/approve
│   ├── POST /{id}/reject
│   └── /{id}/worklogs
│       ├── POST /
│       └── GET /
│
└── /audit-logs
    └── GET /
```
