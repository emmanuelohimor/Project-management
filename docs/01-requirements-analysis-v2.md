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
