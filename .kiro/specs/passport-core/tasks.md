# Implementation Plan: Passport Core

## Overview

Implement the `skeleton/` npm workspace package as a config-driven, stateless TypeScript library. Work proceeds module by module in dependency order: types → config → passport → score → risk → storage → index. Unit tests follow each module; property-based tests are grouped in a final phase. Every domain function is pure; only `JsonStorage` has side effects.

## Tasks

- [ ] 1. Scaffold the skeleton workspace package
  - [ ] 1.1 Create `skeleton/` directory with `package.json` for the npm workspace package
    - Set `name`, `version`, `main`, `types`, `scripts` (build, test) pointing to Vitest
    - Add dependencies: `typescript`, `vitest`, `fast-check`
    - _Requirements: 7.5_
  - [ ] 1.2 Create `skeleton/tsconfig.json` with strict mode enabled
    - Enable `strict`, `noImplicitAny`, `noUncheckedIndexedAccess`, `moduleResolution: node16` (or `bundler`)
    - Set `rootDir: src`, `outDir: dist`
    - _Requirements: (structural — enables all type constraints)_
  - [ ] 1.3 Create the `skeleton/src/` directory structure (empty placeholder files)
    - Touch `types.ts`, `config.ts`, `passport.ts`, `score.ts`, `risk.ts`, `storage.ts`, `index.ts`
    - _Requirements: (structural)_

- [ ] 2. Implement shared types
  - [ ] 2.1 Write `skeleton/src/types.ts` with all shared type and interface definitions
    - Define `FieldType`, `FieldStatus`, `FieldDefinition`, `RiskCondition`, `RiskRule`, `IndustryConfig`
    - Define `ProofLink`, `FieldEntry`, `Passport`
    - Define `IssueType` union and `ValidationIssue`
    - No imports, no exports other than the type/interface declarations; export everything
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 3.1, 3.3, 3.7, 4.1, 6.1_

- [ ] 3. Implement the config module
  - [ ] 3.1 Implement `loadConfig` in `skeleton/src/config.ts`
    - Import types from `./types`
    - Validate presence of all four top-level keys; throw naming the missing key
    - Validate `name` length 1–100; throw on violation
    - Validate `fields` array: 1–200 entries; each entry has all five required sub-properties with valid values; no duplicate keys; throw on first violation naming key/index and property
    - Validate `proofDocumentTypes`: 0–50 strings, each 1–100 chars
    - Validate `riskRules`: 0–100 entries; each `condition.field` must be in the validated field set; throw naming the flag and unknown field
    - Return typed `IndustryConfig` on success
    - _Requirements: 1.1–1.8, 2.1–2.6, 6.6_
  - [ ]* 3.2 Write unit tests for `loadConfig` in `skeleton/src/config.test.ts`
    - One test per invalid input: each missing top-level key, name too short (empty), name too long (101 chars), missing field sub-property for each of the five props, invalid `type` value, duplicate field key, risk rule referencing unknown field, `proofDocumentTypes` entry too long
    - One test for a valid config returning the typed object
    - _Requirements: 2.2, 2.3, 2.4, 2.5, 2.6_

