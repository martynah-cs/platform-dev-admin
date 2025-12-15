# Azure Entra User Provisioning GitHub Action - Specification

## Overview
A GitHub Action workflow that automates the process of provisioning users in Azure Entra (formerly Azure Active Directory) and managing their group memberships. The workflow provides an interactive way to invite users and ensure they are added to specified groups.

## Objectives
1. Provide a reusable, manual workflow for user provisioning
2. Handle both new user creation and existing user scenarios
3. Ensure idempotent operations (safe to run multiple times)
4. Provide clear feedback on actions taken
5. Maintain security best practices for credential management
6. Send Slack notifications when provisioning actions are initiated

## Workflow Trigger
- Type: workflow_dispatch (manual trigger)
- Reason: User provisioning is a deliberate administrative action that should be manually initiated

## Input Parameters

### Required Inputs
1. **user_email**
   - Type: String
   - Description: Email address of the user to provision
   - Validation: Must be a valid email format
   - Example: user@example.com

2. **entra_group_name** (or **entra_group_id**)
   - Type: String
   - Description: Name or Object ID of the Azure Entra group to add the user to
   - Example: Developers or 12345678-1234-1234-1234-123456789abc
   - Note: Group ID is preferred for reliability, but name can be used for usability

### Optional Inputs
3. **display_name**
   - Type: String
   - Description: Display name for the user (if creating new account)
   - Default: Derived from email address if not provided
   - Example: John Doe

4. **send_invitation**
   - Type: Boolean
   - Description: Whether to send an invitation email to the user
   - Default: true

5. **invitation_redirect_url**
   - Type: String
   - Description: URL to redirect user after accepting invitation
   - Default: `https://myapps.microsoft.com`

## Prerequisites and Configuration

### Azure Service Principal
The workflow requires an Azure Service Principal with the following:
- **Required API Permissions** (Microsoft Graph):
  - User.ReadWrite.All - To create and read users
  - User.Invite.All - To send invitation emails
  - Group.ReadWrite.All - To read groups and manage memberships
  - Directory.Read.All - To search for existing users

### GitHub Secrets
The following secrets must be configured in the repository:
1. **AZURE_CLIENT_ID**: Application (client) ID of the service principal
2. **AZURE_TENANT_ID**: Directory (tenant) ID
3. **AZURE_CLIENT_SECRET**: Client secret value
4. **SLACK_WEBHOOK_URL**: Slack incoming webhook URL for the notification channel

### Slack Webhook Configuration
To enable Slack notifications:
1. Create or use an existing Slack app in your workspace
2. Enable Incoming Webhooks for the app
3. Create a webhook for the designated notification channel
4. Add the webhook URL to GitHub Secrets as `SLACK_WEBHOOK_URL`
5. Ensure the Slack app has permission to post to the channel

