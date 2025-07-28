# Update ADO on Commit Workflow

## Overview

The `update-ado-on-commit.yml` workflow automatically updates Azure DevOps work items when commits are pushed to feature branches. It moves referenced work items to the 'Active' state, indicating that development work is in progress.

## Workflow Configuration

```yaml
name: Update Azure DevOps Work Items on Commit
on:
  push:
    branches-ignore:
      - 'main'
      - 'develop'
      - 'qa'
      - 'uat'
      - 'staging'
```

## Triggers

- **Event**: Push events
- **Branches**: All branches except the protected branches listed above
- **Reason for exclusion**: Protected branches typically receive merges rather than direct commits, and different state logic applies

## Behavior

### Commit Uniqueness Check

The workflow includes logic to determine if a commit is unique to the current branch:

1. **Fetches latest refs** from all remote branches
2. **Compares against common source branches** (main, master, develop, staging)
3. **Checks if commit exists** in other branches using `git merge-base`
4. **Only processes unique commits** to prevent state transitions from inherited commits

### Work Item State Updates

- **Target State**: `Active`
- **Pattern Recognition**: Extracts work item IDs using pattern `AB#<number>`
- **State Protection**: Skips work items already in closed states (`Closed`, `Done`, `Completed`, `Removed`)

### Example Scenarios

#### Scenario 1: New Feature Branch
```bash
# Create feature branch from main
git checkout -b feature/user-login

# Make commit with work item reference
git commit -m "Start implementing user login AB#1234"
git push origin feature/user-login
```
**Result**: Work item #1234 moves to 'Active' state

#### Scenario 2: Existing Branch with New Commit
```bash
# On existing feature branch
git commit -m "Fix validation logic AB#1234"
git push origin feature/user-login
```
**Result**: Work item #1234 state is checked and updated if necessary

#### Scenario 3: Merged Commit (Inherited)
```bash
# Merge main into feature branch
git merge main
git push origin feature/user-login
```
**Result**: No work item updates (commits from main are not unique to feature branch)

## Input Parameters

The workflow calls the reusable `update-ado-workitems.yml` with:

| Parameter | Source | Description |
|-----------|--------|-------------|
| `trigger_type` | Static: `'commit'` | Indicates this is a commit-triggered update |
| `target_state` | Static: `'Active'` | Target state for work items |
| `branch_name` | `github.ref_name` | Current branch name |
| `commit_sha` | `github.sha` | SHA of the triggering commit |
| `commit_before_sha` | `github.event.before` | SHA before the push (for comparison) |
| `commit_messages` | `github.event.commits[*].message` | Array of commit messages in the push |

## Error Handling

### Common Failure Cases

1. **Missing Secrets**: Workflow logs error if ADO credentials are not configured
2. **Invalid Work Item ID**: Skips invalid IDs and continues processing others
3. **ADO API Errors**: Logs detailed error messages for debugging
4. **Git Operation Failures**: Falls back to treating commits as unique for safety

### Logging Examples

```
🚀 Starting Azure DevOps work item update process for commit...
📋 Branch: feature/user-login
📋 Before SHA: abc123...
📋 After SHA: def456...
🔍 Checking if commit def456 is unique to branch 'feature/user-login'...
✅ Commit def456 appears to be unique to branch 'feature/user-login'
🔍 Commit 1: "Implement user login AB#1234" contains work items: 1234
📋 Total unique work items to process: 1 (1234)
📡 Making GET request to fetch current state of work item 1234...
📋 Work item 1234 current state: 'To Do'
🔄 Updating work item 1234 from 'To Do' to 'Active'...
✅ SUCCESS: Work item 1234 updated to 'Active' state.
```

## Best Practices

### Commit Message Format
```
<type>: <description> AB#<work-item-id>

Examples:
feat: Add user authentication AB#1234
fix: Resolve login timeout issue AB#5678
refactor: Improve error handling AB#9012 AB#3456
```

### Branch Naming
Use descriptive branch names that indicate the work being done:
```
feature/user-authentication-AB1234
bugfix/login-timeout-AB5678
hotfix/security-patch-AB9012
```

### Multiple Work Items
Reference multiple work items in a single commit:
```
git commit -m "Implement user auth and fix login AB#1234 AB#5678"
```

## Security Considerations

- **Secret Access**: Uses inherited secrets from calling workflow
- **Git Operations**: Limited to read-only operations for safety
- **Work Item Validation**: Validates work item existence before updates
- **Error Boundaries**: Failures in one work item don't affect others

## Integration with Other Workflows

This workflow is designed to work in conjunction with:
- **PR Merge Workflow**: Handles final state transitions when work is completed
- **Branch Protection Rules**: Respects protected branches to avoid conflicts
- **Code Review Process**: Activates work items when development begins

## Troubleshooting

### Work Item Not Activated

1. **Check commit message format**: Ensure `AB#<number>` pattern is used
2. **Verify branch**: Confirm you're not on a protected branch
3. **Check work item state**: Work items in closed states are protected
4. **Review logs**: Check GitHub Actions logs for detailed information

### Unexpected State Changes

1. **Commit uniqueness**: Only unique commits trigger updates
2. **Multiple references**: Each unique work item ID is processed once
3. **State protection**: Closed work items are never moved to Active

### Performance Considerations

- **Branch checking**: Limited to first 10 branches to avoid timeouts
- **Common branch priority**: Checks main/develop branches first
- **Batch processing**: Processes multiple work items in sequence
- **Error recovery**: Continues processing remaining items if one fails