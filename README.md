# archlint

Declarative architecture linter for teams and AI agents. Define your project structure as code, enforce it in CI, and onboard both people and agents in seconds.

## Why archlint?

Project structure drifts. New developers put files in the wrong place. AI coding agents create modules without tests. That `services/` folder that was supposed to stay clean now has 12 random scripts in it.

Architecture decisions live in someone's head, a wiki nobody reads, or a PR comment from 2023.

**archlint makes structure enforceable.** One config file that:

- **Enforces structure** — validates your project layout in CI, like a linter for your architecture
- **Onboards developers** — new team members read the config and instantly understand how the project is organized
- **Guardrails AI agents** — agents read the config before writing code, and validate after. Claude Code, Cursor, Copilot, Aider — any agent that can run a CLI
- **Scaffolds modules** — generate new modules with the right structure, every time
- **Replaces architecture docs** — the config *is* the documentation, and it's enforced

### For developers

```
$ archlint lint

ERROR ARCH001: src/payments/tests/test_service.py - Missing required file
  Fix: Create the file at src/payments/tests/test_service.py.

ERROR ARCH002: src/random_script.py - Undeclared file in strict mode
  Fix: Remove src/random_script.py or add it to the architecture definition.

2 error(s), 0 warning(s). FAILED.
```

### For AI agents

```
$ archlint lint --format json
{
  "passed": false,
  "violations": [
    {
      "code": "ARCH001",
      "path": "src/payments/tests/test_service.py",
      "description": "Missing required file",
      "solution": "Create the file at src/payments/tests/test_service.py."
    }
  ]
}
```

Agents parse the JSON output, read the `solution` field, and fix the violation automatically.

## Highlights

- **Architecture as code** — your project structure is defined, documented, and enforced in one file
- **For teams and agents** — human-readable text output for developers, structured JSON for AI agents
- **Zero dependencies** — stdlib only, installs in milliseconds. YAML support available as optional extra
- **File-type agnostic** — validates any file: `.py`, `Dockerfile`, `.yaml`, `.env`, `Makefile`, anything
- **Two modes** — strict (only declared files allowed) or relaxed (extra files ok), configurable per folder
- **Scaffold from config** — `archlint init` creates modules with the right structure
- **Gradual adoption** — baseline support lets you adopt on existing projects without fixing everything first
- **CI-ready** — runs in seconds, clear exit codes, works in any pipeline

## Installation

```bash
# With pip
pip install archlint

# With uv
uv add archlint

# With YAML config support
pip install archlint[yaml]
```

Requires Python 3.12+. Zero dependencies by default.

## Quickstart

### 1. Define your architecture

Create `.archlint.yaml` (requires `archlint[yaml]`) or `.archlint.toml` in your project root:

```yaml
# .archlint.yaml
version: 1

placeholders:
  domain: "[a-z][a-z0-9_]+"
  name: "[a-z][a-z0-9_]+"
  version: "v[0-9]+"

ignores:
  - __pycache__/
  - .venv/
  - .git/

architecture:
  structure:
    # --- Source code ---
    - path: src
      mode: strict
      children:
        - path: "{domain}"                          # placeholder: constrained, named, reusable
          description: "Business domain package"
          children:
            - path: __init__.py
              code: DOM001
              description: "Package marker"
              solution: "Create an empty __init__.py file."

            - path: service.py
              code: DOM002
              description: "Core business logic"
              solution: "Create service.py with a service class."

            - path: models.py
              code: DOM003
              description: "Data models"
              solution: "Create models.py with model definitions."

            - path: routes.py
              status: optional

            # one-of: at least one must exist
            - code: DOM004
              description: "Schemas module or package"
              solution: "Create schemas.py or a schemas/ directory."
              one_of:
                - path: schemas.py
                - path: schemas/

            # glob: any versioned API folder
            - path: "{version}"
              status: optional
              description: "Versioned API folder (v0, v1, v2, ...)"
              children:
                - path: __init__.py
                - path: endpoints.py
                  code: DOM005
                  description: "HTTP endpoints for this version"
                  solution: "Create endpoints.py with API routes."
                - path: requests.py
                - path: responses.py

            - path: tests
              children:
                - path: __init__.py
                - path: test_{name}.py

        - path: main.py

    # --- Tests mirror source ---
    - path: test
      children:
        - path: "{domain}"
          depends_on: "src/{domain}"                # only required if source exists
          children:
            - path: __init__.py
            - path: test_unit_service.py
              code: TEST001
              description: "Unit tests for service.py"
              solution: "Create test_unit_service.py with unit tests."

            - path: "{version}"
              status: optional
              depends_on: "src/{domain}/{version}"         # mirrors versioned folders
              children:
                - path: __init__.py
                - path: test_int_endpoints.py

    - path: Dockerfile
    - path: pyproject.toml

exceptions:
  - path: src/legacy/
    skip: [DOM001, DOM002]
    reason: "Legacy module, migrating by Q3"
```

