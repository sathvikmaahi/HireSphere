## Purpose

The one designed path from an Administrator's configuration-only default to candidate personal
data: a time-boxed, reasoned, notified elevation that expires on its own. It is an ordinary
permission override with an expiry, deliberately not a separate access mode. Owned by `TS-BL-025`.

## ADDED Requirements

### Requirement: Administrators are configuration-only on candidate data

Administrator roles SHALL NOT hold View permission on candidate personal data by default. Access to
candidate personal data by an administrator SHALL require an explicit, time-boxed, audited
elevation.

*Source: `D16` — config only, with audited break-glass, chosen over unrestricted access with
after-the-fact detection and over a hard block that makes real support work impossible. Inherited
from `talentsphere-wave-1-foundation`'s `access-control/admin-cockpit`; relocated here because D.9
makes break-glass its own item (`design.md` D2).*

#### Scenario: Administrator opens a candidate record

- **WHEN** an Application Administrator requests candidate personal data without an active
  elevation
- **THEN** access is denied

#### Scenario: Administrator opens a configuration surface

- **WHEN** an Application Administrator requests a configuration surface
- **THEN** access is granted without any elevation

### Requirement: Break-glass is an override with an expiry, not a second access mode

Elevation SHALL be represented as a user-level permission override carrying an expiry, evaluated by
the same evaluator and explained by the same explanation surface as any other grant. No parallel
elevated mode or alternative authorization path SHALL exist.

*Source: `design.md` D10, inherited from `talentsphere-wave-1-foundation`'s D5. Reusing the override
mechanism means break-glass inherits the audit trail, the explanation surface and the evaluator; a
parallel elevated mode would be a second authorization path — the thing `design.md` D1 exists to
prevent — and would give an administrator two different answers to "why can this person see this".*

#### Scenario: Elevation appears in the explanation

- **WHEN** an administrator asks why an elevated user can view candidate personal data
- **THEN** the explanation names the elevation as the grant responsible, alongside its expiry

#### Scenario: Authorization paths enumerated

- **WHEN** the paths by which access to candidate personal data can be obtained are enumerated
- **THEN** each resolves through the single evaluator
- **AND** no elevated-mode bypass exists

### Requirement: Elevation requires a typed reason

An elevation request SHALL require a typed reason. A request without one SHALL be rejected and no
elevation SHALL be created.

*Source: `D16`'s "explicit logged elevation";
`talentsphere-wave-1-foundation`'s inherited scenario "Elevation without a reason".*

#### Scenario: Elevation without a reason

- **WHEN** an administrator requests elevation with no reason supplied
- **THEN** the request is rejected with a field-level validation error and no elevation is created

#### Scenario: Reason accompanies the elevation

- **WHEN** an elevation is granted
- **THEN** its reason is carried on the elevation record and on its audit record

### Requirement: Elevation is bounded and expires without administrator action

An elevation SHALL carry a bounded duration and SHALL cease to grant access when that duration
elapses, without requiring any administrator action to end it. The default duration SHALL be
changeable through the audited configuration path rather than by deployment.

*Source: `design.md` D10 — time-boxed rather than per-request, because per-request approval needs a
second human available on demand and real support work does not have one. Expiry bounds exposure
without adding a synchronous dependency on another person.*

#### Scenario: Elevation expires

- **WHEN** an elevation's duration elapses
- **THEN** the elevated access ceases on the next request, with no administrator action taken

#### Scenario: Elevation cannot be open-ended

- **WHEN** an elevation is requested without a duration or with an unbounded one
- **THEN** the request is rejected

#### Scenario: Duration changed as configuration

- **WHEN** the default elevation duration is changed through the audited configuration path
- **THEN** subsequent elevations use the new duration without a deployment

### Requirement: Elevation is notified to a second party

Granting an elevation SHALL notify the configured recipients.

*Source: `D16`'s "break-glass needs a notification target"; `talentsphere-wave-1-foundation`'s D5
and its inherited scenario. The recipient set is configuration — provisionally all other
Application Administrators plus the Auditor role, pending an owner decision recorded in
`design.md` Open Questions. Delivery is `platform-core`'s notification engine.*

#### Scenario: Elevation granted

- **WHEN** an elevation is granted
- **THEN** the configured recipients are notified, identifying the elevated user, the reason, and
  the expiry

#### Scenario: Recipient set is configurable

- **WHEN** the configured notification recipients are changed
- **THEN** subsequent elevations notify the new recipients without a deployment

#### Scenario: Notification failure does not silently drop

- **WHEN** notification delivery fails
- **THEN** the failure is retained and surfaced rather than discarded

### Requirement: Elevation grants only what was requested

An elevation SHALL grant only the specific permissions its request names, and SHALL NOT confer
broader access than the administrator's configuration-only default plus those permissions.

*Source: `AUTHZ-008` least privilege; `D16`'s rejection of unrestricted access. An elevation that
quietly conferred everything would reproduce option 2 with extra ceremony.*

#### Scenario: Scope of the elevation

- **WHEN** an elevated administrator requests data outside the permissions the elevation names
- **THEN** access is denied

#### Scenario: Elevation does not confer approval

- **WHEN** an elevated administrator attempts an Approve action
- **THEN** access is denied, elevation notwithstanding

### Requirement: Elevation can be revoked before it expires

An authorized administrator SHALL be able to end an active elevation before its expiry, taking
effect on the subject's next request.

*Source: `AUTHZ-007`'s immediate effect and `config.yaml`'s uncached-evaluator rule. Expiry bounds
the normal case; revocation is what closes an elevation granted in error, and an elevation that can
only be waited out is one nobody can correct.*

#### Scenario: Elevation revoked

- **WHEN** an administrator ends an active elevation
- **THEN** the elevated access ceases on the subject's next request

#### Scenario: Revoking an already-expired elevation

- **WHEN** an administrator ends an elevation that has already expired
- **THEN** the operation succeeds without error
