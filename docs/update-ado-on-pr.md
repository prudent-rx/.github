# Update ADO on PR Merge Workflow

## Overview

The `update-ado-on-pr.yml` workflow automatically updates Azure DevOps work items when pull requests are merged. It moves referenced work items to different states based on the target branch, reflecting the progression of work through different environments.

## Workflow Configuration

```yaml
name: Update Azure DevOps Work Items on PR Merge
on:
  pull_request:
    types: [closed]
```

## Triggers

- **Event**: Pull request closed
- **Condition**: Only runs when `github.event.pull_request.merged == true`
- **All Branches**: No branch restrictions (behavior depends on target branch)

## State Transition Logic

The workflow determines the target state based on the pull request's base branch:

| Base Branch | Target State | Environment | Purpose |
|-------------|--------------|-------------|---------|
| `develop`, `qa` | `Resolved` | Development/QA | Code ready for testing |
| `uat`, `staging` | `Signoff` | User Acceptance | Ready for user acceptance testing |
| `main`, `master` | `Closed` | Production | Work completed and deployed |
| Other branches | `Todo` | Default | Fallback state |

## Behavior

### Work Item Extraction

The workflow extracts work item IDs from two sources:

1. **PR Title**: Scans the pull request title for `AB#<number>` patterns
2. **PR Description**: Scans the pull request body/description for the same patterns

### State Protection

- **Closed State Protection**: Work items already in closed states (`Closed`, `Done`, `Completed`, `Removed`) are not modified
- **Logging**: Provides detailed logging showing which work items are found and processed

### Example Scenarios

#### Scenario 1: Feature Merge to Develop
```markdown
**PR Title**: Implement user authentication AB#1234
**Base Branch**: develop
**Description**: 
This PR adds user authentication functionality.
Additional work items: AB#5678
```
**Result**: Work items #1234 and #5678 move to 'Resolved' state

#### Scenario 2: Release to Production
```markdown
**PR Title**: Release v2.1.0 AB#1111 AB#2222
**Base Branch**: main
**Description**: Production release containing multiple features
```
**Result**: Work items #1111 and #2222 move to 'Closed' state

#### Scenario 3: Hotfix to UAT
```markdown
**PR Title**: Fix critical security issue AB#9999
**Base Branch**: uat
**Description**: Emergency security patch
```
**Result**: Work item #9999 moves to 'Signoff' state

## Input Parameters

The workflow calls the reusable `update-ado-workitems.yml` with:

| Parameter | Source | Description |
|-----------|--------|-------------|
| `trigger_type` | Static: `'pr_merge'` | Indicates this is a PR merge update |
| `branch_name` | `github.event.pull_request.base.ref` | Target branch name |
| `pr_title` | `github.event.pull_request.title` | Pull request title |
| `pr_body` | `github.event.pull_request.body` | Pull request description |
| `pr_base_branch` | `github.event.pull_request.base.ref` | Base branch (duplicate for clarity) |

## Error Handling

### Common Failure Cases

1. **Missing Secrets**: Workflow logs error if ADO credentials are not configured
2. **Invalid Work Item ID**: Skips invalid IDs and continues processing others
3. **Protected States**: Logs when work items are in protected states
4. **ADO API Errors**: Provides detailed error messages with status codes

### Logging Examples

```
🚀 Starting Azure DevOps work item update process for PR merge...
📄 PR Title: "Implement user authentication AB#1234"
📄 PR Description: "This PR adds user authentication functionality..."
🔍 Work items found in PR title: 1234
🔍 Work items found in PR description: none
📋 Total unique work items to process: 1 (1234)
🎯 Target branch: 'develop' → Target work item state: 'Resolved'

--- Processing work item 1234 ---
📡 Making GET request to fetch current state of work item 1234...
📋 Work item 1234 current state: 'Active'
🔄 Updating work item 1234 from 'Active' to 'Resolved'...
✅ SUCCESS: Work item 1234 updated to 'Resolved' state.

📊 SUMMARY:
✅ Successfully processed: 1 work items
🛡️ Skipped (protected state): 0 work items
❌ Failed with errors: 0 work items
```