Recommended Channel: Create a dedicated channel (e.g., #azure-user-provisioning or #platform-admin-logs) for these notifications to maintain a clear audit trail and avoid cluttering general channels.

### Environment Variables (Optional)
- **DEFAULT_ENTRA_GROUP**: Default group to add users to if not specified

## Workflow Steps

### Step 1: Send Slack Notification (Initiation)
**Purpose**: Notify the designated Slack channel that a user provisioning action has been initiated

**Actions**:
- Send POST request to Slack webhook URL from GitHub secrets
- Include the following information in the notification:
  - GitHub username of the person who triggered the workflow (github.actor variable)
  - User email address being provisioned
  - Target Azure Entra group name/ID
  - Workflow run URL for tracking progress
  - Current timestamp
- Use Slack Block Kit formatting for rich, readable messages

Slack Message Payload Example:

  {
    "text": "Azure Entra User Provisioning Initiated",
    "blocks": [
      {
        "type": "header",
        "text": {
          "type": "plain_text",
          "text": "Azure Entra User Provisioning Started"
        }
      },
      {
        "type": "section",
        "fields": [
          {
            "type": "mrkdwn",
            "text": "*Initiated By:*\n@john.doe"
          },
          {
            "type": "mrkdwn",
            "text": "*User Email:*\nuser@example.com"
          },
          {
            "type": "mrkdwn",
            "text": "*Target Group:*\nDevelopers"
          },
          {
            "type": "mrkdwn",
            "text": "*Status:*\nIn Progress"
          }
        ]
      },
      {
        "type": "context",
        "elements": [
          {
            "type": "mrkdwn",
            "text": "<https://github.com/org/repo/actions/runs/123456|View Workflow Run> | Triggered at 2025-12-15 10:30 UTC"
          }
        ]
      }
    ]
  }

**Error Handling**:
- If Slack notification fails, log a warning but continue the workflow
- Do not fail the entire provisioning process due to notification issues
- Implement retry logic (2 attempts with 5-second delay) for transient Slack API failures
- Catch and log any webhook errors without interrupting main workflow

**Timing**: This step should execute as the first step after workflow trigger, before any Azure operations

### Step 2: Authentication
**Purpose**: Authenticate to Azure using service principal credentials

**Actions**:
- Use Azure CLI or Azure PowerShell action
- Authenticate using service principal credentials from GitHub secrets
- Verify successful authentication before proceeding

**Error Handling**:
- Fail workflow if authentication fails
- Provide clear error message about credential issues

### Step 3: Validate Input Email
**Purpose**: Ensure email address is in valid format

**Actions**:
- Validate email format using regex pattern
- Normalize email address (lowercase, trim whitespace)

**Error Handling**:
- Fail workflow with clear message if email is invalid

### Step 4: Check if User Exists
**Purpose**: Determine if user already has an Azure Entra account

**Actions**:
- Query Microsoft Graph API for user by email/UPN
- Search using userPrincipalName or mail property
- Handle both internal users and guest users

**Outcomes**:
- User exists → Proceed to Step 6
- User does not exist → Proceed to Step 5

**Error Handling**:
- Handle API throttling with retry logic
- Distinguish between "user not found" and API errors

### Step 5: Create User / Send Invitation
**Purpose**: Create guest user account and optionally send invitation

**Actions**:
- Use Microsoft Graph API invitation endpoint: POST /invitations
- Create guest user with provided email address
- Set display name (from input or derived from email)
- Configure invitation message settings
- Set redirect URL for post-invitation
- Conditionally send invitation email based on input parameter

API Payload Example:

  {
    "invitedUserEmailAddress": "user@example.com",
    "invitedUserDisplayName": "John Doe",
    "sendInvitationMessage": true,
    "inviteRedirectUrl": "https://myapps.microsoft.com"
  }

**Output**:
- Capture user object ID from response
- Log invitation status
- Store invitation redemption URL

**Error Handling**:
- Handle invitation failures gracefully
- Provide specific error messages for common issues:
  - Email domain restrictions
  - Invitation quota exceeded
  - Permission issues

### Step 6: Resolve Group ID
**Purpose**: Get the Object ID of the target Azure Entra group

**Actions**:
- If group ID provided, validate it exists
- If group name provided, search for group by display name
- Use Microsoft Graph API: GET /groups?$filter=displayName eq '{name}'

**Error Handling**:
- Fail if group not found
- Fail if multiple groups found with same name (recommend using Object ID)
- Handle permission errors

### Step 7: Check Current Group Membership
**Purpose**: Determine if user is already a member of the target group

**Actions**:
- Query group membership: GET /groups/{group-id}/members
- Check if user object ID is in members list
- Alternative: Check user's group memberships

**Outcomes**:
- User is already a member → Skip to Step 9 (report no action needed)
- User is not a member → Proceed to Step 8

**Error Handling**:
- Handle large group membership lists with pagination
- Handle API errors gracefully

### Step 8: Add User to Group
**Purpose**: Add the user to the specified Azure Entra group

**Actions**:
- Use Microsoft Graph API: POST /groups/{group-id}/members/$ref
- Add user object ID as a member

API Payload Example:

  {
    "@odata.id": "https://graph.microsoft.com/v1.0/directoryObjects/{user-id}"
  }

**Error Handling**:
- Handle "member already exists" errors gracefully
- Retry on transient failures
- Provide specific error messages

### Step 9: Summary Report
**Purpose**: Provide clear feedback on actions taken

**Actions**:
- Generate workflow summary using GitHub Actions summary feature
- Include:
  - User email address
  - Whether user was newly created or already existed
  - Whether invitation was sent
  - Group membership status (added, already member)
  - User Object ID for reference
  - Invitation redemption URL (if created)

**Output Format Example**:
```
✅ User Provisioning Complete

User Email: user@example.com
User Object ID: abcd1234-5678-90ef-ghij-klmnopqrstuv
Status: Newly created guest user
Invitation Sent: Yes
Invitation URL: https://...
Group: Developers
Group Membership: Added successfully
```

## Error Handling Strategy

### Transient Errors
- API throttling (429 responses)
- Network timeouts
- Temporary service unavailability

**Handling**: Implement exponential backoff retry logic (3 attempts)

### Permanent Errors
- Invalid credentials
- Insufficient permissions
- User/group not found
- Invalid email format

**Handling**: Fail fast with clear error messages

### Partial Success Scenarios
- User created but group addition failed
- User exists but cannot be added to group

**Handling**:
- Report partial success clearly
- Provide remediation steps in output
- Do not mark workflow as completely failed if user was created successfully

## Security Considerations

1. **Credential Management**
   - Never log or expose client secrets in workflow output
   - Use GitHub encrypted secrets for all sensitive data
   - Rotate service principal credentials regularly

2. **Least Privilege**
   - Service principal should have only required permissions
   - Consider using separate service principals for different workflows

3. **Audit Trail**
   - All workflow runs are logged in GitHub Actions history
   - Include user email and action taken in workflow summary
   - Slack notifications provide real-time audit trail in designated channel
   - Never include sensitive data (passwords, tokens) in Slack messages

4. **Input Validation**
   - Validate all user inputs
   - Prevent injection attacks through email input
   - Sanitize group names if using display names

## Testing Strategy

### Unit Testing Scenarios
1. Valid email with new user
2. Valid email with existing user
3. Invalid email format
4. User exists, not in group
5. User exists, already in group
6. Non-existent group
7. Multiple groups with same name
8. API authentication failure
9. Insufficient permissions
10. API throttling scenarios
11. Slack webhook failure (should not block workflow)
12. Slack webhook success with proper message formatting

### Test Environment
- Use a non-production Azure Entra tenant for testing
- Create test users and groups
- Verify cleanup after testing

## Implementation Technology Options

### Option 1: Azure CLI
**Pros**:
- Simple commands
- Well-documented
- Built-in retry logic

**Cons**:
- Limited for complex queries
- May require parsing output

### Option 2: Azure PowerShell
**Pros**:
- Rich object manipulation
- Native Microsoft support
- Good for complex logic

**Cons**:
- PowerShell syntax complexity
- Larger action footprint

### Option 3: Microsoft Graph API (Direct HTTP)
**Pros**:
- Most flexible
- Direct control over requests
- Lightweight

**Cons**:
- Requires manual authentication handling
- More code for error handling

### Recommended Approach
Use **Azure CLI** for authentication and basic operations, supplemented with **Microsoft Graph API** calls using `curl` or `gh` for operations not well-supported in CLI.

## Success Criteria

1. Workflow can be triggered manually with email input
2. Successfully creates new guest users
3. Handles existing users gracefully (no errors)
4. Sends invitation emails when requested
5. Adds users to specified groups
6. Skips group addition if user already member
7. Sends Slack notification when provisioning is initiated
8. Slack notification includes initiator, user email, and target group
9. Provides clear, actionable output
10. Fails gracefully with helpful error messages
11. Completes in under 2 minutes for typical operations
12. Maintains security best practices

## Future Enhancements

### Phase 2 Potential Features
1. Support for adding users to multiple groups
2. Set additional user properties (job title, department, etc.)
3. Support for creating internal users (not just guests)
4. Bulk user provisioning from CSV file
5. Integration with HR systems via webhooks
6. Automatic group assignment based on email domain
7. Custom email templates for invitations
8. Enhanced Slack notifications with completion status and error details
9. Microsoft Teams notification integration as an alternative to Slack

## Sample Workflow File Structure

```yaml
name: Provision Azure Entra User
on:
  workflow_dispatch:
    inputs:
      user_email:
        description: 'Email address of user to provision'
        required: true
        type: string
      entra_group_name:
        description: 'Azure Entra group name'
        required: true
        type: string
      display_name:
        description: 'Display name for new users'
        required: false
        type: string
      send_invitation:
        description: 'Send invitation email'
        required: false
        type: boolean
        default: true

jobs:
  provision-user:
    runs-on: ubuntu-latest
    steps:
      - name: Send Slack Notification
      - name: Azure Login
      - name: Validate Email
      - name: Check User Exists
      - name: Create User / Send Invitation
      - name: Resolve Group ID
      - name: Check Group Membership
      - name: Add to Group
      - name: Generate Summary
```

## Maintenance and Operations

### Monitoring
- Review workflow run history monthly
- Track failure rates and common errors
- Monitor service principal credential expiration

### Documentation
- Maintain runbook for common issues
- Document service principal setup process
- Keep list of groups available for provisioning

### Support
- Designate owner/maintainer for the workflow
- Establish escalation path for failures
- Document recovery procedures

## Compliance and Governance

1. **Data Privacy**: Ensure invitation emails comply with organizational data handling policies
2. **Access Control**: Limit who can trigger the workflow using GitHub environment protection rules
3. **Change Management**: Require PR reviews for workflow modifications
4. **Audit Requirements**: Preserve workflow run logs per retention policy

## Appendix

### Useful Microsoft Graph API Endpoints
- Create invitation: `POST https://graph.microsoft.com/v1.0/invitations`
- Get user: `GET https://graph.microsoft.com/v1.0/users/{email}`
- List groups: `GET https://graph.microsoft.com/v1.0/groups`
- Get group members: `GET https://graph.microsoft.com/v1.0/groups/{id}/members`
- Add group member: `POST https://graph.microsoft.com/v1.0/groups/{id}/members/$ref`

### Azure CLI Commands Reference
```bash
# Login with service principal
az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant $AZURE_TENANT_ID

# Check if user exists
az ad user show --id "user@example.com"

# Get group by name
az ad group show --group "GroupName"

# Check group membership
az ad group member check --group "GroupId" --member-id "UserId"

# Add user to group
az ad group member add --group "GroupId" --member-id "UserId"
```

### Required Azure AD Permissions Summary
| Permission | Type | Reason |
|------------|------|--------|
| User.ReadWrite.All | Application | Create and read user accounts |
| User.Invite.All | Application | Send invitation emails |
| Group.ReadWrite.All | Application | Manage group memberships |
| Directory.Read.All | Application | Search directory objects |

---

---

**Document Version**: 1.1
**Last Updated**: 2025-12-15
**Author**: Platform DevOps Team
**Changelog**:
- v1.1: Added Slack notification requirement for provisioning initiation
- v1.0: Initial specification