### 2. Validate your project

```bash
$ archlint lint

ERROR ARCH001: src/payments/tests/test_service.py - Missing required file
  Fix: Create the file at src/payments/tests/test_service.py.

ERROR ARCH002: src/random_script.py - Undeclared file in strict mode
  Fix: Remove src/random_script.py or add it to the architecture definition.

2 error(s), 0 warning(s). FAILED.
```

### 3. Scaffold new modules

`archlint init` reads the architecture config and creates the structure for you. Pass a path and it figures out which templates apply:

```bash
# Scaffold everything from src/ down — prompts for placeholders it can't resolve
$ archlint init src/
Enter value for domain: payments
Enter value for name: payments

Created src/payments/
  FILE src/payments/__init__.py
  FILE src/payments/models.py
  FILE src/payments/service.py
  FILE src/payments/routes.py
  DIR  src/payments/tests/
  FILE src/payments/tests/__init__.py
  FILE src/payments/tests/test_payments.py
```

```bash
# Scaffold a specific module — "orders" is resolved as {domain} automatically
$ archlint init src/orders
Enter value for name: engine

Created src/orders/
  FILE src/orders/__init__.py
  FILE src/orders/models.py
  FILE src/orders/service.py
  DIR  src/orders/tests/
  FILE src/orders/tests/__init__.py
  FILE src/orders/tests/test_orders.py
```

```bash
# Non-interactive mode for CI/agents — pass placeholders explicitly
$ archlint init src/orders --set name orders
```

Scaffolded files are created empty — archlint will flag them until you add content.

## Integrations

### CI (GitHub Actions)

Catch structure violations before they land on `main`:

```yaml
# Full project check
- name: Architecture check
  run: |
    pip install archlint
    archlint lint

# Or only check files changed in the PR (faster)
- name: Architecture check (changed files only)
  run: |
    pip install archlint
    git diff --name-only origin/main | archlint lint --stdin
```

### Pre-commit hooks

Catch violations before they leave the developer's machine. archlint receives the staged filenames, resolves each to its nearest declared scope in the architecture, and lints only those scopes:

```yaml
- repo: local
  hooks:
    - id: archlint
      name: archlint
      entry: archlint lint
      language: system
```

### AI agent guardrails (CLAUDE.md / AGENTS.md / .cursorrules)

Give agents the context to write code in the right place and validate their output:

```markdown
## Architecture

This project uses archlint to enforce structure. Before writing code:
1. Read `.archlint.yaml` to understand the project architecture
2. After making changes, run `archlint lint --format json` to validate

If violations are found, read the `solution` field and fix them.
```

### Developer onboarding

New to the project? The config *is* the architecture guide:

```bash
# Understand the project structure
cat .archlint.yaml

# Create a new domain module with the right structure
archlint init src/payments

# Check your work
archlint lint
```

## Commands

### `archlint lint`

Validate project structure against config.

```bash
archlint lint                          # check entire project
archlint lint src/users/service.py     # check only the scope containing this file
archlint lint --format json            # JSON output (for agents)
archlint lint --stdin                  # read file paths from stdin
archlint lint --config path/to/config  # custom config path
archlint lint --update-baseline        # save current violations as baseline
```

