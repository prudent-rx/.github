# Update ADO Work Items (Reusable Workflow)

## Overview

The `update-ado-workitems.yml` is a reusable workflow that contains the core logic for updating Azure DevOps work items. It's called by both the commit and PR merge workflows to handle the actual integration with Azure DevOps.

## Workflow Type

**Reusable Workflow** (`workflow_call`)

This workflow is designed to be called by other workflows, not triggered directly by GitHub events.

## Input Parameters

### Required Inputs

| Parameter | Type | Description |
|-----------|------|-------------|
| `trigger_type` | string | Type of trigger: `"commit"` or `"pr_merge"` |
| `branch_name` | string | Branch name for context |

### Optional Inputs (Context Dependent)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `target_state` | string | `''` | Target state for work items (optional for pr_merge) |
| `commit_sha` | string | `''` | Commit SHA (required for commit trigger) |
| `commit_before_sha` | string | `''` | Before commit SHA (required for commit trigger) |
| `commit_messages` | string | `'[]'` | JSON array of commit messages (required for commit trigger) |
| `pr_title` | string | `''` | PR title (required for pr_merge trigger) |
| `pr_body` | string | `''` | PR body (required for pr_merge trigger) |
| `pr_base_branch` | string | `''` | PR base branch (required for pr_merge trigger) |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `ADO_ORG_URL` | Optional* | Azure DevOps organization URL |
| `ADO_PAT` | Optional* | Azure DevOps Personal Access Token |
| `ADO_PROJECT` | Optional* | Azure DevOps project name |

*Secrets are optional when passed from calling workflow, but the workflow will attempt to access them directly if not provided.

## Core Functions

### 1. Work Item ID Extraction

```javascript
function extractWorkItemIds(text) {
  if (!text) return [];
  const patterns = [
    /\bAB#(\d+)\b/g
  ];
  const workItemIds = new Set();
  patterns.forEach((pattern) => {
    let match;
    pattern.lastIndex = 0;
    while ((match = pattern.exec(text)) !== null) {
      workItemIds.add(parseInt(match[1]));
    }
  });
  return Array.from(workItemIds);
}
```

**Supported Patterns**:
- `AB#1234` - Standard work item reference format

### 2. Azure DevOps API Integration

#### Get Work Item Current State
```javascript
async function makeRequest(url, options, data = null) {
  // Makes HTTPS requests to Azure DevOps REST API
  // Handles authentication with Basic Auth using PAT
  // Returns JSON response with status code
}
```

#### Update Work Item State
```javascript
async function updateWorkItemState(workItemId, newState) {
  // 1. Fetch current work item state
  // 2. Check if update is needed
  // 3. Apply state protection logic
  // 4. Update work item via PATCH request
  // 5. Verify update success
}
```

### 3. Commit Uniqueness Detection

```javascript
async function checkIfCommitIsUniqueToBranch(branchName, commitSha) {
  // 1. Fetch latest refs from remote
  // 2. Get list of all remote branches
  // 3. Check against common source branches first
  // 4. Use git merge-base to determine ancestry
  // 5. Return true if commit is unique to current branch
}
```

## State Protection Logic

### Closed State Protection

The workflow protects work items in the following states from being modified:
- `Closed`
- `Done` 
- `Completed`
- `Removed`

### Trigger-Specific Logic

#### Commit Trigger Protection
```javascript
// For commit trigger moving to 'Active', protect closed states
if (process.env.TRIGGER_TYPE === 'commit' && newState === 'Active' && closedStates.includes(currentState)) {
  console.log(`🛡️ SKIPPED: Work item ${workItemId} is in closed state '${currentState}'. Will not move back to 'Active'.`);
  return { success: true, reason: `Protected closed state '${currentState}'` };
}
```

#### PR Merge Trigger Protection
```javascript
// For PR merge trigger, protect truly closed states
if (process.env.TRIGGER_TYPE === 'pr_merge' && closedStates.includes(currentState)) {
  console.log(`🛡️ SKIPPED: Work item ${workItemId} is in protected state '${currentState}'. Will not move to '${newState}'.`);
  return { success: false, reason: `Protected state '${currentState}'` };
}
```

## Branch-Based State Mapping

For PR merge triggers, the target state is determined automatically:

```javascript
let targetState = process.env.TARGET_STATE;
if (!targetState) {
  targetState = 'Todo'; // default state
  
  if (baseBranch === 'develop' || baseBranch === 'dev' || baseBranch === 'qa') {
    targetState = 'Resolved';
  } else if (baseBranch === 'uat' || baseBranch === 'staging') {
    targetState = 'Signoff';
  } else if (baseBranch === 'main' || baseBranch === 'master') {
    targetState = 'Closed';
  }
}
```

## Error Handling and Logging

### Comprehensive Logging

The workflow provides detailed logging for troubleshooting:

```
🚀 Starting Azure DevOps work item update process...
📋 Branch: feature/user-auth
📋 Total unique work items to process: 2 (1234, 5678)
🎯 Target state: 'Active'

--- Processing work item 1234 ---
📡 Making GET request to fetch current state...
📋 Work item 1234 current state: 'To Do'
🔄 Updating work item 1234 from 'To Do' to 'Active'...
✅ SUCCESS: Work item 1234 updated to 'Active' state.

📊 SUMMARY:
✅ Successfully processed: 1 work items
🛡️ Skipped (protected state): 0 work items
❌ Failed with errors: 0 work items
```

