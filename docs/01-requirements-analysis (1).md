# Requirements Analysis

**Project:** FimeBag Internal Task Management API
**Deliverable:** 1 of 9, Week 1

---

## 1. Users / Roles

### Administrator

The Administrator has organisation-wide authority over the system.

**Responsibilities:**

- Create and manage user accounts.
- Assign roles to users.
- Create and manage projects.
- Assign Project Managers to projects.
- View and oversee system activity.
- Have access to information across the system.

**Assumption:** The Administrator represents someone with organisation-wide authority, such as HR, the Managing Director, or a designated system administrator.

### Project Manager

A senior staff member responsible for managing one or more projects.

**Responsibilities:**

- View projects they are assigned to manage.
- Create tasks within their assigned projects.
- Assign tasks to staff.
- Monitor task progress.
- Review submitted tasks.
- Approve or reject completed work.
- Manage tasks within their assigned projects.

### Staff

Staff members, including interns and junior staff, who carry out assigned tasks.

**Responsibilities:**

- View tasks assigned to them.
- Work on their assigned tasks.
- Update their task status.
- Submit completed work for review.
- Record work performed against their tasks.

---

## 2. Functional Requirements

### User Management

- Create users.
- Update user information.
- Assign roles.
- Manage/deactivate user accounts.
- Authenticate users.
- Allow any user to change their own password.

### Project Management

- Create projects.
- View projects.
- Update projects.
- Track project status.
- Assign one or more Project Managers to a project.
- Record project objectives, dates and description.

### Task Management

- Create tasks.
- Associate tasks with projects.
- Assign tasks to staff.
- View tasks.
- Update task information.
- Track task status.
- Submit completed tasks for review.
- Approve or reject submitted tasks.

### Worklog

The system should allow staff to record work performed against tasks.

### Audit Trail

The system should record important activities so that authorised users can determine what happened and who performed an action.

### Authentication and Authorisation

- Users must authenticate before accessing protected resources.
- The system must enforce role-based permissions.
- Users should only be able to perform actions appropriate to their role.

---

## 3. Business Rules

### User rules

1. Every user has a role.
2. The system supports three roles: Administrator, Project Manager, and Staff.
3. Only authorised users can access protected resources.
4. Staff cannot modify another staff member's task unless specifically authorised.

### Project rules

5. A project must have at least one Project Manager.
6. A project can have multiple Project Managers.
7. A Project Manager can manage multiple projects.
8. A Project Manager may only manage tasks belonging to projects they are assigned to.
9. Project status can be:
   - Planned
   - Active
   - On Hold
   - Completed
10. A project can move from Planned → Active.
11. An active project may be placed On Hold and later returned to Active.
12. A project can eventually be marked Completed.

### Task rules

13. Every task must belong to a project.
14. A task may be assigned to a staff member.
15. A Project Manager can only create or assign tasks within their assigned projects.
16. A Staff member can only work on tasks assigned to them.
17. Staff members cannot directly mark their tasks as Done.
18. A Staff member can submit their completed work for review.
19. Submitted work changes the task status to Under Review.
20. Only an authorised Project Manager for that project can approve or reject a task.
21. Approval changes the task status to Done.
22. Rejection changes the task status back to In Progress so the staff member can continue working.
23. A completed task should not be moved backwards without an authorised action/rule.

---

## 4. Assumptions

These points were not fixed by the project brief, so the design assumes the following. The reasons behind them are in the [Technical Decision Log](09-technical-decision-log.md).

1. The Administrator represents someone with organisation-wide authority, such as HR, the Managing Director or a designated system administrator.
2. Users are created by an Administrator. There is no self-registration, and each user's email address is their unique login.
3. Each user has exactly one role.
4. A project has at least one Project Manager and may have several. A Project Manager can manage several projects.
5. A task belongs to one project and has at most one assignee, who must be an active Staff member.
6. Task priority is one of `LOW`, `MEDIUM` or `HIGH`.
7. Project status changes follow these steps: Planned → Active, Active ↔ On Hold, and Active → Completed.
8. The Administrator has read-only access to tasks and worklogs. Managing tasks belongs to Project Managers.
9. Staff see only their own tasks, the worklogs on them and the projects those tasks belong to. A Project Manager sees the projects they are assigned to, and their tasks and worklogs. The Administrator sees everything.
10. A completed (Done) task is final. No function reopens it (rule 23).
11. Worklogs cannot be edited or deleted once saved.
12. Only the latest review comment is kept on a task. A rejection reason is required, and the Staff member can read it.
13. Every logged-in user can change their own password. There is no password reset in the first release.
14. The system serves one organisation and is deployed once.

## 5. Open Questions

Questions that still need an answer from FimeBag, such as project length, deployment, reopening completed tasks and password reset, are listed in section 9 of the [Technical Decision Log](09-technical-decision-log.md).
