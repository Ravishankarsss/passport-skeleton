# Requirements Document

## Introduction

Passport Core is the config-driven engine that lives in `skeleton/`. A single industry config file describes fields, proof document types, and risk rules. The engine turns that config into a working passport system — validating configs on load, enforcing field statuses, computing readiness scores, evaluating risk rules, and persisting products through a storage interface. No industry-specific words appear in skeleton code; every industry variation is expressed solely through its config.

## Glossary

- **Config**: A file that describes one industry — its fields, accepted proof document types, and risk rules.
- **Field**: One fact about a product, identified by a key and carrying a label, type, required/optional flag, and a proof-required flag.
- **FieldStatus**: The status of a field value on a product record: `proven`, `claimed`, or `missing`.
- **ProofLink**: A reference to a document that supports a field value. Contains a url, a label, and a documentType matching one of the Config's accepted proof document types.
- **Passport**: The full record for one product: a map of field keys to values, statuses, and proof links.
- **ReadinessScore**: An integer from 0 to 100 representing how proven the fields declared in a Config are for a Passport. Proven fields earn full weight, claimed fields earn half weight, missing fields earn zero. The score is computed using floor (not round), so it reaches 100 only when every Config field is proven.
- **EffectiveStatus**: The status used by the Engine for scoring and risk-rule evaluation. For a FieldEntry with `status` `proven` but no proof links, the effective status is `claimed`. For a FieldEntry with a null `value`, the effective status is `missing`. Otherwise the effective status equals the recorded `status`.
- **RiskRule**: A declarative rule in a Config that produces a named flag when its condition is met on a Passport.
- **RiskFlag**: A named warning produced when a RiskRule's condition is satisfied.
- **Engine**: The skeleton module that processes a Config and operates on Passports.
- **StorageInterface**: The abstract contract for saving, loading, and listing Passports.
- **JsonStorage**: The concrete implementation of StorageInterface that persists Passports as JSON files.
- **ValidationIssue**: A structured error or warning returned when a Passport does not conform to its Config. The full set of `issueType` values includes: `"missing-required"`, `"proof-required"`, `"proof-status-conflict"`, `"value-status-conflict"`, `"unknown-field"`, `"unknown-proof-type"`, `"invalid-value"`, and `"invalid-proof-link"`.

---

## Requirements

### Requirement 1: Config Format

**User Story:** As a developer building an industry app, I want to define my industry in a single config file, so that the skeleton engine can power the app without any code changes.

#### Acceptance Criteria

1. THE Config SHALL contain a `name` string that identifies the industry, with a length between 1 and 100 characters.
2. THE Config SHALL contain a `fields` array of 1 to 200 entries where each entry specifies a `key` (unique identifier string of 1 to 50 characters), a `label` (human-readable string of 1 to 100 characters), a `type` (one of `text`, `number`, `date`, `url`), a `required` boolean, and a `proofRequired` boolean.
3. THE Config SHALL contain a `proofDocumentTypes` array of 0 to 50 strings, where each string is 1 to 100 characters, naming the accepted categories of proof document for the industry.
4. THE Config SHALL contain a `riskRules` array of 0 to 100 entries (which MAY be empty) where each entry specifies a rule as defined in Requirement 6.
5. THE Engine SHALL treat two fields with the same `key` within one Config as a format violation.
6. IF the Config contains a `name` string of length 0 or exceeding 100 characters, THEN THE Engine SHALL reject the Config with an error indicating the `name` field is invalid.
7. IF the Config contains a `fields` array entry with a `key` of length 0 or exceeding 50 characters, THEN THE Engine SHALL reject the Config with an error indicating which entry has an invalid `key`.
8. IF the Config omits any of the required top-level properties (`name`, `fields`, `proofDocumentTypes`, `riskRules`), THEN THE Engine SHALL reject the Config with an error indicating which required property is missing.

---

### Requirement 2: Config Validation

**User Story:** As a developer, I want the engine to reject an invalid config with a clear error, so that misconfiguration is caught at load time rather than at runtime.

