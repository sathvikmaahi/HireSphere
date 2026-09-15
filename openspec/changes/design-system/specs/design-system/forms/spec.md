## Purpose

The core form and input components every data-entry screen in TalentSphere is assembled from, and
the field-plus-summary presentation of validation errors. Owned by `TS-BL-011`.

## ADDED Requirements

### Requirement: Core control set

The system SHALL provide text input, multi-line text, select, checkbox, radio group and toggle
controls, built from the defined color, spacing, radius and type tokens and sized to the defined
control height and padding steps.

*Source: `reference/design-spec.md` §4.3's size and padding steps and `reference/layout-spec.md`
§8's control-padding guidance. `UI-001`'s desktop-first framing sets the density.*

#### Scenario: Control built from tokens

- **WHEN** any control renders
- **THEN** every color, spacing, radius and type value resolves from an existing token

#### Scenario: Disabled control

- **WHEN** a control is disabled
- **THEN** it renders at the uniform reduced opacity and does not respond to pointer input

### Requirement: Field wrapper carries label, description and error

Every control SHALL be presentable inside a field wrapper carrying a label, an optional description
and an optional error message, with the label programmatically associated with its control and the
error announced to assistive technology.

*Source: `UI-009`'s labels and screen-reader semantics; `UI-007`'s field-level errors.*

#### Scenario: Labelled control

- **WHEN** a control renders inside a field wrapper
- **THEN** its label is programmatically associated with it

#### Scenario: Field in error

- **WHEN** a field carries an error
- **THEN** the message is displayed with the field and announced to assistive technology

### Requirement: Validation errors appear at field level and summary level

A form SHALL present validation errors both on each affected field and in a summary listing them
together.

*Source: `UI-007`, which requires both levels rather than either.*

#### Scenario: Form with multiple errors

- **WHEN** a submitted form fails validation on several fields
- **THEN** each field shows its own error
- **AND** a summary lists them together

#### Scenario: Summary navigates to a field

- **WHEN** a user activates an entry in the validation summary
- **THEN** focus moves to the corresponding field

#### Scenario: Errors cleared

- **WHEN** a previously invalid field becomes valid
- **THEN** both its field-level error and its summary entry are removed

### Requirement: Controls are keyboard-operable and visibly focused

Every control SHALL be fully operable by keyboard and SHALL present the defined focus indication
when focused.

*Source: `UI-009`; `reference/design-spec.md` §4.3's focus rule.*

#### Scenario: Keyboard-only form completion

- **WHEN** a user completes a form without a pointing device
- **THEN** every control can be reached, changed and submitted using the keyboard alone

#### Scenario: Focus indication

- **WHEN** a control receives keyboard focus
- **THEN** the focus indication is visible

### Requirement: Controls hold no submission or validation logic

Controls and the field wrapper SHALL render the value, error and change handler they are given.
They SHALL NOT perform submission, call an API, or derive validation rules themselves.

*Source: `design.md` D1. The validation rules belong to the feature that owns the record, and the
server is authoritative for all of them; a control that carried its own rules would place a second,
divergent copy on the client.*

#### Scenario: Value change

- **WHEN** a control's value changes
- **THEN** it invokes the change handler it was given rather than storing the value itself

#### Scenario: Server-side validation failure

- **WHEN** a submission is refused by the server with field errors
- **THEN** those errors render at field and summary level identically to client-detected ones
