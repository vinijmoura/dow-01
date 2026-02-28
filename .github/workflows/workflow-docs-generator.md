---
description: |
  When any YML file in .github/workflows/ is changed, this workflow creates or updates
  documentation in the docs/ folder. Each YML file gets a corresponding markdown file
  that describes its steps in plain language.

on:
  push:
    branches: [main]
    paths:
      - '.github/workflows/*.yml'

permissions:
  contents: read
  issues: read
  pull-requests: read

network: defaults

tools:
  github:
    lockdown: false
  edit:

safe-outputs:
  create-pull-request:
    title-prefix: "[docs] "
    labels: [documentation]
    draft: false
    if-no-changes: ignore

engine: copilot
---

# Workflow Documentation Generator

When a GitHub Actions workflow YML file changes, create or update documentation for it in the `docs/` folder.

## Task

For each YML file that was modified or added in `.github/workflows/` in the commit that triggered this workflow:

1. Read the content of the YML file.
2. Analyze its structure: workflow name, triggers (`on:`), environment variables (`env:`), jobs, and steps.
3. Create or update a corresponding markdown documentation file at `docs/<yml-filename-without-extension>.md`.
4. If the `docs/` folder does not exist, create it.

## Documentation Format

Each generated markdown file must follow this structure:

```
# <Workflow Name>

## Overview

<Brief description of what the workflow does and when it runs.>

## Triggers

<Explain each trigger condition. For example: "Runs on push to the `main` branch" or "Runs on pull requests targeting `main`". Also mention any path filters.>

## Environment Variables

<List and describe any environment variables defined in the `env:` section. If there are none, omit this section.>

## Jobs

### <Job Name>

**Runs on**: <runner>

**Needs**: <list any job dependencies, or "None">

**Condition**: <any `if:` condition, or "Always runs">

**Permissions**: <list permissions, or "Default">

#### Steps

1. **<Step Name>**: <Plain-language explanation of what this step does, including any key parameters or conditions (e.g., `if:` conditions, action used, command run).>
2. ...

### <Next Job Name>

...
```

## Instructions

- Use clear, non-technical language where possible so that anyone can understand the workflow.
- Explain what each step accomplishes, not just the technical command.
- If a step has an `if:` condition, describe when it runs.
- If a step uses a known GitHub Action (e.g., `actions/checkout`, `actions/setup-dotnet`), briefly explain what that action does.
- After writing all documentation files, create a pull request with the changes.

## Context

- Current repository: ${{ github.repository }}
- Commit SHA: ${{ github.event.head_commit.id }}
- Files changed: use the GitHub API to retrieve the list of files changed in commit `${{ github.event.head_commit.id }}` (e.g., `GET /repos/{owner}/{repo}/commits/{sha}`) and filter for files matching `.github/workflows/*.yml`