When given specific files, archlint resolves each to its nearest declared scope in the architecture config, then lints that scope. This catches missing siblings, strict mode violations, and conditional dependencies — not just the file itself.

```bash
# These all lint the src/users/ scope
archlint lint src/users/service.py
archlint lint src/users/models.py src/users/service.py   # deduped

# Pipe from git
git diff --name-only origin/main | archlint lint --stdin
```

| Exit code | Meaning |
|-----------|---------|
| 0 | All checks passed |
| 1 | Violations found |
| 2 | Config error |

### `archlint init <path>`

Scaffold files and folders by walking the architecture config from the given path down. Resolves placeholders from the path when possible, prompts for the rest.

```bash
archlint init src/                    # scaffold everything under src/, prompts for placeholders
archlint init src/payments            # scaffold a specific module, infers {domain}=payments
archlint init src/payments --set name payments  # non-interactive, for CI/agents
archlint init .                       # scaffold the entire project
archlint init src/payments --dry-run  # preview what would be created
```

If a placeholder can't be resolved and there's no TTY (e.g., running in CI), it errors instead of prompting.

### `archlint fix`

Auto-fix violations by creating missing files and folders. Like `init`, but driven by lint results instead of a path.

```bash
archlint fix                           # fix all violations that can be auto-fixed
archlint fix --dry-run                 # preview what would be created
archlint fix --format json             # output fixed violations as JSON
```

Only fixes structural violations (missing files/folders). Cannot fix empty files (that's your job), undeclared files in strict mode (archlint won't delete your code), or naming violations.

### `archlint diff`

Show a two-way gap analysis between your config and your project:

```bash
$ archlint diff

Missing (declared but not found):
  + src/payments/tests/test_service.py
  + src/payments/schemas.py

Extra (found but not declared):
  - src/random_script.py
  - src/payments/temp_fix.py

3 files to add, 2 files to remove.
```

Useful for understanding how far your project is from the declared architecture before committing to strict mode.

### `archlint generate-config`

Scan an existing project and generate a config from its current structure. The fastest way to adopt archlint — start from what you have, then tighten.

```bash
archlint generate-config                    # output to stdout
archlint generate-config -o .archlint.yaml  # write to file
archlint generate-config --strict           # generate with strict mode
```

```
$ archlint generate-config

Generated .archlint.yaml from project structure.
Detected:
  - 3 domain modules (users, orders, payments) → created template "domain_module"
  - 12 placeholders inferred
  - 47 files mapped

Review the generated config and adjust as needed.
```

The generated config is a starting point — it captures what exists but doesn't know your intent. Review it, remove things that shouldn't be there, add rules, then commit.

### `archlint check-config`

Validate your archlint config file. Points to exact locations when something is wrong.

```bash
$ archlint check-config

Config error at templates[0].files[2]:
  Undefined placeholder '{naem}' — did you mean '{name}'?

Config error at architecture.structure[1].children[0]:
  Template 'domian_module' not found — did you mean 'domain_module'?

2 errors found.
```

## Config reference

### Path matching: placeholders vs globs

archlint supports two kinds of dynamic path matching:

**Placeholders** — named, regex-constrained, reusable. Use when you need to constrain what matches or reference the matched value later (in solutions, scaffold, or conditionals).

```yaml
placeholders:
  domain: "[a-z][a-z0-9_]+"
  version: "v[0-9]+"

architecture:
  structure:
    - path: "{domain}"          # matches "users", "orders" — not "My-Module"
    - path: "{version}"         # matches "v0", "v1" — not "vendor", "views"
```

**Globs** — simple wildcards, no constraints. Use when any match is fine and you don't need the name.

```yaml
architecture:
  structure:
    - path: "*.config.yaml"    # any config file
    - path: "docker-compose.*" # docker-compose.dev.yaml, docker-compose.prod.yaml
```

**Rule of thumb**: if you'd be upset that `vagrant/` matched your `v*` pattern, use a placeholder with a regex. If you truly mean "anything", use a glob.

