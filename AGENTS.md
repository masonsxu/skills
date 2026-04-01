# AGENTS.md — Agent Skills Repository Guide

This is a **documentation-only repository** containing Agent Skills definitions (Markdown + JSON). There is no executable code, no build system, and no test suite in this repo itself.

## Repository Structure

```
skills/
  <skill-name>/SKILL.md                      # Standalone skill
  <skill-name>/evals/evals.json              # Optional eval definitions
  <group-name>/<skill-name>/SKILL.md         # Skill within a group
```

- Each skill lives in its own directory with a `SKILL.md` file.
- Evals are optional and stored in `evals/evals.json` within the skill directory.

## Validation Commands

Since this is a Markdown-only repo, validation is limited to:

```bash
# Check YAML frontmatter in all SKILL.md files
grep -rL '^---$' skills/*/SKILL.md skills/*/*/SKILL.md

# Verify no broken internal links in README.md (manual review)
# Verify evals.json is valid JSON
python3 -c "import json, glob; [json.load(open(f)) for f in glob.glob('skills/**/evals/*.json', recursive=True)]"
```

There are no build, lint, or test commands for this repository.

## SKILL.md File Format

Every SKILL.md must begin with YAML frontmatter:

```yaml
---
name: <kebab-case-skill-name>
description: |
  Multi-line description of when to trigger this skill.
  Be specific about keywords and scenarios that should activate it.
argument-hint: "<optional-argument-hint>"  # Optional
---
```

### Frontmatter Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | kebab-case identifier for the skill |
| `description` | Yes | Detailed trigger description — mentions keywords, scenarios, library names |
| `argument-hint` | No | Placeholder showing expected user input format (e.g., `<domain-name>`) |

### Body Structure

SKILL.md body should follow this pattern:

1. **Title** (H1) — Skill name
2. **Brief description** — What the skill does and what `$ARGUMENTS` it accepts
3. **Architecture overview** — Context the agent needs (directory layout, layering, etc.)
4. **Step-by-step instructions** — Numbered steps the agent follows sequentially
5. **Code examples** — Inline code blocks showing correct patterns
6. **Verification steps** — Commands to confirm the work is correct

## Code Style Guidelines (for SKILL.md Content)

### Markdown Formatting

- Use H1 (`#`) for the skill title, H2 (`##`) for major sections, H3 (`###`) for subsections.
- Use fenced code blocks with language tags: `go`, `bash`, `ts`, `json`, `yaml`, `ini`.
- Use tables for structured reference data (ports, env vars, coverage targets, etc.).
- Keep lines under 120 characters where practical.

### Go Code Examples (in CloudWeGo Skills)

When writing Go code examples within skills, follow these conventions from the target project:

- **Import order**: Standard library → Third-party libraries → Project internal packages.
- **Interface naming**: No `I` prefix (e.g., `UserProfileRepository`).
- **Implementation naming**: `Impl` suffix (e.g., `UserProfileRepositoryImpl`).
- **Max line length**: 120 characters.
- **Error handling chain**: `DAL (errno.WrapDatabaseError) → Logic (errno.ErrNo) → Handler (errno.ToKitexError) → Gateway (kerrors.FromBizStatusError)`.
- **Error codes**: 6-digit business codes segmented by domain (e.g., `100xxx` = user, `200xxx` = org).
- **Dependency injection**: Use Wire; each Provider returns an interface type.
- **Testing**: table-driven tests with `testify/assert` and `testify/require`; mocks via `go.uber.org/mock/gomock`.

### TypeScript Code Examples (in pretext-integration Skill)

- Use TypeScript with explicit types.
- Import from `@chenglou/pretext` using named imports.
- Show the two-phase pattern: `prepare*()` once, then `layout*()` many times.

## Evals Format (`evals.json`)

```json
{
  "skill_name": "<matching-name-from-frontmatter>",
  "evals": [
    {
      "id": 0,
      "prompt": "User prompt describing a task",
      "expected_output": "Description of what the correct response should contain",
      "files": []
    }
  ]
}
```

- `skill_name` must match the `name` field in SKILL.md frontmatter.
- `id` starts at 0 and increments.
- `files` lists any file paths the eval needs (usually empty).

## Adding a New Skill

1. Create directory: `skills/<skill-name>/` or `skills/<group>/<skill-name>/`.
2. Write `SKILL.md` with proper frontmatter and structured body.
3. Optionally add `evals/evals.json` for automated testing.
4. Update `README.md` tables to include the new skill.
5. Verify frontmatter parses correctly and all code examples are syntactically valid.

## CloudWeGo Target Project Commands (Reference)

These commands are referenced by the CloudWeGo skills for use in the **target project**, not this repo:

```bash
# Code generation
cd rpc/identity_srv && ./script/gen_kitex_code.sh
cd gateway && ./script/gen_hertz_code.sh [identity|permission]
cd rpc/identity_srv/wire && wire
cd gateway/internal/wire && wire

# Build & verify
cd rpc/identity_srv && go build ./...
cd gateway && go build ./...

# Lint
cd rpc/identity_srv && golangci-lint run
cd gateway && golangci-lint run

# Run all tests
cd rpc/identity_srv && go test ./... -v -count=1

# Run a single test
cd rpc/identity_srv && go test ./biz/logic/department/ -v -run TestLogicImpl_CreateDepartment

# Test coverage
cd rpc/identity_srv && go test ./... -coverprofile=coverage.out && go tool cover -func=coverage.out | grep total
```

## Language

Skills in the `cloudwego-microservice/` group use **Chinese** for descriptions, comments, and user-facing text. The `pretext-integration` skill uses **English**. Match the language of the existing skills in the same group when adding new ones.
