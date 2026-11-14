# Tech

- Language: TypeScript, strict mode.
- Framework: Next.js with React.
- Package manager: npm with workspaces.
- Tests: Vitest for unit tests, fast-check for property-based tests.
- Storage in version 1: JSON files. Access goes through one storage
  interface so a database can replace it later.
- No paid services and no secret keys in the repo.

## Code rules
- Keep functions small and typed. No "any".
- Validate every config file against the config schema on load.
- Write tests with each feature, not after.
- Use plain, readable names.