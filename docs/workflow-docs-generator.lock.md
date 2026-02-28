# Workflow Documentation Generator

## Overview

This workflow automatically generates or updates documentation for GitHub Actions workflow files. Whenever a `.yml` file in the `.github/workflows/` folder is pushed to the `main` branch, the workflow uses GitHub Copilot CLI as an AI agent to analyze the changed workflow files and create or update corresponding markdown documentation in the `docs/` folder.

> **Note:** This file was auto-generated from `.github/workflows/workflow-docs-generator.lock.yml`, which is itself a compiled file managed by `gh-aw`. Do not edit the lock file directly.

## Triggers

Runs on **push** to the `main` branch, but only when files matching `.github/workflows/*.yml` are changed.

## Jobs

### pre_activation

**Runs on**: `ubuntu-slim`

**Needs**: None

**Condition**: Always runs

**Permissions**: Default

#### Steps

1. **Setup Scripts**: Installs the `gh-aw` helper scripts used by subsequent steps into `/opt/gh-aw/actions`.
2. **Check team membership for workflow**: Verifies that the actor who triggered the push has at least `write` (or `maintainer`/`admin`) access to the repository. Sets the `activated` output to `true` or `false` based on this check.

---

### activation

**Runs on**: `ubuntu-slim`

**Needs**: `pre_activation`

**Condition**: Only runs if `pre_activation` determined the workflow should be activated (`needs.pre_activation.outputs.activated == 'true'`).

**Permissions**: `contents: read`

#### Steps

1. **Setup Scripts**: Installs the `gh-aw` helper scripts used by subsequent steps.
2. **Validate COPILOT_GITHUB_TOKEN secret**: Checks that the `COPILOT_GITHUB_TOKEN` secret is present and valid. This token is required for the GitHub Copilot CLI engine to operate.
3. **Validate context variables**: Runs a script to confirm all required GitHub context variables (e.g., repository, actor, run ID) are available and well-formed.
4. **Checkout .github and .agents folders**: Uses `actions/checkout` to check out only the `.github` and `.agents` directories (sparse checkout) from the repository. This provides access to the workflow configuration without fetching the full repository.
5. **Check workflow file timestamps**: Validates that the workflow lock file has not been tampered with or run out of order by comparing file timestamps via the GitHub API.
6. **Create prompt with built-in context**: Assembles the full prompt that will be passed to the AI agent. This includes system instructions, security policies, output format guidelines, and the task-specific instructions from the `.md` source file (`workflow-docs-generator.md`).
7. **Interpolate variables and render templates**: Processes template syntax in the prompt file (e.g., `{{#runtime-import ...}}`), inlining referenced files.
8. **Substitute placeholders**: Replaces `__GH_AW_...__` placeholder tokens in the prompt with the actual GitHub context values (actor, run ID, commit SHA, etc.).
9. **Validate prompt placeholders**: Ensures no unreplaced placeholder tokens remain in the final prompt.
10. **Print prompt**: Prints a summary of the prompt (without secrets) to the workflow log for debugging.
11. **Upload prompt artifact**: Uploads the assembled prompt as a workflow artifact (only on success) for later use by the `agent` job.

---

### agent

**Runs on**: `ubuntu-latest`

**Needs**: `activation`

**Condition**: Always runs (inherits from `activation` dependency)

**Permissions**: `contents: read`, `issues: read`, `pull-requests: read`

#### Steps

