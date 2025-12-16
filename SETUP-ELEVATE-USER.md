# Setup Instructions for Elevate User Workflow

## Overview
The elevate-user workflow adds existing Azure Entra users to the CS-DEV-Admins group with required approval.

## GitHub Environment Setup

To enable the approval requirement, you need to create a GitHub environment with required reviewers:

### Step 1: Create the Environment

1. Go to your GitHub repository
2. Navigate to Settings > Environments
3. Click "New environment"
4. Name it: **user-elevation-approval**
5. Click "Configure environment"

### Step 2: Configure Required Reviewers

1. Check "Required reviewers"
2. Add the GitHub usernames of people who can approve elevation requests
3. Recommended: Add at least 2 reviewers for security
4. Click "Save protection rules"

### Optional: Additional Protection Rules

You can also configure:
- **Wait timer**: Add a minimum wait time before deployment
- **Deployment branches**: Limit which branches can trigger this workflow
- **Environment secrets**: Add environment-specific secrets if needed

## How the Approval Process Works

### 1. Request Submission
- User triggers the workflow with an email address
- Workflow validates the user exists in Azure Entra
- Sends Slack notification with approval request
- Workflow pauses at "elevate-user" job

### 2. Approval Review
- Designated reviewers receive GitHub notification
- Reviewers click the Slack "Review in GitHub" button or go to Actions tab
- Review the request details
- Approve or Reject the request

### 3. Execution
- If approved: User is added to CS-DEV-Admins group
- If rejected: Workflow stops without making changes
- Completion notification sent to Slack

## Slack Notifications

The workflow sends two Slack notifications:

1. **Approval Request** (sent immediately)
   - Shows who requested elevation
   - User details and target group
   - Link to review in GitHub Actions

2. **Completion Notification** (sent after approval/rejection)
   - Final status (Elevated Successfully / Already Member / Failed)
   - User details
   - Link to workflow run

## Required Secrets

The workflow uses the same secrets as the provision-user workflow:
- AZURE_CLIENT_ID
- AZURE_TENANT_ID
- AZURE_CLIENT_SECRET
- SLACK_NOTIFY_WEBHOOK_URL

## Permissions Required

The Azure service principal needs these Microsoft Graph permissions:
- User.Read.All - To verify user exists
- Group.ReadWrite.All - To add user to group
- Directory.Read.All - To search for groups

## Usage

### Trigger the Workflow

1. Go to Actions > Elevate User to CS-DEV-Admins
2. Click "Run workflow"
3. Enter the user's email address
4. Click "Run workflow"

### Approve the Request

1. Go to Actions tab
2. Click on the running workflow
3. Click "Review deployments"
4. Select "user-elevation-approval"
5. Add optional comment
6. Click "Approve and deploy" or "Reject"

## Security Notes

- Only designated reviewers can approve elevation requests
- All elevation requests are logged in GitHub Actions history
- Slack notifications provide audit trail
- User must already exist in Azure Entra tenant
- Workflow verifies user isn't already in group before adding

## Troubleshooting

### Environment Not Found
If you see "Environment not found" error:
1. Ensure environment is named exactly: user-elevation-approval
2. Check environment is configured in repository settings

### No Reviewers Configured
If workflow proceeds without waiting:
1. Verify required reviewers are added to the environment
2. Ensure reviewers have appropriate repository access

### Approval Not Working
1. Check reviewers have notifications enabled
2. Verify reviewers have at least write access to repository
3. Try refreshing the Actions page

## Example Workflow Run

1. Alice triggers elevation for bob@example.com
2. Slack notification sent to approval channel
3. Workflow waits for approval
4. Bob (reviewer) clicks "Review in GitHub"
5. Bob approves the request
6. bob@example.com is added to CS-DEV-Admins
7. Completion notification sent to Slack