#### Acceptance Criteria

1. WHEN a Config is loaded, THE Engine SHALL validate all criteria below before any further processing, reporting the first validation error encountered and halting further validation.
2. IF a Config is missing any of the required top-level properties (`name`, `fields`, `proofDocumentTypes`, `riskRules`), THEN THE Engine SHALL throw an error that names the missing property.
3. IF a Config field entry is missing any of the required field-level properties (`key`, `label`, `type`, `required`, `proofRequired`), THEN THE Engine SHALL throw an error that names the field key (or its zero-based index if the key is absent) and the missing property.
4. IF a Config field entry contains a `type` value not in `{text, number, date, url}`, THEN THE Engine SHALL throw an error that names the field key and the invalid type value.
5. IF a Config contains duplicate field keys, THEN THE Engine SHALL throw an error that names the duplicated key.
6. WHEN a Config passes all schema checks, THE Engine SHALL return the validated Config object to the caller.
7. THE Engine SHALL ensure that for any structurally valid Config, validating, then serialising, then deserialising, then validating again produces a Config object with the same number of field entries and identical `key`, `type`, and `label` values for each field at the same index.

---

### Requirement 3: Product Record (Passport)

**User Story:** As a developer, I want to construct a Passport that records a value, a status, and optional proof links for each field, so that the completeness of a product's data is always explicit.

#### Acceptance Criteria

1. THE Passport SHALL contain a `productId` string consisting only of letters (A–Z, a–z), digits (0–9), hyphens (-), and underscores (_), with a length between 1 and 100 characters, that uniquely identifies the product within a single Config's namespace.
2. THE Passport SHALL contain a `configName` string matching the `name` of the Config it was created under.
3. THE Passport SHALL contain a `fields` map from field key to a `FieldEntry` that holds a `value` (string or null), a `status` whose value is one of `proven`, `claimed`, or `missing`, and a `proofLinks` array of ProofLink objects containing between 0 and 100 entries.
4. IF a FieldEntry has `status` equal to `proven` but has an empty `proofLinks` array, its **effective status** for scoring and risk evaluation is `claimed`.
5. IF a FieldEntry has `value` equal to null, its **effective status** for scoring and risk evaluation is `missing`, regardless of the recorded `status`.
6. IF a FieldEntry has `value` equal to null and `status` not equal to `missing`, or has `status` equal to `proven` with an empty `proofLinks` array, THE Engine SHALL record the inconsistency but SHALL NOT reject Passport construction.
7. THE Passport SHALL require each ProofLink to contain a non-empty `url` string of at most 2048 characters that is an absolute URL with protocol `http:` or `https:` only, a non-empty `label` string of at most 256 characters, and a non-empty `documentType` string.

---

### Requirement 4: Passport Validation

**User Story:** As a developer, I want to validate a Passport against its Config and receive a list of issues, so that incomplete or inconsistent data is surfaced before a Passport is published.

#### Acceptance Criteria

