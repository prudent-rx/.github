# Prudent-Rx GitHub Workflows

This repository contains shared GitHub workflows and templates for the Prudent-Rx organization. These workflows automate integration between GitHub and Azure DevOps (ADO) to streamline development workflows.

## Table of Contents

- [Overview](#overview)
- [Workflows](#workflows)
- [Setup and Configuration](#setup-and-configuration)
- [Usage Examples](#usage-examples)
- [State Transition Logic](#state-transition-logic)
- [Troubleshooting](#troubleshooting)
- [Documentation](#documentation)
- [Contributing](#contributing)
- [Security Considerations](#security-considerations)

## Overview

The workflows in this repository automatically update Azure DevOps work item states based on GitHub activities:

- **Commits**: Move work items to 'Active' state when unique commits are pushed to feature branches
- **Pull Request Merges**: Move work items to appropriate states based on the target branch

## Workflows

### 1. Update ADO on Commit (`update-ado-on-commit.yml`)

**Purpose**: Automatically moves referenced ADO work items to 'Active' state when commits are pushed to feature branches.

**Triggers**: 
- Push events to any branch except: `main`, `develop`, `qa`, `uat`, `staging`

**Behavior**:
- Only processes commits that are unique to the current branch (not inherited from source branches)
- Protects work items in closed states from being moved back to 'Active'
- Extracts work item IDs from commit messages using pattern `AB#<number>`

### 2. Update ADO on PR Merge (`update-ado-on-pr.yml`)

**Purpose**: Automatically moves referenced ADO work items to appropriate states when pull requests are merged.

**Triggers**:
- Pull request closed events (only when merged)

**Behavior**:
- Determines target state based on the base branch:
  - `develop`/`qa` → `Resolved`
  - `uat`/`staging` → `Signoff` 
  - `main`/`master` → `Closed`
  - Other branches → `Todo`
- Extracts work item IDs from PR title and description
- Protects work items already in closed states

### 3. Update ADO Work Items (`update-ado-workitems.yml`)

**Purpose**: Reusable workflow containing the core logic for updating Azure DevOps work items.

**Type**: Reusable workflow called by the other two workflows

**Features**:
- Handles both commit and PR merge triggers
- Fetches current work item state before updating
- Implements state protection logic
- Provides detailed logging and error handling
- Supports both repository and organizational secrets

## Setup and Configuration

### Required Secrets

Configure these secrets at the organization or repository level:

| Secret | Description | Required |
|--------|-------------|----------|
| `ADO_ORG_URL` | Azure DevOps organization URL (e.g., `https://dev.azure.com/your-org`) | Yes |
| `ADO_PAT` | Azure DevOps Personal Access Token with work item read/write permissions | Yes |
| `ADO_PROJECT` | Azure DevOps project name | Yes |

### Azure DevOps Personal Access Token Setup

1. Go to Azure DevOps → User Settings → Personal Access Tokens
2. Create a new token with the following scopes:
   - **Work Items**: Read & Write
   - **Project and Team**: Read (if using project-level PAT)
3. Copy the token and add it as `ADO_PAT` secret in GitHub

### Work Item Reference Pattern

Reference work items in commit messages or PR titles/descriptions using:
```
AB#<work-item-id>
```

**Examples**:
- `Fix login issue AB#1234`
- `Implement new feature AB#5678 AB#9012`
- `Update documentation for AB#3456`

## Usage Examples

### Commit Workflow

When you push a commit to a feature branch:

```bash
git commit -m "Fix authentication bug AB#1234"
git push origin feature/auth-fix
```

**Result**: Work item #1234 will be moved to 'Active' state (if it's not already in a closed state).

### Pull Request Workflow

When creating a PR:

```markdown
**Title**: Implement user authentication AB#1234

**Description**: 
This PR implements the new authentication system.
Related work items: AB#1234, AB#5678
```

**Result**: When merged to `develop`, work items #1234 and #5678 will be moved to 'Resolved' state.

## State Transition Logic

### Commit Trigger
- **Target State**: `Active`
- **Protection**: Work items in closed states (`Closed`, `Done`, `Completed`, `Removed`) are not moved
- **Uniqueness Check**: Only processes commits unique to the current branch to prevent circumventing other state transitions

### PR Merge Trigger
- **Target States** (based on base branch):
  - `develop`, `qa` → `Resolved`
  - `uat`, `staging` → `Signoff`
  - `main`, `master` → `Closed`
  - Other branches → `Todo`
- **Protection**: Work items in closed states are not moved to prevent data loss

## Troubleshooting

### Common Issues

#### Work items not updating
1. **Check secrets**: Ensure `ADO_ORG_URL`, `ADO_PAT`, and `ADO_PROJECT` are configured correctly
2. **Verify PAT permissions**: Token must have Work Items Read & Write access
3. **Check work item ID format**: Must use `AB#<number>` pattern
4. **Review workflow logs**: Check GitHub Actions logs for detailed error messages

#### Work items in wrong state
1. **State protection**: Closed work items are intentionally protected from updates
2. **Branch-based logic**: PR merge states depend on the target branch
3. **Commit uniqueness**: Only unique commits (not from source branch) trigger updates

#### Secrets not working
1. **Organizational vs Repository secrets**: Workflows try organizational secrets first, then repository secrets
2. **Secret names**: Must match exactly: `ADO_ORG_URL`, `ADO_PAT`, `ADO_PROJECT`
3. **PAT expiration**: Check if the Personal Access Token has expired

### Debugging Steps

1. **Check workflow runs**: Go to Actions tab in GitHub to see workflow execution logs
2. **Verify work item exists**: Ensure the referenced work item ID exists in Azure DevOps
3. **Test PAT manually**: Use the PAT with Azure DevOps REST API to verify permissions
4. **Check branch filtering**: Commit workflow only runs on non-protected branches

## Contributing

When modifying these workflows:

1. **Test thoroughly**: Use a test repository and test Azure DevOps project
2. **Maintain backward compatibility**: Existing work item references should continue working
3. **Update documentation**: Keep this README up to date with any changes
4. **Follow security practices**: Never expose secrets in logs or commit them to code

## Security Considerations

- **Secrets**: Never log or expose Azure DevOps credentials
- **Permissions**: Use principle of least privilege for PAT scopes
- **Validation**: Always validate work item IDs before processing
- **Error handling**: Fail gracefully without exposing sensitive information

## Documentation

### Quick Start
- **[Quick Setup Guide](docs/quick-setup.md)** - 5-minute setup instructions
- **[Troubleshooting](docs/quick-setup.md#quick-troubleshooting)** - Common issues and fixes

### Detailed Documentation
- **[Update ADO on Commit](docs/update-ado-on-commit.md)** - Detailed documentation for commit workflow
- **[Update ADO on PR Merge](docs/update-ado-on-pr.md)** - Detailed documentation for PR merge workflow  
- **[Update ADO Work Items (Reusable)](docs/update-ado-workitems.md)** - Core workflow documentation

---

For questions or issues with these workflows, please contact the DevOps team or create an issue in this repository.
