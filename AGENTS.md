# Repository Guidelines

## Project Structure & Module Organization

- `format/` contains the normative on-disk format specifications for WAL, SST,
  manifests, leases, and shared value encoding.
- `schema/` contains JSON Schema projections used to validate decoded fixtures.
  Wire layouts in `format/` remain authoritative.
- `conformance/` defines interoperability and fixture requirements. No golden
  fixtures are published yet.
- `notes/` contains non-normative research and implementation findings.

## Build, Test, and Development Commands

There is no compilation step or project-specific test runner. Before submitting:

```sh
jq empty schema/*.json       # Parse every JSON Schema file
git diff --check             # Detect whitespace errors
rg -n 'TODO: verify' format/ # Review unresolved format questions
```

## Specification Style & Naming Conventions

Write concise Markdown with descriptive headings, tables for field layouts, and
backticks around field names, filenames, and literal values. Wrap prose near 80
columns and use LF line endings. JSON uses two-space indentation.

Use *must*, *must not*, *required*, *should*, and *may* in the RFC 2119 sense.
Clearly separate wire-format requirements from implementation policy. Mark
unsettled claims `TODO: verify` and include them in the document's **Open
questions** section. Keep schema constraints synchronized with their normative
format rules.

## Specification Authority

This repository is the source of truth. Midge, Pants, and other implementations are
evidence used to reconstruct intent and test interoperability, not normative
authorities. Write requirements as `spec -> implementation` contracts without
implementation attribution. When code and specification disagree, fix the code if
the rule is sound; otherwise amend the specification deliberately, documenting the
rationale and compatibility impact. Never silently change a rule to match current
code. Keep reverse-engineering provenance and engine-specific findings in `notes/`.

## Testing Guidelines

For schema changes, exercise both an accepted fixture and one that must be rejected.
Reader-only malformed or reserved-state fixtures must be separate from conforming
writer-output fixtures.

## Commit & Pull Request Guidelines

Use short, imperative commit subjects consistent with history, such as `Clarify
format contracts and conformance schemas`. Keep commits focused and update
`CHANGELOG.md` when normative behavior, schemas, or conformance expectations change.

Pull requests should summarize the affected format, compatibility impact, validation
commands, and relevant issues or evidence. Call out breaking wire changes,
unresolved questions, and schema limitations. Screenshots are unnecessary.