1. WHEN a Passport is validated against a Config, THE Engine SHALL return a list of ValidationIssue objects, each containing at minimum a `fieldKey` string property and an `issueType` string property, where the list MAY be empty if no issues are found.
2. WHEN a required field (as declared in the Config) has `status` equal to `missing` on the Passport, THE Engine SHALL include a ValidationIssue with `fieldKey` set to that field's key and `issueType` set to `"missing-required"`.
3. WHEN a field declared in the Config is not present in the Passport's `fields` map, THE Engine SHALL treat that field as having effective status `missing` for all validation checks.
4. WHEN a field declared with `proofRequired` equal to `true` in the Config has a `status` other than `"proven"` on the Passport, THE Engine SHALL include a ValidationIssue with `fieldKey` set to that field's key and `issueType` set to `"proof-required"`.
5. WHEN a FieldEntry carries `status` equal to `"proven"` but has an empty `proofLinks` array, THE Engine SHALL include a ValidationIssue with `fieldKey` set to that field's key and `issueType` set to `"proof-status-conflict"`.
6. WHEN a FieldEntry has `value` equal to null and `status` not equal to `missing`, THE Engine SHALL include a ValidationIssue with `fieldKey` set to that field's key and `issueType` set to `'value-status-conflict'`.
7. WHEN a Passport contains a field key not present in the Config, THE Engine SHALL include a ValidationIssue with `fieldKey` set to that unknown key and `issueType` set to `"unknown-field"`.
8. WHEN a ProofLink on a Passport has a `documentType` value that is not present in the Config's `proofDocumentTypes` array, THE Engine SHALL include a ValidationIssue with `fieldKey` set to the field's key and `issueType` set to `'unknown-proof-type'`.
9. WHEN a FieldEntry has a non-null `value` that does not conform to its declared field `type` — where `number` requires the value to match `/^-?\d+(\.\d+)?$/` exactly, `date` requires a string matching `YYYY-MM-DD` that also survives a round-trip check: parsing the string to a date and formatting it back as `YYYY-MM-DD` must produce the original string, thereby rejecting calendar-impossible dates such as `2026-02-31`, and `url` requires a string that is an absolute URL with protocol `http:` or `https:` only — THE Engine SHALL include a ValidationIssue with `fieldKey` set to that field's key and `issueType` set to `'invalid-value'`. Fields of type `text` are not type-checked.
10. WHEN a ProofLink on a Passport has a `url` that is not an absolute URL with protocol `http:` or `https:`, THE Engine SHALL include a ValidationIssue with `fieldKey` set to the field's key and `issueType` set to `'invalid-proof-link'`.
11. THE Engine SHALL return all ValidationIssues found in a single pass, not stop at the first issue.
12. THE Engine SHALL produce exactly one ValidationIssue per violated rule per field key, such that a single field violating multiple rules (e.g., both `missing-required` and `proof-required`) produces one ValidationIssue for each applicable rule.

---

### Requirement 5: Readiness Score

**User Story:** As a developer, I want a numeric readiness score from 0 to 100 for a Passport, so that the completeness of any product record can be communicated at a glance.

#### Acceptance Criteria

1. WHEN a ReadinessScore is computed for a Passport and Config, THE Engine SHALL return an integer in the closed range [0, 100].
2. THE Engine SHALL compute the ReadinessScore based solely on the fields declared in the Config, iterating over Config fields (not Passport fields). A Config field absent from the Passport is treated as having effective status `missing`.
3. FOR each Config field, THE Engine SHALL assign an effective score contribution: a field with effective status `proven` contributes its full weight; a field with effective status `claimed` contributes half its weight; a field with effective status `missing` contributes zero weight.
4. THE Engine SHALL assign each required field a weight at least twice the weight assigned to each optional field when computing the ReadinessScore.
5. WHEN every field declared in the Config has effective status `proven`, THE Engine SHALL return a ReadinessScore of 100.
6. THE Engine SHALL use `Math.floor` (truncation toward zero) rather than rounding when converting the weighted ratio to an integer, so that a ReadinessScore of 100 is returned **only when** every Config field has effective status `proven`.
7. WHEN every field declared in the Config has effective status `missing`, THE Engine SHALL return a ReadinessScore of 0.
8. IF a Config declares no fields, THE Engine SHALL return a ReadinessScore of 0.
9. WHEN a Passport and a Config are identical on two separate calls, THE Engine SHALL return the same ReadinessScore both times (determinism property).
10. WHEN a field's effective status changes from `missing` to `claimed`, THE Engine SHALL return a ReadinessScore greater than or equal to the ReadinessScore before that change (monotonicity — missing to claimed).
11. WHEN a field's effective status changes from `claimed` to `proven`, THE Engine SHALL return a ReadinessScore greater than or equal to the ReadinessScore before that change (monotonicity — claimed to proven).

---

### Requirement 6: Risk Rules

**User Story:** As an industry config author, I want to declare simple risk rules in the config, so that common data-quality problems are flagged automatically without writing code.

