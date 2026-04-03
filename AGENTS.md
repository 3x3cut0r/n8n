# AGENTS.md

## Purpose
- This repository is not a conventional application codebase.
- It primarily contains exported `n8n` workflows in `workflows/`, a Docker Compose deployment in `docker/`, small docs in `docs/`, and data-table exports in `data-tables/`.
- Optimize for safe, minimal edits to workflow JSON and deployment config.

## Repo Layout
- `workflows/`: exported n8n workflow JSON files.
- `docker/docker-compose.yml`: local deployment definition for the n8n service.
- `docs/`: operator notes and setup instructions.
- `data-tables/`: exported CSVs used by workflows.
- `README.md`: currently minimal; do not assume it contains operational detail.

## Rules Files
- No `.cursor/rules/`, `.cursorrules`, or `.github/copilot-instructions.md` files were found.
- Treat this file as the repository-specific instruction set for coding agents.

## Tooling Reality
- There is no `package.json`, `pnpm`, `npm`, `yarn`, `pytest`, `go test`, or other standard build/test runner in this repo.
- Validation is mostly workflow-JSON validation, Compose validation, and manual testing inside n8n.
- In the current workspace used to author this file, `jq` is available and `docker` is not installed.
- Do not invent missing scripts. State clearly when a command is repo-external or unavailable in the current shell.

## Build / Lint / Test Commands

### Primary Validation Commands
- Validate one workflow JSON file:
  - `jq empty "workflows/Signal - Send Message.json"`
- Validate all workflow JSON files:
  - `find workflows -name '*.json' -print0 | xargs -0 -n1 jq empty`
- Validate the Compose file syntax:
  - `docker compose -f docker/docker-compose.yml config -q`
- Render the Compose config for inspection:
  - `docker compose -f docker/docker-compose.yml config`
- List Compose services:
  - `docker compose -f docker/docker-compose.yml config --services`

### Runtime Commands
- Start n8n locally with Docker Compose:
  - `docker compose -f docker/docker-compose.yml up -d`
- Stop the stack:
  - `docker compose -f docker/docker-compose.yml down`
- Restart only n8n:
  - `docker compose -f docker/docker-compose.yml restart n8n`
- Tail n8n logs:
  - `docker compose -f docker/docker-compose.yml logs -f n8n`

### Single-Test Guidance
- There is no unit-test framework in this repository.
- The closest thing to a “single test” is:
  - `jq empty "workflows/<Workflow Name>.json"` for syntax validation of one workflow file.
  - Then run that workflow manually in the n8n UI using its `Manual Trigger` path.
- Many workflows in this repo intentionally include a `Manual Trigger` plus test input nodes for this purpose.

### Change Verification Workflow
- For workflow JSON changes:
  - Validate the edited file with `jq empty`.
  - Inspect the diff to ensure no accidental node rewrites, ID churn, or coordinate churn.
  - If n8n is available, import or open the workflow and execute the `Manual Trigger` path.
- For Compose changes:
  - Run `docker compose -f docker/docker-compose.yml config -q`.
  - Review exposed ports, environment values, and volume mounts.

## General Editing Principles
- Prefer the smallest correct change.
- Preserve existing repository structure and naming patterns.
- Do not reformat unrelated files.
- Do not rewrite entire workflow exports when only one node or field needs to change.
- Do not change workflow IDs, node IDs, `versionId`, or `meta.instanceId` unless the export process requires it.

## Formatting Conventions
- Use UTF-8 JSON as exported by n8n.
- Preserve two-space indentation in JSON files.
- Keep trailing commas out of JSON.
- Keep object/array formatting compact and consistent with existing workflow exports.
- Do not hand-minify workflow JSON.

## File Naming Conventions
- Workflow filenames usually follow `<Domain> - <Action>.json`.
- Match the existing capitalization and separator style when adding new workflows.

## Workflow Naming Conventions
- Use descriptive workflow names that match the filename.
- Prefer a clear domain/topic first, then the action.
- Keep names operator-friendly; these files are meant to be read and used in the n8n UI.
- Name nodes by intent, not by node type alone.
- Good examples from the repo include `Normalize input`, `Resolve and validate payload`, and `Set test input Airbus`.

## Imports / Dependencies
- There are no JS/TS module imports anywhere in this repo.
- Inside n8n `Code` nodes, do not add `import`, `require`, or external package assumptions.
- Use n8n-provided globals and helpers only.
- Prefer native n8n nodes over `Code` nodes when the workflow can be expressed cleanly without code.

## Code Node Style
- Keep `Code` node JavaScript direct and defensive.
- Normalize inputs early.
- Use explicit runtime checks such as `typeof value === 'string'` and `Array.isArray(value)`.
- Trim user-provided strings before validation.
- Return a single array of items in standard n8n style: `return [{ json: { ... } }];`
- Keep helper functions local to the node unless duplication is clearly worse.
- Prefer straightforward code over abstraction-heavy code.

## Types And Data Handling
- This repo is plain JSON plus embedded JavaScript, not TypeScript.
- Use runtime type narrowing instead of pretending types are guaranteed.
- Treat all incoming workflow data as untrusted.
- Validate required fields before external side effects.
- Normalize optional fields to empty strings, booleans, or explicit nullish values as needed.
- When reading nested input, guard each step carefully.

## Error Handling
- Fail early on invalid input before making HTTP requests or executing sub-workflows.
- Use `throw new Error('message')` for hard-stop validation or workflow errors.
- Use explicit boolean status fields like `okToSend` or `okToRoute` when a controlled branch is better than an exception.
- Include concise, operator-meaningful error messages.
- When an HTTP request can fail but the workflow should continue, use the node-level error-routing settings intentionally.
- Keep success/error payloads structured and predictable.

## Credentials And Secrets
- Never hardcode secrets, tokens, API keys, passwords, or private URLs into workflow JSON.
- Prefer n8n Credentials for authentication.
- Existing workflows may reference `$env.*`; do not expand that pattern casually.
- For new work, prefer Credentials or clearly centralized config nodes over hidden magic values.
- If a configurable value is expected to change, keep it in a visible `Set` node or credential, not buried in code.

## Node Choice Guidelines
- Prefer official n8n nodes first.
- Use `HTTP Request` nodes for external APIs rather than manual HTTP inside `Code` nodes.
- Use `Code` nodes for transformation, normalization, validation, or parsing when built-in nodes would be clumsy.
- Add a `Manual Trigger` test path for workflows that should be runnable by hand.

## Layout And Positioning
- Preserve the current visual layout of existing workflows.
- Do not reposition existing nodes just to make them “prettier”.
- When adding nodes, align them with the surrounding workflow structure and keep branches readable.

## Compose File Guidelines
- Keep `docker/docker-compose.yml` minimal and readable.
- Preserve existing service names unless a rename is required.
- Review changes to:
  - `ports`
  - `environment`
  - `volumes`
  - restart policy
- Do not expose new ports without a clear reason.
- Do not hardcode secrets in Compose files.

## Documentation Expectations
- If a workflow requires setup that is not obvious from the JSON, add or update docs in `docs/`.
- Keep docs practical and short.
- Prefer concrete setup steps, URLs, credential names, and expected inputs/outputs.

## Agent Checklist Before Finishing
- Confirm which files actually changed.
- Validate edited workflow JSON with `jq empty`.
- Review diffs for accidental export noise.
- Validate Compose changes with `docker compose ... config -q` when Docker is available.
- Mention clearly if a command could not be run because the tool is missing in the current environment.
- Do not claim automated tests passed when this repo has no automated test suite.
