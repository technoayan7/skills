---
name: pr-desc-updater
description: >
  Generates and updates rich, descriptive Pull Request titles and bodies on GitHub.
  Analyzes git diffs, branch names, and commit history to formulate titles in the format
  "Feature/{ticket-id} - {title}", Jira ticket links, CMR ticket placeholders, and detailed feature summaries.
---

# PR Description & Title Updater Skill

## OBSERVATION — On Invocation (run immediately)

```bash
for _AGENT_DIR in "$HOME/.claude" "$HOME/.gemini" "$HOME/.agents" "$HOME/.codex"; do
  _OBSERVATION_SKILL="$_AGENT_DIR/skills/observation/SKILL.md"
  [ -f "$_OBSERVATION_SKILL" ] && break; _OBSERVATION_SKILL=""
done
if [ -n "$_OBSERVATION_SKILL" ]; then
  export _T_EVENT_ID=$(python3 -c "import uuid; print(uuid.uuid4())")
  export _T_TS=$(date -u +"%Y-%m-%d %H:%M:%S")
  export _T_USER=$(whoami); export _T_HOST=$(hostname); export _T_PLATFORM=$(uname -s)
  export _T_INPUT_SUMMARY="pr-desc-updater invoked"; export _T_OUTPUT_SUMMARY="pr-desc-updater completed"; export _T_STATUS="success"
  _T_PAYLOAD=$(python3 - <<'PY'
import json, os
print(json.dumps({"event_id":os.environ.get("_T_EVENT_ID",""),"ts":os.environ.get("_T_TS",""),"username":os.environ.get("_T_USER",""),"hostname":os.environ.get("_T_HOST",""),"platform":os.environ.get("_T_PLATFORM",""),"tool_name":"pr-desc-updater","tool_input_summary":os.environ.get("_T_INPUT_SUMMARY",""),"tool_output_summary":os.environ.get("_T_OUTPUT_SUMMARY",""),"status":os.environ.get("_T_STATUS","success"),"error_type":"","error_message":"","stack_trace":"","duration_ms":0,"goose_session_id":os.environ.get("GOOSE_SESSION_ID","")},separators=(",",":")))
PY
)
  curl -s -o /dev/null -X POST "https://regression.qa.cashfree.net/testgenerationsvc/v1/metrics/tool-event" \
    -H "Content-Type: application/json" --data-binary "$_T_PAYLOAD"
fi
```


## Overview

This skill automates inspecting Pull Requests, extracting technical & feature changes from git diffs, formatting clean PR titles (`Feature/{ticket-id} - {title}`), automatically linking Jira tickets, including CMR ticket placeholders, and updating GitHub PR descriptions via `gh CLI`.

---

## Trigger Scenarios

Use this skill when the user asks to:
- "Update PR description and title"
- "Enrich PR description for PR <number>"
- "Add detailed feature changes to PR <url>"
- "Format PR title with ticket ID and Jira link"
- Runs `/pr-desc-updater` or asks to prepare PR release notes.

---

## Workflow Steps

### Step 1: Identify Target PR & Repository

1. If a PR URL or PR number is provided in the request (e.g., `https://github.com/cashfree-tech/payoutbankvalidationsvc/pull/339` or `339`), extract the **PR Number** and **Repo**.
2. If no PR is specified, inspect the current workspace git branch:
   ```bash
   git branch --show-current
   ```
   And look up the associated open PR using:
   ```bash
   gh pr view --json number,title,body,headRefName
   ```

### Step 2: Extract Ticket ID & Branch Context

1. Parse the head branch name (e.g., `feature/VRS-16602`, `feature/VRS-16369`, `fix/DEVO-4306`).
2. Extract the Ticket ID (e.g., `VRS-16602`).
3. Construct the official Jira link: `https://cashfree.atlassian.net/browse/{TICKET_ID}`.

### Step 3: Analyze Code Changes & Diff

1. Retrieve full commit history and diff for the PR:
   ```bash
   gh pr view <PR_NUMBER> --json commits,files,title,body
   gh pr diff <PR_NUMBER>
   ```
2. Analyze modified files and group changes logically:
   - **Data Models & Schemas**: Struct additions, JSON tags, enum definitions.
   - **Gateway & Integration Layer**: Network repositories, HTTP clients, vendor APIs.
   - **Business Logic & Service Layer**: Request handlers, response builders, configuration maps.
   - **Configuration & Infrastructure**: YAML configs, ESL manifests, environment settings.
   - **Testing & Tooling**: Unit test coverage, mocks, test tools.

### Step 4: Formulate Standardized PR Title

Construct the PR Title strictly in the required format:
`Feature/{ticket-id}: {Descriptive Summary of Changes}`

**Examples**:
- `Feature/VRS-16602: Mobile360 Risk Intelligence Multi-Source Support (MNRL, FRI, Internal)`
- `Feature/VRS-16369: Refactored Risk Intelligence with Multi-Source Support`

### Step 5: Draft Comprehensive PR Description

Format the PR body in GitHub-Flavored Markdown using the template below:

```markdown
## Overview
<Concise 1-2 sentence high-level summary of what this PR accomplishes and the business problem it solves.>

## Ticket & Tracking Links
- **Jira Ticket**: [{TICKET_ID}](https://cashfree.atlassian.net/browse/{TICKET_ID})
- **CMR Ticket**: <!-- Paste CMR link here e.g. [CMR-XXXX](link) -->

## Detailed Feature Changes
- **Data Models & Enums**: <Details on new structs, enum types, severity logic, etc.>
- **Gateway & Repository Layer**: <Details on new HTTP repos, interfaces, vendor calls.>
- **Response Builders & Service Logic**: <Details on builder methods, field filtering, fallback logic.>
- **Configuration & Infrastructure**: <Details on config map keys, ESL manifests, timeouts.>
- **Testing & Tooling**: <Details on test additions, mock updates, gateway test tools.>

## Impacted Components & Services
- `payoutbankvalidationsvc` (`risk`, `mobile360`)
- `gatewayTestTool`
- `configmap` / `environments.esl`

## Type of Change
- [x] New feature (non-breaking change adding functionality)
- [x] Refactoring (code layout and architectural improvements)
- [ ] Bug fix (non-breaking change fixing an issue)

## Deployment Risk & Rollback Strategy
- **Risk Assessment**: Low (Backward-compatible API additions and config mappings).
- **Rollback Strategy**: Revert PR commit or disable corresponding VRS feature flag in merchant configuration.

## How Has This Been Tested?
- **Unit Tests**: <Describe unit test coverage & test commands run.>
- **Integration / Tool Verification**: <Describe verification in gateway test tool or staging environment.>

## Checklist
- [x] Performed self-review of code changes.
- [x] Attached / linked Jira ticket.
- [x] Added CMR ticket placeholder for release deployment.
- [x] Added proper logging and error handling.
- [x] Verified existing and updated tests pass.
```

### Step 6: Apply Updates to GitHub

1. Write the drafted PR description to a scratch file (e.g., `<appDataDir>/brain/<conversation-id>/scratch/pr_body.md`).
2. Execute `gh pr edit` to update both title and description:
   ```bash
   gh pr edit <PR_NUMBER> --title "<FORMULATED_TITLE>" --body-file "<PATH_TO_SCRATCH_FILE>"
   ```

### Step 7: Present Results to User

Summarize the actions taken, displaying:
- The updated **PR Title**
- The GitHub **PR URL**
- Jira & CMR ticket link placeholders
- A preview of the **Detailed PR Description** added to GitHub.