#### Acceptance Criteria

1. THE Config SHALL support RiskRule entries where each entry has a `flag` string (the name of the RiskFlag to raise) and a `condition` object.
2. THE RiskRule `condition` SHALL support the predicate `fieldMissing`, which takes a `field` argument whose value is a field key defined in the Config, and is satisfied when that field's effective status is `missing` on the evaluated Passport.
3. THE RiskRule `condition` SHALL support the predicate `fieldNotProven`, which takes a `field` argument whose value is a field key defined in the Config, and is satisfied when that field's effective status is not `proven` on the evaluated Passport.
4. WHEN a Passport is evaluated against a Config, THE Engine SHALL return a deduplicated list of RiskFlag names for every RiskRule whose condition is satisfied, where each name appears at most once regardless of how many rules raise it.
5. WHEN no RiskRule condition is satisfied, THE Engine SHALL return an empty list of RiskFlags.
6. IF a RiskRule `condition` references a field key that is not declared in the Config, THEN THE Engine SHALL reject the Config with a validation error at Config load time before any Passport evaluation is performed.

---

### Requirement 7: Storage Interface

**User Story:** As a developer, I want to save, load, and list Passports through a single interface, so that the storage backend can be swapped without changing any other skeleton code.

#### Acceptance Criteria

1. THE StorageInterface SHALL define a `save(passport: Passport): Promise<void>` operation that persists a Passport by its `productId`.
2. THE StorageInterface SHALL define a `load(productId: string): Promise<Passport | null>` operation that returns the Passport for the given id, or null if no Passport with that id exists.
3. THE StorageInterface SHALL define a `list(): Promise<string[]>` operation that returns only the `productId` values of Passports stored by `JsonStorage`; THE JsonStorage implementation SHALL return only stems of files whose name matches the productId pattern `/^[A-Za-z0-9_-]{1,100}\.json$/`, excluding temp files and any other stray files.
4. WHEN `save` is called with a `productId` that already exists, THE StorageInterface SHALL overwrite the existing record.
5. THE JsonStorage SHALL implement StorageInterface by reading and writing one JSON file per Passport in a configurable directory; IF the configured directory does not exist, THEN THE JsonStorage SHALL create it before the first write operation.
6. IF `save` or `load` is called with a `productId` that does not conform to the character and length constraints defined in Requirement 3, THEN THE JsonStorage SHALL reject the operation with an error before performing any file I/O.
7. THE JsonStorage SHALL construct file paths by joining the configured directory with the `productId` as a filename; THE JsonStorage SHALL never read from or write to any path outside the configured directory.
8. WHEN JsonStorage writes a Passport, THE JsonStorage SHALL serialise every field of the Passport to JSON such that a subsequent `load` for the same `productId` returns a Passport whose fields are deeply equal to the saved Passport.
9. WHEN `save` is called followed by `load` for the same `productId`, THE JsonStorage SHALL return a Passport object whose every field value is strictly equal to the corresponding field value of the saved Passport.
10. WHEN `load` is called with a `productId` for which no file exists, THE JsonStorage SHALL return null.
11. IF a `save` operation fails due to an I/O error, THEN THE JsonStorage SHALL reject the returned Promise with an error indicating the write failure, and any previously stored Passport for that `productId` SHALL remain unchanged.
12. WHEN `JsonStorage.save` performs an atomic write, THE JsonStorage SHALL use a temp filename that does not end in `.json` (e.g., `<productId>.tmp`) so that partially written files are never returned by `list`.
13. IF `load` encounters a file whose content cannot be parsed as a valid Passport, THEN THE JsonStorage SHALL reject the returned Promise with an error indicating the corrupt or invalid record, using `parsePassport` internally to validate the parsed JSON.
14. THE StorageInterface package SHALL export a `parsePassport(raw: unknown): Passport` function that validates the full runtime shape of a Passport and throws a descriptive error naming the offending field path if any structural constraint is violated; apps MAY call this function on form or file input before passing data to validation or scoring functions.