## Best Practices

### PR Title Format
Include work item references in the title for visibility:
```
feat: Add user authentication AB#1234
fix: Resolve login timeout AB#5678
refactor: Improve error handling AB#9012 AB#3456
```

### PR Description Template
Use a consistent template that includes work item references:
```markdown
## Description
Brief description of changes

## Work Items
- AB#1234: Implement user authentication
- AB#5678: Add password validation

## Testing
- [ ] Unit tests added
- [ ] Integration tests updated
- [ ] Manual testing completed

## Related Work Items
AB#1234 AB#5678
```

### Branch Strategy Alignment

Ensure your branching strategy aligns with the state transitions:

```
feature/auth → develop (Resolved)
develop → uat (Signoff)
uat → main (Closed)
```

## Advanced Configuration

### Custom State Mapping

If your Azure DevOps project uses different state names, you can modify the reusable workflow or override the target state:

```yaml
# In calling workflow (not recommended - modify reusable workflow instead)
jobs:
  update-ado-workitem-pr-merge:
    if: github.event.pull_request.merged == true
    uses: ./.github/workflows/update-ado-workitems.yml
    with:
      trigger_type: 'pr_merge'
      target_state: 'Custom State'  # Override automatic determination
      # ... other parameters
```

### Multiple Work Item Patterns

The workflow currently supports the `AB#<number>` pattern. To support additional patterns, modify the `extractWorkItemIds` function in the reusable workflow:

```javascript
// Add new patterns to the array
const patterns = [
  /\bAB#(\d+)\b/g,
  /\bTASK-(\d+)\b/g,    // Example: TASK-1234
  /\bWI(\d+)\b/g        // Example: WI1234
];
```

## Integration Scenarios

### Feature Development Workflow

1. **Feature Branch Creation**: Create branch from `develop`
2. **Development**: Commits move work items to 'Active' (via commit workflow)
3. **PR to Develop**: Merge moves work items to 'Resolved'
4. **PR to UAT**: Merge moves work items to 'Signoff'
5. **PR to Main**: Merge moves work items to 'Closed'

### Hotfix Workflow

1. **Hotfix Branch**: Create from `main`
2. **Development**: Commits move work items to 'Active'
3. **PR to Main**: Direct merge moves work items to 'Closed'

### Release Workflow

1. **Release Branch**: Create from `develop`
2. **Bug Fixes**: Individual commits maintain 'Active' state
3. **PR to UAT**: Moves work items to 'Signoff'
4. **PR to Main**: Final merge moves work items to 'Closed'

## Security Considerations

- **Secret Inheritance**: Uses secrets passed from organization/repository level
- **Read-Only Operations**: Only reads PR data, doesn't modify GitHub content
- **Work Item Validation**: Validates work item existence before updates
- **Error Isolation**: Failures in one work item don't affect others

## Troubleshooting

### Work Items Not Updated

1. **Check PR merge status**: Workflow only runs on actual merges, not just closes
2. **Verify work item format**: Ensure `AB#<number>` pattern in title or description
3. **Review branch mapping**: Confirm target branch has defined state mapping
4. **Check secrets**: Ensure ADO credentials are properly configured

### Wrong Target State

1. **Branch name matching**: Verify exact branch name matches the logic
2. **Case sensitivity**: Branch names are case-sensitive in the logic
3. **Custom states**: Your ADO project might use different state names

### Protected State Issues

1. **Intentional protection**: Closed work items are protected by design
2. **Manual override**: Use Azure DevOps directly for exceptional cases
3. **State validation**: Check current state in ADO before expecting changes

### Performance Optimization

- **Batch processing**: Multiple work items are processed sequentially
- **Error recovery**: Failed items don't stop processing of remaining items
- **Efficient extraction**: Work item IDs are deduplicated before processing
- **Minimal API calls**: Only necessary ADO API calls are made