- [ ] 4. Implement the passport module
  - [ ] 4.1 Implement `effectiveStatus` in `skeleton/src/passport.ts`
    - Import `FieldEntry` and `FieldStatus` from `./types`
    - Implement the three-case rule: null value → `"missing"`, status=proven + empty proofLinks → `"claimed"`, otherwise return recorded status
    - Export `effectiveStatus`
    - _Requirements: 3.4, 3.5_
  - [ ] 4.2 Implement `parsePassport` in `skeleton/src/passport.ts`
    - Validate top-level shape: `productId` present and matches `/^[A-Za-z0-9_-]{1,100}$/`, `configName` is non-empty string, `fields` is a plain object
    - For each entry in `fields`: `value` is string or null, `status` is one of the three allowed values, `proofLinks` is an array of 0–100 entries
    - For each `ProofLink`: `url` non-empty ≤2048 chars, `label` non-empty ≤256 chars, `documentType` non-empty string
    - Throw synchronously with a message naming the offending path (e.g. `"fields.material.proofLinks[0].url: must be a non-empty string"`)
    - Return typed `Passport` on success
    - Export `parsePassport`
    - _Requirements: 3.1, 3.3, 3.7, 7.14_
  - [ ] 4.3 Implement `validatePassport` in `skeleton/src/passport.ts`
    - Iterate all Config fields; for each compute effective status (use `effectiveStatus`, treat absent passport fields as missing)
    - Collect issues in one pass (not fail-fast): `missing-required`, `proof-required`, `proof-status-conflict`, `value-status-conflict`, `invalid-value` (with type-format checks for `number`, `date`, `url`), `unknown-proof-type`, `invalid-proof-link`
    - After config fields, iterate passport-only fields and emit `unknown-field`
    - Return `ValidationIssue[]`
    - Export `validatePassport`
    - _Requirements: 4.1–4.12_
  - [ ]* 4.4 Write unit tests for `effectiveStatus`, `parsePassport`, and `validatePassport` in `skeleton/src/passport.test.ts`
    - `effectiveStatus`: test all three branches including the proven-with-empty-links case
    - `parsePassport`: test a valid passport returns correctly; test each structural violation throws with the correct path in the message
    - `validatePassport`: one test per `issueType` to confirm the right issue is returned; test that all issues are collected in a single pass for a multi-violation passport; test empty list for a fully valid passport; test absent config field treated as missing
    - _Requirements: 3.4, 3.5, 4.1–4.12_

- [ ] 5. Checkpoint — config and passport modules
  - Ensure all existing tests pass before proceeding. Ask the user if any questions arise.

- [ ] 6. Implement the score module
  - [ ] 6.1 Implement `computeScore` in `skeleton/src/score.ts`
    - Import `Passport`, `IndustryConfig` from `./types`; import `effectiveStatus` from `./passport`
    - Iterate `config.fields`; look up each entry in `passport.fields` (absent = missing)
    - Required weight = 2, optional weight = 1; proven = full weight, claimed = 0.5×, missing = 0
    - `score = total === 0 ? 0 : Math.floor(100 * earned / total)`
    - Return integer in [0, 100]
    - Export `computeScore`
    - _Requirements: 5.1–5.8_
  - [ ]* 6.2 Write unit tests for `computeScore` in `skeleton/src/score.test.ts`
    - All fields proven → 100; all fields missing → 0; no fields in config → 0
    - Mixed passport with known inputs and computed expected value (verify floor, not round)
    - Required vs optional weight difference (required field proven vs optional field proven)
    - _Requirements: 5.1–5.8_

- [ ] 7. Implement the risk module
  - [ ] 7.1 Implement `evaluateRisks` in `skeleton/src/risk.ts`
    - Import `Passport`, `IndustryConfig` from `./types`; import `effectiveStatus` from `./passport`
    - Iterate `config.riskRules`; for each rule evaluate condition using `effectiveStatus`
    - `fieldMissing`: effective status === `"missing"`; `fieldNotProven`: effective status !== `"proven"`
    - Collect `rule.flag` for all satisfied conditions, deduplicate, return `string[]`
    - Export `evaluateRisks`
    - _Requirements: 6.1–6.5_
  - [ ]* 7.2 Write unit tests for `evaluateRisks` in `skeleton/src/risk.test.ts`
    - One test per predicate: `fieldMissing` fires when field is missing; `fieldNotProven` fires when field is claimed or missing; neither fires when field is proven
    - Deduplication test: two rules raising the same flag → one entry in the result
    - Empty result test: no rules satisfied → empty list
    - _Requirements: 6.2–6.5_

- [ ] 8. Checkpoint — score and risk modules
  - Ensure all existing tests pass before proceeding. Ask the user if any questions arise.