### Error Recovery

- **Individual Failures**: Failed work items don't stop processing of others
- **Network Errors**: Detailed error messages with status codes
- **Authentication Issues**: Clear messaging about missing secrets
- **Git Operation Failures**: Safe defaults to avoid blocking workflows

## Performance Optimizations

### Branch Checking Limits
```javascript
// Limit branch checks to avoid excessive API calls
for (const otherBranch of allBranches.slice(0, 10)) {
  // Check only first 10 branches for performance
}
```

### Common Branch Priority
```javascript
// Check common source branches first for efficiency
const commonSourceBranches = ['main', 'master', 'develop', 'staging'].filter(branch => 
  allBranches.includes(branch)
);
```

### Request Optimization
- **Minimal API Calls**: Only necessary Azure DevOps API requests
- **Efficient JSON Parsing**: Safe JSON parsing with error handling
- **Connection Reuse**: HTTPS connections managed efficiently

## Security Features

### Authentication
```javascript
const auth = Buffer.from(`:${pat}`).toString('base64');
const options = {
  headers: {
    'Authorization': `Basic ${auth}`,
    'Content-Type': 'application/json'
  }
};
```

### Secret Management
- **Environment Variables**: Secrets accessed via environment variables
- **No Logging**: Secrets are never logged or exposed
- **Optional Secrets**: Supports both passed and directly accessed secrets

### Input Validation
- **Work Item ID Validation**: Ensures numeric work item IDs
- **Branch Name Sanitization**: Safe handling of branch names
- **JSON Parsing**: Safe parsing of input parameters

## Extensibility

### Adding New Work Item Patterns

To support additional work item reference patterns:

```javascript
const patterns = [
  /\bAB#(\d+)\b/g,           // Current: AB#1234
  /\bTASK-(\d+)\b/g,         // New: TASK-1234
  /\bUSER-STORY-(\d+)\b/g,   // New: USER-STORY-1234
  /\b#(\d+)\b/g              // New: #1234
];
```

### Custom State Mappings

Modify the branch-to-state mapping:

```javascript
// Add new branch patterns
if (baseBranch === 'develop' || baseBranch === 'dev' || baseBranch === 'qa') {
  targetState = 'Resolved';
} else if (baseBranch === 'uat' || baseBranch === 'staging' || baseBranch === 'preprod') {
  targetState = 'Signoff';
} else if (baseBranch === 'main' || baseBranch === 'master' || baseBranch === 'production') {
  targetState = 'Closed';
}
```

### Additional Trigger Types

To support new trigger types:

```javascript
// Add new trigger type
if (triggerType === 'commit') {
  await processCommitTrigger();
} else if (triggerType === 'pr_merge') {
  await processPRMergeTrigger();
} else if (triggerType === 'manual') {
  await processManualTrigger();
} else {
  console.log(`❌ ERROR: Invalid trigger type '${triggerType}'.`);
}
```

## API Reference

### Azure DevOps REST API Endpoints

#### Get Work Item
```
GET https://dev.azure.com/{organization}/{project}/_apis/wit/workitems/{id}?api-version=7.0
```

#### Update Work Item
```
PATCH https://dev.azure.com/{organization}/{project}/_apis/wit/workitems/{id}?api-version=7.0
Content-Type: application/json-patch+json

[
  {
    "op": "add",
    "path": "/fields/System.State",
    "value": "Active"
  }
]
```

### Git Commands Used

```bash
# Fetch latest references
git fetch --all

# List remote branches
git branch -r

# Check commit ancestry
git merge-base --is-ancestor <commit> <branch>
```

## Testing and Validation

### Unit Testing Approach

While this workflow doesn't include formal unit tests, you can test it by:

1. **Mock Azure DevOps Project**: Create a test project with test work items
2. **Test Repository**: Use a dedicated test repository
3. **Secret Configuration**: Use test credentials with limited permissions
4. **Workflow Validation**: Test each trigger type and state transition

### Integration Testing

```yaml
# Example test workflow
name: Test ADO Integration
on:
  workflow_dispatch:
    inputs:
      test_work_item:
        description: 'Test work item ID'
        required: true

jobs:
  test-commit-trigger:
    uses: ./.github/workflows/update-ado-workitems.yml
    with:
      trigger_type: 'commit'
      target_state: 'Active'
      branch_name: 'test-branch'
      commit_sha: 'test-sha'
      commit_messages: '["Test commit AB#${{ github.event.inputs.test_work_item }}"]'
    secrets:
      ADO_ORG_URL: ${{ secrets.TEST_ADO_ORG_URL }}
      ADO_PAT: ${{ secrets.TEST_ADO_PAT }}
      ADO_PROJECT: ${{ secrets.TEST_ADO_PROJECT }}
```

## Maintenance

### Regular Updates
- **Azure DevOps API**: Monitor for API version updates
- **GitHub Actions**: Keep action versions current
- **Dependencies**: Update Node.js built-in modules if needed

### Monitoring
- **Workflow Runs**: Regular review of workflow execution logs
- **Error Patterns**: Monitor for recurring errors
- **Performance**: Track execution times and API response times

### Documentation
- **API Changes**: Update documentation for Azure DevOps API changes
- **State Mappings**: Document any custom state mappings
- **Pattern Updates**: Update documentation when adding new work item patterns