### Inline rule codes

Every file or folder entry can carry its own `code`, `description`, and `solution` directly. This gives agents the context right where the file is defined — no indirection to a separate rules section.

```yaml
- path: service.py
  code: DOM002
  description: "Core business logic layer"
  solution: "Create service.py with a service class."
```

When archlint reports a violation for this file, it uses the inline code, description, and solution.

### Modes

Set per folder, not globally:

- **`strict`** — only declared files and folders are allowed. Anything else is a violation.
- **`relaxed`** (default) — declared files must exist, but extra files are fine.

```yaml
architecture:
  structure:
    - path: src
      mode: strict        # locked down
      children:
        - path: scripts
          mode: relaxed  # anything goes here
```

### `forbidden`

Block files or patterns. Any file matching a forbidden pattern is a violation **unless** it's explicitly declared in the architecture.

```yaml
- path: test/{domain}
  mode: relaxed
  forbidden:
    - path: "test_int*.py"
      code: TEST001
      description: "Integration tests must be named test_int_endpoints.py"
      solution: "Rename to test_int_endpoints.py."
    - path: utils.py
      code: PROJ001
      description: "utils.py is banned — use specific module names"
      solution: "Move utilities into a named module (e.g., formatting.py, validation.py)."
  children:
    - path: __init__.py
    - path: test_unit_service.py
    - path: test_int_endpoints.py   # declared, so not blocked by the forbidden pattern
```

In strict mode, `forbidden` is redundant — anything not declared is already a violation.

### `one_of`

At least one of the listed paths must exist:

```yaml
- code: DOM004
  description: "Schemas module or package"
  solution: "Create schemas.py or a schemas/ directory."
  one_of:
    - path: schemas.py
    - path: schemas/
```

### `depends_on`

A folder or file is only required if another path exists. Essential for test mirroring:

```yaml
# test/users/ is only required if src/users/ exists
- path: test
  children:
    - path: "{domain}"
      depends_on: "src/{domain}"
      children:
        - path: test_unit_service.py

        # versioned test folders mirror versioned source folders
        - path: "{version}"
          depends_on: "src/{domain}/{version}"
          children:
            - path: test_int_endpoints.py
```

### Templates

Reusable file/folder structures:

```yaml
templates:
  - id: domain_module
    description: "Standard domain module"
    files:
      - path: __init__.py
      - path: service.py
      - path: routes.py
        status: optional      # required is the default
    folders:
      - path: tests
        files:
          - path: __init__.py
```

Reference them in architecture:

```yaml
architecture:
  structure:
    - path: src
      children:
        - path: "{domain}"
          uses_template: domain_module
```



### Settings

All have defaults, all are optional:

```yaml
settings:
  allow_empty_files: false    # default: false
  follow_symlinks: false      # default: false
  output_format: text         # text | json, default: text
```

Settings can also live in `pyproject.toml`:

```toml
[tool.archlint]
config = ".archlint.yaml"
ignores = ["__pycache__/", ".venv/"]
baseline = ".archlint-baseline.json"
output_format = "text"
```

**Precedence**: CLI flags > `.archlint.yaml` > `pyproject.toml` > defaults.

### Baseline (gradual adoption)

For existing projects, generate a baseline to suppress current violations:

```bash
archlint lint --update-baseline
```

Future runs ignore baselined violations, only reporting new ones.

### Exceptions

Suppress specific rules for specific paths:

```yaml
exceptions:
  - path: src/legacy/
    skip: [DOM001, DOM002]
    reason: "Legacy module, migrating by Q3"
```

## Built-in rule codes

These are always available regardless of your config:

| Code | Description |
|------|-------------|
| ARCH001 | Missing required file |
| ARCH002 | Undeclared file in strict mode |
| ARCH003 | Empty file detected |
| ARCH004 | Missing required folder |
| ARCH005 | Conditional rule violation |
| ARCH006 | Forbidden file in relaxed mode |

You can also define your own codes inline on any entry (`DOM001`, `TEST001`, etc.). Custom codes use `error` severity by default.

## License

MIT
