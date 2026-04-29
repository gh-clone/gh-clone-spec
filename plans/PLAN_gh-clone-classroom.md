# Plan for gh-clone-classroom

### Tier 1: Organizations, Rosters & LTI Integrations
- [ ] Define the `classroom_organizations` Postgres table: `id`, `org_id`, `title`, `created_at`.
- [ ] Define the `classroom_rosters` Postgres table: `id`, `classroom_id`, `identifier_type` (Email, Student ID), `identifier`.
- [ ] Build the Angular UI for teachers to import Student Rosters via CSV, implementing in-browser parsing (`PapaParse`) and column validation before submitting to the Rust API.
- [ ] Implement the Student Linkage authentication flow: students click a secure invitation link, authenticate via GitHub OAuth, and explicitly map their GitHub Username to their Student ID on the roster.
- [ ] Implement the LTI (Learning Tools Interoperability) 1.3 core specification in Rust, managing OpenID Connect (OIDC) handshakes and JWT signing to embed the classroom as a secure external tool.
- [ ] Support LTI Deep Linking (Content Item Message), allowing teachers to browse and create new assignments directly from within their Canvas, Blackboard, or Moodle dashboards.
- [ ] Implement the LTI Names and Role Provisioning Services (NRPS) to automatically sync and update the Classroom Roster when students add or drop the course in the LMS.

### Tier 2: Assignment Workflows & Git Automation
- [ ] Define the `classroom_assignments` table: `id`, `classroom_id`, `title`, `deadline`, `template_repo_id`, `visibility`, `is_group_assignment`.
- [ ] Architect the "Accept Assignment" background worker in Rust: intercept the student's acceptance and automatically generate a new private repository inside the Classroom Organization.
- [ ] Execute `git push --mirror` programmatically within the worker to copy the exact Git tree, commit history, and starter code from the teacher's `template_repo_id` to the student's repository.
- [ ] Append the student's GitHub username to the generated repository name automatically (e.g., `data-structures-hw1-samuel`).
- [ ] Enforce strict Role-Based Access Control (RBAC): grant the student `Write` access to their specific generated repository, but grant the Teacher and Teaching Assistants `Admin` access globally.
- [ ] Build Group Assignments logic: allow students to form or join teams during the acceptance flow, provisioning a single repository shared via a dynamically created GitHub Team.
- [ ] Implement repository archiving: provide an Admin UI button to safely archive all student repositories for a specific assignment once the semester concludes.

### Tier 3: Auto-Grading & Continuous Integration
- [ ] Define the `classroom_autograding_tests` Postgres schema, storing exact runner configurations: `name`, `setup_command`, `run_command`, `input`, `expected_output`, `points`, `timeout`.
- [ ] Build a custom GitHub Actions Composite Action (`classroom/autograding-action`) that securely executes the teacher's test definitions against the student's pushed code.
- [ ] Parse the structured JSON output of the autograding action and POST the scores directly to the internal Rust `POST /classrooms/{id}/assignments/{assign_id}/grades` API using a scoped runner token.
- [ ] Render the "Autograding Dashboard" in Angular, giving teachers a sortable data grid of all students, their latest commit statuses, and their final calculated scores.
- [ ] Implement LMS Gradebook Synchronization: utilize the LTI Advantage Assignment and Grade Services (AGS) specification to push the student's final score automatically into the Canvas/Moodle gradebook.
- [ ] Build the "Feedback Pull Request" mechanism: optionally generate a permanent, open PR in the student's repository containing the starter code diff, allowing teachers to leave line-level code review comments.
- [ ] Enforce "Deadline Locks": implement a `tokio` cron worker that automatically revokes student `Write` access (downgrading to `Read`) precisely at the assignment's configured deadline timestamp.

### Tier 4: Anti-Plagiarism & Analytics
- [ ] Integrate with external code similarity tools (e.g., MOSS - Measure of Software Similarity): build a worker to extract all student repositories for an assignment and submit them via the MOSS protocol.
- [ ] Parse the MOSS similarity reports and render highlighted, side-by-side plagiarism warnings directly in the Teacher's Angular dashboard.
- [ ] Track detailed commit analytics per student: export time-to-first-commit and commit frequency metrics to help teachers identify struggling students early.
- [ ] Implement "Teaching Assistant" roles: extend the standard organization RBAC to allow TAs to view rosters and grade assignments without having destructive Admin permissions over the organization.

### Tier 5: Extensions & SSO Identity Mapping
- [ ] Build the "Student Developer Pack" API link: automatically grant students access to GitHub Pro tier features and internal compute credits once they link an `.edu` email address.
- [ ] Implement the "Deadline Extension" feature: provide an Angular form for the Teacher to modify the `deadline` exclusively for a subset of students via the Postgres `classroom_extensions` table.
- [ ] Map enterprise SAML SSO identities to the Classroom rosters natively: if the school uses Azure AD, use SCIM synchronization to build the initial student roster automatically.
- [ ] Design an immutable grading audit log: track every manual override of an auto-graded score by a Teaching Assistant.
