## Purpose

Defines the single authorization model for TalentSphere: how role grants, direct grants, explicit denials, and data scopes combine into one deny-by-default verdict, and how that verdict is enforced and explained.

## ADDED Requirements

### Requirement: Deny-by-default authorization

The system SHALL deny every page and action unless access is granted by role, by direct assignment, or by an approved exception.

#### Scenario: No applicable grant

- **WHEN** a user requests a page or action for which no role grant, direct grant, or exception exists
- **THEN** access is denied

#### Scenario: Newly introduced page

- **WHEN** a new page or action is added to the permission catalog
- **THEN** it is denied to every role until explicitly granted

### Requirement: Nine action flags

The permission model SHALL support at least the action flags View, Create, Edit, Delete, Approve, Run AI, Export, Assign, and Administer, evaluated per page.

#### Scenario: Run AI is independently controllable

- **WHEN** a user holds View on a page but not Run AI
- **THEN** the user may read the page and is denied any action that triggers an AI task

#### Scenario: Approve is independently controllable

- **WHEN** a user holds Edit but not Approve on a page
- **THEN** the user may modify content and is denied the approval action

### Requirement: Grant, deny, and unset cell states

Each permission assignment SHALL be expressible as grant, deny, or unset, where unset means "no opinion" and contributes nothing to the verdict.

#### Scenario: Unset does not grant

- **WHEN** a role holds unset for an action and no other grant applies
- **THEN** access is denied

#### Scenario: Unset does not deny

- **WHEN** one of a user's roles holds unset for an action and another role holds grant
- **THEN** access is granted

### Requirement: Explicit denial overrides grants

An explicit denial SHALL override any role-based or direct grant, at both role and user level.

#### Scenario: Direct denial beats role grant

- **WHEN** a user's role grants an action and a direct user-level denial exists for that same action
- **THEN** access is denied

#### Scenario: Denial in one of several roles

- **WHEN** a user holds two roles, one granting and one explicitly denying the same action
- **THEN** access is denied

### Requirement: Server-side enforcement on every surface

Every page and every API endpoint SHALL enforce authorization server-side through the centralized permission evaluator. Direct API calls SHALL fail when the user lacks the required permission, even when the corresponding UI control is hidden.

#### Scenario: Hidden control called directly

- **WHEN** a user without Edit permission calls the update endpoint directly, bypassing the UI
- **THEN** the request is rejected with an authorization error
- **AND** no partial write occurs

#### Scenario: Endpoint added without an authorization decision

- **WHEN** an endpoint is exposed without declaring a required permission
- **THEN** it denies all requests rather than allowing them

### Requirement: Permission explanation

The system SHALL expose, to authorized administrators, an explanation of any permission verdict identifying the contributing role grants, direct grants, and explicit denials.

#### Scenario: Explaining a denial

- **WHEN** an administrator asks why a specific user cannot access a specific page and action
- **THEN** the response identifies the deciding factor, including which denial overrode which grant, and which roles were consulted

#### Scenario: Explaining a grant

- **WHEN** an administrator asks why a user does have access
- **THEN** the response names the role or direct assignment that granted it

### Requirement: Per-user permission overrides

The system SHALL support direct per-user grants and denials that operate independently of role membership. Every override SHALL require a reason.

#### Scenario: Override without a reason

- **WHEN** an administrator creates a user-level override with no reason supplied
- **THEN** the request is rejected

#### Scenario: Override survives role change

- **WHEN** a user's roles are reassigned
- **THEN** existing direct overrides continue to apply until explicitly removed

### Requirement: Seeded role set

The system SHALL seed nine roles: System Administrator, Application Administrator, Practice Manager, Recruitment Manager, Recruiter, Interviewer, Hiring Panel Member, Auditor or Compliance Reviewer, and AI Service Account.

#### Scenario: Seeded roles are protected

- **WHEN** a user attempts to delete a seeded system role
- **THEN** the request is rejected while the role remains editable in its permission assignments

#### Scenario: Auditor cannot modify recruitment decisions

- **WHEN** the Auditor role is evaluated for Create, Edit, Delete, Approve, or Assign on any recruitment resource
- **THEN** access is denied by seeded configuration

### Requirement: Least-privilege defaults

Recruiters, Interviewers, and Hiring Panel Members SHALL receive least-privilege access by default.

#### Scenario: Freshly seeded interviewer

- **WHEN** a user is granted only the Interviewer role with no additional configuration
- **THEN** the user receives no administrative, export, or approval capability

### Requirement: Data scope predicates

Beyond page and action flags, the authorization model SHALL carry a data scope per grant. Interviewer access SHALL be scoped to that user's own assigned interviews. Write access to posting-bound resources SHALL be scoped to postings the user is assigned to or authorized to manage, while read scope SHALL be governed by configuration rather than hard-coded.

#### Scenario: Interviewer scope predicate is defined

- **WHEN** the Interviewer role's grants are inspected
- **THEN** each carries an assignment-scoped predicate, so the role can never resolve to all interviews regardless of page grants

#### Scenario: Write scope on an unassigned posting

- **WHEN** a user with a posting-scoped write grant attempts a write against a posting they are not assigned to
- **THEN** the request is denied even though the page-level action is granted

#### Scenario: Read scope is configuration, not code

- **WHEN** an administrator changes the configured read scope for a role
- **THEN** the evaluator's read verdicts change accordingly with no code deployment

### Requirement: Service account authorization

The AI Service Account SHALL be authorized as a first-class actor with controlled service permissions, and SHALL NOT hold permissions that constitute a human accountability decision.

#### Scenario: Service account cannot approve

- **WHEN** the AI Service Account is evaluated for the Approve action on any page
- **THEN** access is denied and cannot be granted through the matrix

#### Scenario: Service account acts on background work

- **WHEN** a background AI task runs under the service account
- **THEN** its permission verdicts are evaluated by the same evaluator as human requests

### Requirement: Audited permission changes

Every permission change SHALL be audited with actor, subject, previous value, new value, timestamp, reason, and the affected page and action.

#### Scenario: Matrix cell changed

- **WHEN** an administrator changes a matrix cell from unset to grant
- **THEN** an audit record captures both the previous and the new value along with the affected page and action

### Requirement: Immediate effect of authorization changes

A permission or role change SHALL take effect for the affected user's subsequent requests without requiring re-authentication, and SHALL NOT be served from a stale cached verdict.

#### Scenario: Access removed mid-session

- **WHEN** an administrator revokes a permission from a user holding an active session
- **THEN** that user's next request for the affected action is denied
