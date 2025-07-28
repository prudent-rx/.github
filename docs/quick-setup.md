# Quick Setup Guide

## Prerequisites

- Azure DevOps organization and project
- GitHub repository with workflows enabled
- Admin access to configure secrets

## 5-Minute Setup

### 1. Create Azure DevOps Personal Access Token

1. Go to Azure DevOps → Click your profile → Personal Access Tokens
2. Click "New Token"
3. Configure:
   - **Name**: `GitHub-ADO-Integration`
   - **Expiration**: 90 days (or your organization's policy)
   - **Scopes**: Custom defined → Work Items (Read & write)
4. Copy the token (you won't see it again!)

### 2. Configure GitHub Secrets

Navigate to your repository or organization settings:

**Repository Level**: `Settings` → `Secrets and variables` → `Actions`
**Organization Level**: `Settings` → `Secrets and variables` → `Actions`

Add these secrets:

| Secret Name | Value | Example |
|-------------|--------|---------|
| `ADO_ORG_URL` | Your Azure DevOps URL | `https://dev.azure.com/myorg` |
| `ADO_PAT` | Token from step 1 | `abcdef123456...` |
| `ADO_PROJECT` | Your project name | `MyProject` |

### 3. Add Workflows to Repository

Copy the workflow files to `.github/workflows/` in your repository:
- `update-ado-on-commit.yml`
- `update-ado-on-pr.yml` 
- `update-ado-workitems.yml`

### 4. Test the Integration

1. Create a test work item in Azure DevOps (note the ID)
2. Create a feature branch: `git checkout -b test/integration`
3. Make a commit: `git commit -m "Test integration AB#<work-item-id>"`
4. Push the branch: `git push origin test/integration`
5. Check GitHub Actions for workflow execution
6. Verify work item state changed to "Active" in Azure DevOps

## Common Issues & Quick Fixes

### ❌ "Missing required secrets"
**Fix**: Verify secret names are exactly: `ADO_ORG_URL`, `ADO_PAT`, `ADO_PROJECT`

### ❌ "Work item not found"
**Fix**: Ensure work item ID exists and PAT has access to the project

### ❌ "Unauthorized"
**Fix**: Check PAT permissions include "Work Items (Read & write)"

### ❌ "Work item not updating"
**Fix**: Verify work item reference format: `AB#1234` (case sensitive)

### ❌ "Workflow not triggering"
**Fix**: Check branch names - commits to protected branches don't trigger the workflow

## Work Item Reference Examples

✅ **Correct formats**:
```
Fix login bug AB#1234
Implement feature AB#5678 and AB#9012
Update documentation AB#3456
```

❌ **Incorrect formats**:
```
Fix login bug ab#1234  (lowercase 'ab')
Implement feature #5678  (missing 'AB')
Update docs 1234  (no pattern)
```

## Quick Troubleshooting

1. **Check GitHub Actions logs** - Go to Actions tab in your repository
2. **Verify secrets** - Ensure all three secrets are configured correctly
3. **Test PAT manually** - Try accessing Azure DevOps API with your PAT
4. **Check work item state** - Some states are protected from changes
5. **Verify branch** - Ensure you're not committing to protected branches

## Support

- **Detailed Documentation**: See `/docs/` folder for comprehensive guides
- **GitHub Issues**: Report problems in this repository
- **Azure DevOps**: Check work item permissions and project access