- [ ] 9. Implement the storage module
  - [ ] 9.1 Define `StorageInterface` in `skeleton/src/storage.ts`
    - Import `Passport` from `./types`
    - Export the `StorageInterface` interface with `save`, `load`, `list` signatures
    - _Requirements: 7.1–7.3_
  - [ ] 9.2 Implement `JsonStorage` class in `skeleton/src/storage.ts`
    - Constructor accepts `directory: string`
    - `save`: validate `productId` regex, check path confinement, mkdir recursively if needed, write to `<productId>.tmp` then rename to `<productId>.json`
    - `load`: validate `productId`, check path confinement, read file (return null on ENOENT), parse JSON, call `parsePassport`, return `Passport`; reject with descriptive error on parse failure or invalid shape
    - `list`: read directory, filter filenames ending in `.json`, strip suffix, keep only stems matching `/^[A-Za-z0-9_-]{1,100}$/`, return stems
    - Import `parsePassport` from `./passport`
    - _Requirements: 7.4–7.13_
  - [ ]* 9.3 Write unit tests for `JsonStorage` in `skeleton/src/storage.test.ts`
    - Save then load round-trip using a `tmp` directory: assert deep equality
    - Overwrite: save twice with same `productId`, load returns the second value
    - Load missing → null
    - Invalid `productId` on save → rejected Promise, no file created
    - Invalid `productId` on load → rejected Promise
    - Corrupt JSON file → rejected Promise with descriptive error
    - Atomic write: verify no `.json` file appears mid-write (check temp file naming)
    - _Requirements: 7.4–7.14_

- [ ] 10. Wire the public entry point
  - [ ] 10.1 Write `skeleton/src/index.ts` re-exporting everything
    - Re-export from `./config`: `loadConfig`
    - Re-export from `./passport`: `parsePassport`, `effectiveStatus`, `validatePassport`
    - Re-export from `./score`: `computeScore`
    - Re-export from `./risk`: `evaluateRisks`
    - Re-export from `./storage`: `StorageInterface`, `JsonStorage`
    - Re-export all types from `./types`
    - No new logic; purely re-exports
    - _Requirements: 7.14 (parsePassport publicly exported)_

- [ ] 11. Checkpoint — full module wiring
  - Build the TypeScript project (`tsc --noEmit`) and run all unit tests. Ensure everything compiles and all tests pass. Ask the user if any questions arise.

- [ ] 12. Property-based tests — scoring invariants
  - [ ]* 12.1 Write property test for Property 2 (score range) in `skeleton/src/score.property.test.ts`
    - Tag: `// Feature: passport-core, Property 2: Score range`
    - Use `fc.record` for arbitrary `Passport` and `IndustryConfig`; assert `0 <= result <= 100` and `Number.isInteger(result)`
    - Minimum 100 iterations
    - _Requirements: 5.1_
  - [ ]* 12.2 Write property test for Property 3 (score determinism) in `skeleton/src/score.property.test.ts`
    - Tag: `// Feature: passport-core, Property 3: Score determinism`
    - Use `fc.clone` to get two identical inputs; assert both calls return the same integer
    - _Requirements: 5.8 (5.9)_
  - [ ]* 12.3 Write property test for Property 4 (monotonicity: missing → claimed) in `skeleton/src/score.property.test.ts`
    - Tag: `// Feature: passport-core, Property 4: Monotonicity — missing to claimed`
    - Generate passport + config where at least one config field has effective status `"missing"`; record score; set entry to `{ value: "x", status: "claimed", proofLinks: [] }`; assert new score ≥ original
    - _Requirements: 5.10_
  - [ ]* 12.4 Write property test for Property 5 (monotonicity: claimed → proven) in `skeleton/src/score.property.test.ts`
    - Tag: `// Feature: passport-core, Property 5: Monotonicity — claimed to proven`
    - Generate passport + config where at least one field has effective status `"claimed"`; upgrade to proven with one proof link; assert new score ≥ original
    - _Requirements: 5.11_
  - [ ]* 12.5 Write property test for Property 7 (score 100 iff all proven) in `skeleton/src/score.property.test.ts`
    - Tag: `// Feature: passport-core, Property 7: Score is 100 iff every Config field is proven`
    - Compute `allProven` flag; assert `computeScore === 100` iff `allProven`
    - _Requirements: 5.5_

