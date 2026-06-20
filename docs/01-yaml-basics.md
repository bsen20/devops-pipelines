# 01 — YAML Basics

> YAML (YAML Ain't Markup Language) is the standard for defining CI/CD pipelines, Kubernetes manifests, Terraform configs, and Docker Compose files.

---

## Why YAML for Pipelines?

| Feature | Why It Matters |
|---------|---------------|
| Human-readable | Non-developers can read and review pipeline configs |
| Hierarchical | Naturally models pipeline stages, jobs, steps |
| No brackets/braces | Cleaner than JSON for complex configs |
| Anchors & aliases | DRY — reuse blocks across a pipeline file |
| Multi-doc support | Multiple pipelines in one file |

---

## Syntax Fundamentals

### Scalars (Basic Values)

```yaml
# Strings (quotes optional for simple strings)
name: John
greeting: "Hello, World!"       # Quotes force string interpretation
version: "1.0"                  # Without quotes, this is a float

# Numbers
port: 8080
timeout: 30.5
negative: -42

# Booleans
enabled: true
debug: false
yes: yes
no: no

# Null
value: null
empty: ~

# Dates (ISO 8601)
timestamp: 2025-06-20T14:30:00Z
date: 2025-06-20
```

### Collections

```yaml
# Sequences (arrays/lists)
steps:
  - npm install
  - npm test
  - npm build

# Inline (JSON-style)
steps: [npm install, npm test, npm build]

# Mappings (dictionaries/objects)
job:
  name: build
  runs-on: ubuntu-latest
  steps:
    - name: Checkout
      uses: actions/checkout@v4

# Nested structures
deploy:
  production:
    server: prod.example.com
    port: 443
    ssl: true
  staging:
    server: staging.example.com
    port: 8080
    ssl: true
```

### Multi-line Strings

```yaml
# Literal block (preserves newlines — useful for shell scripts)
script: |
  echo "Starting deployment"
  docker build -t app .
  docker push app

# Folded block (wraps lines into spaces — useful for long messages)
description: >
  This is a very long description
  that will be folded into a single
  line when parsed.

# Flow scalar (single line, no escaping issues)
command: >
  kubectl apply -f deployment.yaml
  && kubectl rollout status deployment/app
```

---

## Anchors, Aliases, and Extensions (DRY)

```yaml
# Define reusable blocks with anchors (&)
x-defaults: &defaults
  runs-on: ubuntu-latest
  env:
    NODE_ENV: production

x-cache: &cache
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

# Reuse with aliases (*)
jobs:
  lint:
    <<: *defaults                     # Merge anchor
    steps:
      - uses: actions/checkout@v4
      - *cache                         # Insert cached steps
      - run: npm run lint

  test:
    <<: *defaults
    steps:
      - uses: actions/checkout@v4
      - *cache
      - run: npm test

# Override specific fields
  build:
    <<: *defaults
    runs-on: windows-latest           # Override just the runner
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
```

---

## Multi-Document YAML

```yaml
# File: config.yaml
---
# Document 1: Development config
apiVersion: v1
kind: Config
environment: dev
---
# Document 2: Production config
apiVersion: v1
kind: Config
environment: prod
replicas: 5
---
# Document 3: Monitoring config
apiVersion: monitoring/v1
kind: Dashboard
title: "Production Overview"
```

Parsed as separate documents — useful for Kubernetes manifests.

---

## Common YAML Mistakes

| Mistake | Wrong | Correct |
|---------|-------|---------|
| Tab indentation | `\tname: John` | Spaces only |
| Inconsistent spacing | `- item1\n - item2` | Same indentation level |
| Unquoted booleans | `value: yes` | `value: "yes"` (if string) |
| Unquoted special chars | `text: yes:no` | `text: "yes:no"` |
| Missing space after colon | `key:value` | `key: value` |
| Trailing spaces | `key: value ` | `key: value` |

**Golden rule:** Use 2-space indentation. NEVER use tabs. Be consistent.

---

## YAML Validation

```bash
# Validate YAML syntax
python -c "import yaml; yaml.safe_load(open('pipeline.yaml'))"

# Using yamllint
yamllint pipeline.yaml

# Using online validator
# https://www.yamllint.com/
```

---

## YAML in CI/CD — Common Patterns

### Environment Variables

```yaml
env:
  NODE_VERSION: "20"
  REGISTRY: ghcr.io

jobs:
  build:
    steps:
      - run: echo "Node ${{ env.NODE_VERSION }}"
```

### Matrix Builds

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 22]

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
```

### Conditional Steps

```yaml
steps:
  - name: Deploy to production
    if: github.ref == 'refs/heads/main'
    run: ./deploy.sh
```

---

## YAML Linting Rules (yamllint)

```yaml
# .yamllint
extends: default

rules:
  line-length:
    max: 120
    level: warning
  indentation:
    spaces: 2
    indent-sequences: consistent
  document-start: disable
  truthy:
    level: error
```

---

## Quick Reference

```yaml
# Structure
key: value              # Scalar
key: [a, b, c]         # Inline list
key:                    # Block list
  - a
  - b
key: {a: 1, b: 2}     # Inline map
key:                    # Block map
  a: 1
  b: 2

# String types
plain: hello            # Unquoted
single: 'hello'         # Single-quoted (no escaping)
double: "hello\nworld"  # Double-quoted (escape sequences)
literal: |              # Preserve newlines
  line1
  line2
folded: >               # Fold into one line
  line1
  line2

# Reuse
&anchor                 # Define anchor
*anchor                 # Reference anchor
<<: *anchor             # Merge anchor

# Types
true, false, yes, no    # Booleans
123, 3.14               # Numbers
null, ~                 # Null
2025-06-20              # Date (ISO)
```
