# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`skypermit-qa-kit` is the QA/testing companion for **SkyPermit**, the app in
the sibling directory `../skypermit`. Right now it is a **test-design kit**:
written test cases plus tooling that checks them. No automated test framework
exists yet. There is no `package.json`, and Playwright, Vitest and similar
tools are not installed. Don't assume one is in use until it appears in a
`package.json` or config file here.

**Do not modify `../skypermit`** unless explicitly asked. Treat it as the
read-only system under test. (`.claude/settings.json` grants read access to it
via `additionalDirectories`.)

## Layout and how the pieces fit

- `test-cases/<module>/*.md`: the live registry of E2E test cases
  (`flight-request/`, `auth/`). Each file holds one or more cases in the format
  defined by `.claude/skills/e2e-test-design/templates.md`: ID, Title, Doc ref,
  Role(s), Preconditions (optional; omit it rather than writing "None"), Steps,
  Expected result, Rationale. See `test-cases/README.md`.
- `.claude/skills/e2e-test-design/`: the `e2e-test-design` skill (how to design
  a case around the state machine and gate pipeline), `templates.md` (the
  required format) and `examples.md` (worked teaching examples; these
  deliberately live apart from `test-cases/`).
- `scripts/`: plain Node scripts with no dependencies and no install step (see
  `scripts/README.md`):
  - `validate-cases.mjs`: *structural* checks on cases (required fields, ID
    format and uniqueness, Doc ref path convention, cited docs exist, steps
    name a `data-testid`, anti-patterns).
  - `rule-coverage.mjs`: cross-references the `rule_reference` table in
    `../skypermit/docs/06-business-rules.md` §6 against cases. A rule counts
    as covered only when a case asserts it in **Expected result**. Doc-ref
    mentions (`ref?`) and negated mentions ("not `ZC-2026.1 §3`") don't count.
- `.claude/agents/case-reviewer.md`: the *semantic* reviewer (do Steps reach
  the Preconditions, does Expected follow, does the cited rule/selector/gate
  code actually exist in `../skypermit`). It runs `validate-cases.mjs` first
  and does not repeat its checks. The split is deliberate: regex-decidable
  checks go to the script, anything needing judgment goes to the agent.
- `.claude/commands/`: `/coverage [--gaps|--markdown]` (runs and interprets
  `rule-coverage.mjs`) and `/check-test-coverage <ID>` (checks one case against
  the app's real code; it reports only and never edits).
- `api-automation/`: a separate, still-empty API-level suite with its own
  `api-automation/CLAUDE.md` (endpoint map, auth and status-code facts). Keep
  API tests and E2E/UI work in separate directories.

## Commands

The scripts use `import.meta.dirname`, so they need **Node ≥ 20.11**. The
system `node` on this machine is v10 and fails with `SyntaxError: Unexpected
token {`. Use a newer Node.

```bash
node scripts/validate-cases.mjs                 # structural check of all cases
node scripts/validate-cases.mjs --json
node scripts/validate-cases.mjs --cases <dir> --app <dir>

node scripts/rule-coverage.mjs                  # coverage table
node scripts/rule-coverage.mjs --gaps           # uncovered rules only
node scripts/rule-coverage.mjs --markdown
node scripts/rule-coverage.mjs --rules <path> --cases <dir>
```

Exit codes for both: `0` clean, `1` problems/gaps found, `2` couldn't run.
`rule-coverage` exiting `1` is the normal state while coverage is incomplete.
If it exits `2`, the §6 heading in `06-business-rules.md` has probably moved;
fix the parser instead of reporting zero coverage.

There is no build, lint, or test-runner command yet.

## Conventions for writing cases

- **IDs** are `E2E-<OP|REV|X>-<3 digits>`. Sequence numbers are **global across
  all modules** and never reused. Check every `test-cases/` subdirectory
  before choosing the next number.
- **Doc ref** paths are repo-rooted (`skypermit/docs/...`), never
  `../skypermit/...`. Prefer a `rule_reference` anchor (e.g. `A-7 §1`), then
  a numbered section, then a quoted heading.
- **Selector discipline** (`.claude/rules/selector-discipline.md`): before
  writing or editing Steps, grep every `data-testid` against `../skypermit`
  (`app/`, `components/`). A missing selector is a bug to report against
  skypermit. Don't invent a selector or fall back to text/CSS. Known churn:
  `operator-id-input` was removed; use `operator-login-link` /
  `operator-register-link` / `operator-session-info` /
  `operator-logout-button` instead.
- Expected results must name the specific error (gate + `rule_reference`, or
  `409`), not "shows an error".

## The system under test

Read these in `../skypermit` before writing any test:

1. `../skypermit/CLAUDE.md`: stack, gate implementation status, conventions.
2. `docs/01-data-model.md`: entities, fields, enums.
3. `docs/02-state-machine.md`: the seven flight-request states. Any transition
   not listed is illegal and returns `409`.
4. `docs/03-api-contract.md`: endpoints under `/api/v1` and the `422` body
   (`failing_gate`, `rule_reference`, `message`).
5. `docs/04-frontend-spec.md`: the operator and reviewer portals and the
   **`data-testid` contract**, which is the primary hook for E2E work.
6. `docs/06-business-rules.md` (plus `07-clarifications.md` for corrections):
   gate logic and the `rule_reference` citations. Gate code lives in
   `../skypermit/lib/gates/`. A rule that is documented but missing from that
   code is unimplemented behavior.

Key facts worth not re-deriving:

- Gates run in fixed order, `PILOT → AIRCRAFT → ZONE → TIME_WINDOW`, and stop
  at the first failure. A case asserting gate X must keep every earlier gate
  passing. Getting this wrong is the most common design error here.
- `POST /flight-requests/{id}/validate` (UI: `flight-request-check-button`)
  runs the same pipeline as `submit` but never changes state, so prefer it for
  pure gate tests. A request can still pass `validate` and fail `submit`,
  because the aircraft overlap check counts only `APPROVED`/`UNDER_REVIEW`
  requests.
- Operators authenticate via real Supabase Auth (session cookie). Reviewers
  use a static `X-Reviewer-Id` header with no login. Never mix the two; a
  cross-role case switches cleanly at the handoff.
- Pilot/aircraft/zone pickers are deliberately **not** pre-filtered. Invalid
  combinations must fail at validate/submit, not at selection.
- For `KBP-CTR-3` and `ODS-CTR-5`, an amendment overrides the general
  `ZC-2026.1` curfew (for `KBP-CTR-3` that's `A-7 §1`). Cases should assert
  the amended citation, not the general one.

## Running the app under test

`../skypermit` needs its own `.env.local` and a Supabase instance set up per
`../skypermit/supabase/SETUP.md`. Then, from `../skypermit`:

```bash
npm run dev     # serves the app at localhost:3000
```

Once a framework is chosen here, update this file with the install, run-all,
single-test and lint commands, and with how the suite points at a running
`skypermit` instance (base URL, operator vs. reviewer auth setup).