1. **Setup Scripts**: Installs the `gh-aw` helper scripts.
2. **Checkout repository**: Uses `actions/checkout` to check out the full repository so the agent can read and modify files.
3. **Create gh-aw temp directory**: Creates the `/tmp/gh-aw/agent/` temporary directory used by the agent during execution.
4. **Configure Git credentials**: Sets up Git with a `github-actions[bot]` identity and authenticates using a GitHub token so the agent can commit changes.
5. **Checkout PR branch**: (Only runs if the trigger came from a pull request context.) Checks out the PR's head branch so changes are applied to the correct branch.
6. **Generate agentic run info**: Generates metadata about the current run (model to use, run ID, etc.) that is passed to the Copilot CLI.
7. **Install GitHub Copilot CLI**: Downloads and installs the GitHub Copilot CLI binary (version `0.0.419`) used to drive the AI agent.
8. **Install awf binary**: Installs the `awf` binary (version `v0.23.0`), the agent workflow framework runner.
9. **Download container images**: Pre-pulls Docker images for the agent firewall, API proxy, squid proxy, MCP gateway, GitHub MCP server, and Node.js runtime. These provide a sandboxed, network-controlled environment for the agent.
10. **Write Safe Outputs Config**: Writes the configuration file that defines which output actions (e.g., `create_pull_request`) the agent is allowed to perform and their constraints.
11. **Generate Safe Outputs MCP Server Config**: Generates the MCP (Model Context Protocol) server configuration for the safe-outputs server, which provides the agent with controlled GitHub action tools.
12. **Start Safe Outputs MCP HTTP Server**: Starts the safe-outputs MCP HTTP server that exposes permitted GitHub actions (like creating a PR) to the AI agent in a controlled way.
13. **Start MCP Gateway**: Starts the MCP Gateway service, which proxies and logs all MCP tool calls made by the agent. This provides observability and enforces the network firewall.
14. **Generate workflow overview**: Generates a short summary of the current workflow run context and injects it into the agent's working environment.
15. **Download prompt artifact**: Downloads the compiled prompt artifact produced by the `activation` job.
16. **Clean git credentials**: Removes any leftover Git credential configurations before the agent starts, ensuring a clean state.
17. **Execute GitHub Copilot CLI**: The core step — runs the Copilot CLI agent with the assembled prompt. The agent reads the changed workflow files, analyzes their structure, and creates or updates documentation files in `docs/`. All GitHub API interactions go through the safe-outputs MCP server.
18. **Configure Git credentials** (post-agent): Re-configures Git credentials after the agent run to allow the agent's file changes to be committed and pushed if needed.
19. **Copy Copilot session state files to logs**: (Always runs.) Copies the Copilot CLI session logs to the artifacts staging area for upload.
20. **Stop MCP Gateway**: (Always runs.) Shuts down the MCP Gateway service and saves its logs.
21. **Redact secrets in logs**: (Always runs.) Scans log files and redacts any secret values before they are uploaded as artifacts.
22. **Upload Safe Outputs**: (Always runs.) Uploads the safe-outputs JSONL file (containing the agent's requested actions) as an artifact for the `safe_outputs` job to process.
23. **Ingest agent output**: (Always runs.) Reads and parses the agent's output (tool calls and results) into a structured format.
24. **Upload sanitized agent output**: (Always runs, if agent output exists.) Uploads a sanitized copy of the agent's output as an artifact.
25. **Upload engine output files**: Uploads the raw Copilot CLI engine output files as artifacts.
26. **Parse agent logs for step summary**: (Always runs.) Parses the agent logs and writes a human-readable summary to the GitHub Actions step summary page.
27. **Parse MCP Gateway logs for step summary**: (Always runs.) Parses MCP Gateway logs and adds tool-call details to the step summary.
28. **Print firewall logs**: (Always runs.) Prints the network firewall logs to the workflow log for auditing.
29. **Upload agent artifacts**: (Always runs.) Bundles and uploads all agent-related log files as a single artifact.
30. **Check if detection needed**: (Always runs.) Determines whether a threat-detection pass should be run on the agent's output (e.g., if the agent produced file changes).
31. **Clear MCP configuration for detection**: (Always runs, if detection is needed.) Resets the MCP configuration before running the threat-detection agent so it uses a minimal toolset.
32. **Prepare threat detection files**: (Always runs, if detection is needed.) Stages the agent output files for analysis by the threat-detection agent.
33. **Setup threat detection**: (Always runs, if detection is needed.) Writes the threat-detection prompt and configuration.
34. **Ensure threat-detection directory and log**: (Always runs, if detection is needed.) Creates the threat-detection working directory and log file.
35. **Execute GitHub Copilot CLI** (threat detection): (Always runs, if detection is needed.) Runs a second Copilot CLI pass whose sole job is to analyze the agent's output for prompt injection, data exfiltration attempts, or other security violations.
36. **Parse threat detection results**: (Always runs, if detection is needed.) Reads the threat-detection agent's verdict and outputs a pass/fail conclusion.
37. **Upload threat detection log**: (Always runs, if detection is needed.) Uploads the threat-detection log as an artifact.
38. **Set detection conclusion**: (Always runs.) Finalizes the detection result (`success` or `failure`) and makes it available as a job output.

---

### safe_outputs

**Runs on**: `ubuntu-slim`

**Needs**: `activation`, `agent`

**Condition**: Runs if the workflow was not cancelled, the `agent` job was not skipped, and threat detection passed (`needs.agent.outputs.detection_success == 'true'`).

**Permissions**: `contents: write`, `issues: write`, `pull-requests: write`

#### Steps

1. **Setup Scripts**: Installs the `gh-aw` helper scripts.
2. **Download agent output artifact**: Downloads the agent's output artifact containing requested GitHub actions (e.g., create a PR).
3. **Setup agent output environment variable**: Locates the agent output JSON file and sets the `GH_AW_AGENT_OUTPUT` environment variable pointing to it.
4. **Download patch artifact**: Downloads any file-change patch artifact produced by the agent.
5. **Checkout repository**: (Only runs if the agent requested a `create_pull_request` action.) Checks out the repository's base branch so a new PR branch can be created.
6. **Configure Git credentials**: (Only runs if creating a PR.) Configures Git with the `github-actions[bot]` identity and authentication token.
7. **Process Safe Outputs**: The main step — reads the agent's requested actions and executes them against the GitHub API. For `create_pull_request`, this applies the agent's file changes as a patch, pushes a new branch, and opens a PR with the `documentation` label and a `[docs]` title prefix.
8. **Upload safe output items manifest**: (Always runs.) Uploads a manifest of all processed safe-output items as an artifact.

---

### conclusion

**Runs on**: `ubuntu-slim`

**Needs**: `activation`, `agent`, `safe_outputs`

**Condition**: Always runs (as long as the `agent` job was not skipped).

**Permissions**: `contents: write`, `issues: write`, `pull-requests: write`

#### Steps

1. **Setup Scripts**: Installs the `gh-aw` helper scripts.
2. **Download agent output artifact**: Downloads the agent's output artifact for post-processing.
3. **Setup agent output environment variable**: Sets the `GH_AW_AGENT_OUTPUT` environment variable.
4. **Process No-Op Messages**: If the agent reported a `noop` (no action needed), this step records and surfaces that message in the workflow summary.
5. **Record Missing Tool**: If the agent reported a `missing_tool` (a capability it needed but didn't have), this step records and surfaces that information.
6. **Handle Agent Failure**: If the `agent` job failed, this step creates a summary or notification explaining the failure and linking to the workflow run.

---

### pre_activation *(runs first, before all other jobs)*

**Runs on**: `ubuntu-slim`

**Needs**: None

**Condition**: Always runs

**Permissions**: Default

> See the [pre_activation](#pre_activation) job described above — this job is defined after the others in the YAML but runs first as it has no dependencies.