- [ ] 13. Property-based tests — config and passport invariants
  - [ ]* 13.1 Write property test for Property 1 (config round-trip) in `skeleton/src/config.property.test.ts`
    - Tag: `// Feature: passport-core, Property 1: Config validation round-trip`
    - Arbitrary valid `IndustryConfig` via `fc.record`; call `loadConfig(JSON.parse(JSON.stringify(config)))`; assert field count and per-index `key`/`type`/`label` match
    - _Requirements: 2.7_
  - [ ]* 13.2 Write property test for Property 8 (validatePassport never throws) in `skeleton/src/passport.property.test.ts`
    - Tag: `// Feature: passport-core, Property 8: validatePassport never throws for any parsed Passport`
    - Arbitrary structurally valid `Passport` and `IndustryConfig`; wrap `validatePassport` in try/catch; assert no exception and return value is an array
    - _Requirements: 4.1, 4.11_
  - [ ]* 13.3 Write property test for Property 11 (parsePassport round-trip) in `skeleton/src/passport.property.test.ts`
    - Tag: `// Feature: passport-core, Property 11: parsePassport round-trip`
    - Arbitrary valid `Passport`; assert `parsePassport(JSON.parse(JSON.stringify(p)))` deeply equals `p`
    - _Requirements: 3.1_

- [ ] 14. Property-based tests — risk and storage invariants
  - [ ]* 14.1 Write property test for Property 9 (fieldMissing ⊆ fieldNotProven) in `skeleton/src/risk.property.test.ts`
    - Tag: `// Feature: passport-core, Property 9: fieldMissing flags are a subset of fieldNotProven flags`
    - Config where each field has at least one `fieldMissing` rule and one `fieldNotProven` rule; for each field where `fieldMissing` fired, assert `fieldNotProven` also fired
    - _Requirements: 6.2, 6.3_
  - [ ]* 14.2 Write property test for Property 6 (storage round-trip) in `skeleton/src/storage.property.test.ts`
    - Tag: `// Feature: passport-core, Property 6: Storage round-trip`
    - Arbitrary valid `Passport`; write via `JsonStorage.save`, read via `JsonStorage.load`; assert deep equality with Vitest `toStrictEqual`; use a per-test `tmp` directory and clean up after
    - _Requirements: 7.8, 7.9_
  - [ ]* 14.3 Write property test for Property 10 (JsonStorage rejects invalid productIds) in `skeleton/src/storage.property.test.ts`
    - Tag: `// Feature: passport-core, Property 10: JsonStorage rejects invalid productIds with no file created`
    - Arbitrary string failing `/^[A-Za-z0-9_-]{1,100}$/`; assert `save` rejects; assert no `.json` or `.tmp` file created in directory
    - _Requirements: 7.6_

- [ ] 15. Final checkpoint — all tests pass
  - Run `tsc --noEmit` and the full test suite (`vitest --run`). Ensure compilation succeeds and all unit and property tests pass. Ask the user if any questions arise.

## Notes

- Sub-tasks marked `*` are optional and can be skipped for a faster MVP, but are strongly recommended to validate the correctness properties defined in the design.
- Each property test must be tagged with `// Feature: passport-core, Property N: <property text>` and run a minimum of 100 iterations.
- `effectiveStatus` lives in `passport.ts` and is imported by both `score.ts` and `risk.ts` — do not duplicate it.
- `parsePassport` is both used internally by `JsonStorage.load` and exported publicly via `index.ts`.
- No industry-specific vocabulary is allowed in any `skeleton/` source file.
- TypeScript strict mode is mandatory; `any` is not permitted.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2", "1.3"] },
    { "id": 1, "tasks": ["2.1"] },
    { "id": 2, "tasks": ["3.1"] },
    { "id": 3, "tasks": ["3.2", "4.1"] },
    { "id": 4, "tasks": ["4.2", "4.3"] },
    { "id": 5, "tasks": ["4.4", "6.1"] },
    { "id": 6, "tasks": ["6.2", "7.1"] },
    { "id": 7, "tasks": ["7.2", "9.1"] },
    { "id": 8, "tasks": ["9.2"] },
    { "id": 9, "tasks": ["9.3", "10.1"] },
    { "id": 10, "tasks": ["12.1", "12.2", "12.3", "12.4", "12.5", "13.1", "13.2", "13.3", "14.1", "14.2", "14.3"] }
  ]
}
```
