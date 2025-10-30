# Slack System Roles to Okta Identity Governance - Implementation Guide

**Author**: Solutions Engineering  
**Last Updated**: October 30, 2025  
**Version**: 1.0

## Overview

This guide provides a complete implementation for syncing Slack system roles to Okta Identity Governance (OIG) as entitlements for access certification. This is a temporary solution until native Slack support is available in OIG (H1 2026).

### What This Solution Does

- Reads system role assignments from Slack Admin API
- Creates corresponding entitlements in OIG
- Syncs user assignments (grants) to match Slack state
- Runs daily with differential sync (only updates what changed)
- Tracks execution metadata for monitoring and auditing

### Architecture

```
┌─────────────┐         ┌──────────────┐         ┌─────────────────┐
│   Slack     │         │    Okta      │         │   Okta Identity │
│   Admin API │ ──────> │  Workflows   │ ──────> │   Governance    │
│             │         │              │         │   (Entitlements)│
└─────────────┘         └──────────────┘         └─────────────────┘
                                │
                                ▼
                        ┌──────────────┐
                        │  Workflows   │
                        │   Tables     │
                        │  (Metadata)  │
                        └──────────────┘
```

### Sync Strategy: Differential Sync

This implementation uses **differential sync** to preserve audit trails and prevent access loss during failures:

1. Fetch current assignments from Slack
2. Fetch current entitlements/grants from Okta
3. Calculate differences (additions and removals)
4. Apply only the changes
5. Record execution metadata

**Why differential sync?**
- Preserves accurate timestamps and change history for compliance
- Protects against access removal during partial failures
- More efficient (fewer API calls)
- Clear audit trail of what changed and why

---

## Prerequisites

### Okta Requirements

- **Okta Identity Governance** subscription
- Slack application already configured in Okta for provisioning
- Governance Engine enabled on the Slack app
- Super Admin or admin with the following permissions:
  - Application Administrator for Slack app
  - Governance Administrator
  - API token management

### Slack Requirements

- Slack Enterprise Grid plan (required for system roles)
- Slack app with Admin API access
- Required OAuth scopes:
  - `admin.roles:read` - Read system roles
  - `admin.roles.listAssignments` - List role assignments
  - `admin.users:read` - Read user information
  
### Workflows Requirements

- Okta Workflows enabled
- Workflows connectors:
  - Okta Connector (configured)
  - HTTP Connector (for Slack API calls)

---

## Slack System Roles Reference

The following system roles from Slack will be synced as entitlements:

| Role Name | Description |
|-----------|-------------|
| Analytics Admin | Access analytics dashboard |
| Audit Logs Admin | Access audit logs and exports |
| Content Admin | Review and manage flagged content, custom emoji, Slackbot responses, DLP |
| Roles Admin | Create custom roles and manage system role assignments |
| User Lifecycle Admin | Manage sessions and deactivate accounts |
| Workflows Admin | Manage workflows |

Reference: https://slack.com/help/articles/201314026-Permissions-by-role-in-Slack

---

## Implementation Steps

### Phase 1: Slack App Configuration

#### 1.1 Create or Update Slack App

1. Go to https://api.slack.com/apps
2. Create a new app or select your existing Slack app
3. Navigate to **OAuth & Permissions**
4. Add the following scopes under **User Token Scopes**:
   - `admin.roles:read`
   - `admin.users:read`
5. Click **Save Changes**

#### 1.2 Install App to Workspace

1. In your Slack app settings, go to **Install App**
2. Click **Install to Workspace**
3. Authorize the requested permissions
4. Copy the **User OAuth Token** (starts with `xoxp-`)
   - Store this securely - you'll need it for Workflows

#### 1.3 Verify API Access

Test your token using curl:

```bash
curl -H "Authorization: Bearer xoxp-YOUR-TOKEN" \
  "https://slack.com/api/admin.roles.listAssignments"
```

Expected response: `{"ok": true, "role_assignments": [...]}`

---

### Phase 2: Okta Configuration

#### 2.1 Enable Governance Engine on Slack App

1. Log in to Okta Admin Console
2. Navigate to **Applications** > **Applications**
3. Find and select your **Slack** application
4. Click the **Governance** tab
5. Toggle **Enable Governance Engine** to ON
6. Click **Save**

#### 2.2 Record Slack App ID

1. In the Slack application page, look at the URL
2. The app ID is the last segment: `https://[tenant]-admin.okta.com/admin/app/[instance]/[APP_ID]`
3. Example: `0oa1b2c3d4e5f6g7h8i9`
4. Save this - you'll need it for API calls

#### 2.3 Create API Token

1. Navigate to **Security** > **API**
2. Click **Tokens** tab
3. Click **Create Token**
4. Name: `Slack-OIG-Sync-Workflow`
5. Click **Create Token**
6. Copy the token value (you won't see it again)
7. Store securely

---

### Phase 3: Workflows Setup

#### 3.1 Create Workflows Tables

Create two tables in Workflows to store data:

**Table 1: sync_execution_log**

| Column Name | Type | Description |
|-------------|------|-------------|
| execution_id | Text | Unique ID for each execution |
| start_time | Text | ISO timestamp when sync started |
| end_time | Text | ISO timestamp when sync completed |
| duration_seconds | Number | Total execution time |
| status | Text | SUCCESS, FAILED, or PARTIAL |
| total_slack_users | Number | Total users with system roles in Slack |
| total_entitlements_synced | Number | Number of entitlements created/updated |
| assignments_added | Number | Number of new grants created |
| assignments_removed | Number | Number of grants revoked |
| users_not_in_okta | Text | JSON array of Slack users not found in Okta |
| error_message | Text | Error details if sync failed |

**Table 2: system_role_mapping**

| Column Name | Type | Description |
|-------------|------|-------------|
| slack_role_id | Text | Slack system role identifier |
| slack_role_name | Text | Display name in Slack |
| okta_entitlement_id | Text | OIG entitlement ID |
| created_at | Text | ISO timestamp of entitlement creation |
| last_synced | Text | ISO timestamp of last successful sync |

To create these tables:

1. Open Okta Workflows console
2. Navigate to **Tables**
3. Click **New Table**
4. Enter the table name and add columns as specified above
5. Click **Save**

#### 3.2 Configure Connections

**HTTP Connection for Slack API:**

1. In Workflows, go to **Connections**
2. Click **New Connection**
3. Select **HTTP** connector
4. Click **Create**
5. In the connection settings:
   - Name: `Slack Admin API`
   - Base URL: `https://slack.com/api`
   - Auth Type: `Custom`
   - Headers:
     - Key: `Authorization`
     - Value: `Bearer xoxp-YOUR-SLACK-TOKEN`
     - Key: `Content-Type`
     - Value: `application/json`
6. Click **Save**

**Okta Connector:**

1. In Workflows, go to **Connections**
2. Click **New Connection**
3. Select **Okta** connector
4. Follow the authorization flow
5. Ensure the connection uses an API token with sufficient privileges

**API Connector for OIG:**

1. In Workflows, go to **Connections**
2. Click **New Connection**
3. Select **API Connector**
4. Settings:
   - Name: `Okta IGA API`
   - Base URL: `https://[your-tenant].okta.com/governance/api/v1`
   - Auth Type: `Custom`
   - Headers:
     - Key: `Authorization`
     - Value: `SSWS YOUR-API-TOKEN`
     - Key: `Accept`
     - Value: `application/json`
     - Key: `Content-Type`
     - Value: `application/json`
5. Click **Save**

---

### Phase 4: Workflow Implementation

#### 4.1 Main Flow: Sync Slack System Roles to OIG

**Flow Name**: `Slack System Roles - Daily Sync`

**Trigger**: Scheduled (Daily at 2:00 AM)

**High-Level Steps**:

1. Initialize execution metadata
2. Fetch system role assignments from Slack
3. Fetch current entitlements and grants from OIG
4. Calculate differences (additions and removals)
5. Create/update entitlements in OIG
6. Add new grants (assignments)
7. Remove revoked grants
8. Update execution log

**Detailed Flow Design**:

```
[Scheduled Trigger - Daily 2AM]
  │
  ├─> [Create Execution ID] (UUID)
  │     │
  │     └─> execution_id = Text - Compose: "slack-sync-{UUID}-{TIMESTAMP}"
  │         start_time = Date - Current Time (ISO format)
  │
  ├─> [Call Slack API - List Role Assignments]
  │     │
  │     └─> GET https://slack.com/api/admin.roles.listAssignments
  │         └─> Response format:
  │             {
  │               "ok": true,
  │               "role_assignments": [
  │                 {
  │                   "role_id": "Rv01234",
  │                   "role_name": "Analytics Admin",
  │                   "entity_type": "USER",
  │                   "entity_id": "U012345",
  │                   "date_create": 1635000000
  │                 }
  │               ]
  │             }
  │
  ├─> [Call Helper: Get User Details for Each Role Assignment]
  │     │
  │     └─> For each role assignment, call Slack users.info API
  │         Store: user_id, email, slack_role_id, slack_role_name
  │         Result: List of role assignments with email addresses
  │
  ├─> [Filter: Remove Users Without Email]
  │     │
  │     └─> Only keep role assignments where email exists
  │
  ├─> [Call Okta API - Search Users by Email]
  │     │
  │     └─> For each email, search Okta:
  │         GET /api/v1/users?filter=profile.email eq "user@example.com"
  │         Store: okta_user_id, email
  │         Track users not found in separate list
  │
  ├─> [Call OIG API - List Current Entitlements for Slack App]
  │     │
  │     └─> GET /governance/api/v1/resources/{slack_app_id}/entitlements
  │         Store existing entitlements by value (role name)
  │
  ├─> [Calculate Entitlements to Create]
  │     │
  │     └─> Compare Slack roles vs existing entitlements
  │         New roles = Slack roles NOT in OIG entitlements
  │
  ├─> [Create New Entitlements in OIG]
  │     │
  │     └─> For each new role:
  │         POST /governance/api/v1/resources/{slack_app_id}/entitlements
  │         Body:
  │         {
  │           "name": "Slack System Role - {role_name}",
  │           "attribute": "slack_system_role",
  │           "dataType": "string",
  │           "description": "System role in Slack: {role_name}"
  │         }
  │         
  │         Response includes entitlement_id
  │         
  │         POST /governance/api/v1/entitlements/{entitlement_id}/values
  │         Body:
  │         {
  │           "value": "{slack_role_name}",
  │           "displayName": "{slack_role_name}"
  │         }
  │         
  │         Store in system_role_mapping table
  │
  ├─> [Call OIG API - List Current Grants for Slack App]
  │     │
  │     └─> GET /governance/api/v1/resources/{slack_app_id}/grants
  │         Store: okta_user_id, entitlement_id, grant_id
  │
  ├─> [Calculate Grant Changes]
  │     │
  │     ├─> Grants to Add:
  │     │     For each (okta_user_id, slack_role) in Slack
  │     │     If NOT exists in OIG grants
  │     │     → Add to "add" list
  │     │
  │     └─> Grants to Remove:
  │           For each grant in OIG
  │           If NOT exists in Slack assignments
  │           → Add to "remove" list
  │
  ├─> [Add New Grants]
  │     │
  │     └─> For each grant to add:
  │         POST /governance/api/v1/resources/{slack_app_id}/grants
  │         Body:
  │         {
  │           "principalId": "{okta_user_id}",
  │           "principalType": "USER",
  │           "entitlementId": "{entitlement_id}",
  │           "entitlementValue": "{slack_role_name}",
  │           "grantSource": "IMPORT"
  │         }
  │
  ├─> [Remove Revoked Grants]
  │     │
  │     └─> For each grant to remove:
  │         DELETE /governance/api/v1/grants/{grant_id}
  │
  ├─> [Calculate Execution Statistics]
  │     │
  │     └─> end_time = Current Time (ISO)
  │         duration_seconds = (end_time - start_time) in seconds
  │         status = "SUCCESS" (or "FAILED" if errors)
  │
  └─> [Write Execution Log to Table]
        │
        └─> INSERT into sync_execution_log
            All calculated statistics and metadata
```

#### 4.2 Helper Flows

**Flow 2: Get Slack User Details**

**Purpose**: Fetch email addresses for Slack users

**Inputs**: 
- slack_user_id (Text)

**Steps**:
1. Call Slack API: `GET /users.info?user={slack_user_id}`
2. Extract email from response: `response.user.profile.email`
3. Return email

**Outputs**:
- email (Text)

---

**Flow 3: Search Okta User by Email**

**Purpose**: Find Okta user ID by email

**Inputs**:
- email (Text)

**Steps**:
1. Call Okta API: `GET /api/v1/users?filter=profile.email eq "{email}"`
2. Check if users found (list length > 0)
3. If found, extract first user's ID
4. Return user ID or null

**Outputs**:
- okta_user_id (Text or null)

---

**Flow 4: Create or Get Entitlement**

**Purpose**: Ensure entitlement exists for a Slack role

**Inputs**:
- slack_app_id (Text)
- slack_role_name (Text)

**Steps**:
1. Query system_role_mapping table by slack_role_name
2. If found, return okta_entitlement_id
3. If not found:
   - Create entitlement in OIG
   - Create entitlement value
   - Store in system_role_mapping table
   - Return new okta_entitlement_id

**Outputs**:
- okta_entitlement_id (Text)

---

#### 4.3 Error Handling

**Each API call should include error handling:**

```
[API Call Card]
  │
  ├─> On Success: Continue to next step
  │
  └─> On Error:
        ├─> Log error details
        ├─> Set execution status = "FAILED"
        ├─> Record error_message in execution log
        └─> Send notification (optional)
```

**Key error scenarios to handle:**

1. **Slack API Rate Limits**
   - Slack allows 100+ requests per minute for most endpoints
   - If rate limited (429 response), wait and retry
   - Consider implementing retry logic with exponential backoff

2. **Okta API Errors**
   - 401: Invalid or expired API token
   - 404: Resource (app, user) not found
   - 429: Rate limit exceeded
   - Log and alert on these errors

3. **User Not Found in Okta**
   - Don't fail entire sync
   - Log the user's email in `users_not_in_okta` field
   - Continue processing other users

4. **Partial Failures**
   - If some grants succeed and some fail
   - Set status = "PARTIAL"
   - Record both successes and failures in metadata

---

### Phase 5: Testing

#### 5.1 Test with Small Dataset

**Before running full sync:**

1. Manually assign 2-3 test users to different Slack system roles
2. Run the workflow manually (don't wait for scheduled run)
3. Verify:
   - Entitlements created in OIG for each role
   - Grants created for test users
   - Execution log recorded with accurate counts
   - No errors in workflow execution history

#### 5.2 Verify in Okta Admin Console

1. Navigate to **Applications** > **Slack** > **Governance** > **Entitlements**
2. Confirm entitlements exist for Slack system roles
3. Click each entitlement to view assigned users
4. Verify users match Slack assignments

#### 5.3 Test Differential Sync

**Test that only changes are applied:**

1. Run initial sync (note assignments_added count)
2. Don't make any changes in Slack
3. Run sync again
4. Verify:
   - `assignments_added = 0`
   - `assignments_removed = 0`
   - Execution completes successfully

**Test adding a new assignment:**

1. Assign a new user to a system role in Slack
2. Run sync
3. Verify:
   - `assignments_added = 1`
   - New grant appears in OIG

**Test revoking an assignment:**

1. Remove a user from a system role in Slack
2. Run sync
3. Verify:
   - `assignments_removed = 1`
   - Grant removed from OIG

#### 5.4 Test Error Scenarios

**Test user not in Okta:**

1. Create a test user in Slack (with email) who doesn't exist in Okta
2. Assign them a system role
3. Run sync
4. Verify:
   - Workflow doesn't fail
   - User's email appears in `users_not_in_okta` field
   - Other users' grants still processed successfully

---

### Phase 6: Deployment

#### 6.1 Enable Daily Schedule

1. Open the main sync flow in Workflows
2. Verify the scheduled trigger is set to run daily at 2:00 AM (or preferred time)
3. Toggle the flow to **ON**
4. Confirm the schedule appears in **Scheduled Flows**

#### 6.2 Set Up Monitoring

**Monitor workflow execution:**

1. Check Workflows execution history daily
2. Review `sync_execution_log` table for:
   - Execution status (look for FAILED)
   - Duration trends (sudden increases may indicate issues)
   - Error messages

**Create alerts (optional):**

Consider adding a notification flow that:
- Triggers on workflow failure
- Sends email or Slack message to admin team
- Includes error details from execution log

#### 6.3 Document for Team

Create a quick reference for your team:

- Location of workflow in Okta Workflows
- How to manually trigger a sync
- Where to view execution history
- What to do if sync fails
- When native Slack support launches (H1 2026), how to sunset this workflow

---

## Operational Guide

### Daily Operations

**What happens each day:**

1. Workflow runs at 2:00 AM (or configured time)
2. Syncs Slack system role assignments to OIG
3. Records execution metadata in table
4. Completes in ~1-5 minutes (depending on user count)

**No action needed unless:**
- Workflow execution fails (check execution history)
- Execution duration suddenly increases (may indicate API issues)
- High number of `users_not_in_okta` (may need investigation)

### Monitoring Execution Logs

**Query execution logs table:**

```
Recent executions:
- Sort by start_time descending
- Review status field
- Check for errors

Success metrics:
- status = "SUCCESS"
- assignments_added + assignments_removed = expected changes
- No values in error_message field

Failure indicators:
- status = "FAILED" or "PARTIAL"
- error_message populated
- Duration much longer than normal
```

### Troubleshooting

**Workflow fails with 401 error:**
- Slack or Okta API token expired/invalid
- Regenerate token and update connection

**Workflow fails with 404 error:**
- Slack app not found (check app ID)
- Governance Engine not enabled on app
- Verify prerequisites

**Users missing from sync:**
- Check if user exists in Okta (search by email)
- Verify email address matches between Slack and Okta
- Review `users_not_in_okta` field in execution log

**Grants not appearing in OIG:**
- Verify entitlements were created successfully
- Check grant creation API response for errors
- Confirm user is assigned to Slack app in Okta

**Sync taking too long:**
- Check Slack API rate limit status
- Consider optimizing API calls (batch requests where possible)
- Review Workflows execution history for bottlenecks

---

## API Reference

### Slack Admin API Endpoints

**List Role Assignments**
```
GET https://slack.com/api/admin.roles.listAssignments
Headers:
  Authorization: Bearer xoxp-YOUR-TOKEN

Response:
{
  "ok": true,
  "role_assignments": [
    {
      "role_id": "Rv01234",
      "role_name": "Analytics Admin",
      "entity_type": "USER",
      "entity_id": "U012345",
      "date_create": 1635000000
    }
  ]
}
```

**Get User Info**
```
GET https://slack.com/api/users.info?user=U012345
Headers:
  Authorization: Bearer xoxp-YOUR-TOKEN

Response:
{
  "ok": true,
  "user": {
    "id": "U012345",
    "profile": {
      "email": "user@example.com",
      "real_name": "John Doe"
    }
  }
}
```

### Okta Identity Governance API Endpoints

**List Entitlements for App**
```
GET https://{tenant}.okta.com/governance/api/v1/resources/{app_id}/entitlements
Headers:
  Authorization: SSWS {api_token}
  Accept: application/json

Response:
{
  "_embedded": {
    "entitlements": [
      {
        "id": "esp123",
        "name": "Slack System Role - Analytics Admin",
        "attribute": "slack_system_role",
        "dataType": "string"
      }
    ]
  }
}
```

**Create Entitlement**
```
POST https://{tenant}.okta.com/governance/api/v1/resources/{app_id}/entitlements
Headers:
  Authorization: SSWS {api_token}
  Content-Type: application/json

Body:
{
  "name": "Slack System Role - Analytics Admin",
  "attribute": "slack_system_role",
  "dataType": "string",
  "description": "System role in Slack: Analytics Admin"
}

Response:
{
  "id": "esp123",
  "name": "Slack System Role - Analytics Admin",
  "attribute": "slack_system_role",
  "dataType": "string"
}
```

**Create Entitlement Value**
```
POST https://{tenant}.okta.com/governance/api/v1/entitlements/{entitlement_id}/values
Headers:
  Authorization: SSWS {api_token}
  Content-Type: application/json

Body:
{
  "value": "Analytics Admin",
  "displayName": "Analytics Admin"
}

Response:
{
  "id": "entv123",
  "value": "Analytics Admin",
  "displayName": "Analytics Admin"
}
```

**List Grants for App**
```
GET https://{tenant}.okta.com/governance/api/v1/resources/{app_id}/grants
Headers:
  Authorization: SSWS {api_token}
  Accept: application/json

Response:
{
  "_embedded": {
    "grants": [
      {
        "id": "grt123",
        "principalId": "00u1b2c3d4e5f6g7h8",
        "principalType": "USER",
        "entitlementId": "esp123",
        "entitlementValue": "Analytics Admin"
      }
    ]
  }
}
```

**Create Grant**
```
POST https://{tenant}.okta.com/governance/api/v1/resources/{app_id}/grants
Headers:
  Authorization: SSWS {api_token}
  Content-Type: application/json

Body:
{
  "principalId": "00u1b2c3d4e5f6g7h8",
  "principalType": "USER",
  "entitlementId": "esp123",
  "entitlementValue": "Analytics Admin",
  "grantSource": "IMPORT"
}

Response:
{
  "id": "grt123",
  "principalId": "00u1b2c3d4e5f6g7h8",
  "status": "ACTIVE"
}
```

**Delete Grant**
```
DELETE https://{tenant}.okta.com/governance/api/v1/grants/{grant_id}
Headers:
  Authorization: SSWS {api_token}

Response: 204 No Content
```

---

## Migration to Native Support

When Okta releases native Slack support in H1 2026:

1. **Before disabling this workflow:**
   - Export execution logs for historical reference
   - Document any custom entitlement mappings
   - Verify native integration covers all system roles

2. **Transition:**
   - Enable native Slack entitlement integration
   - Verify all roles and assignments migrate correctly
   - Run parallel sync for 1-2 weeks to validate

3. **Cleanup:**
   - Disable scheduled workflow
   - Archive workflows and tables
   - Remove custom API connections
   - Update documentation

---

## Appendix

### A. Slack System Roles - Full List

| Role ID Pattern | Role Name | Description |
|-----------------|-----------|-------------|
| Rv* (varies) | Analytics Admin | View analytics dashboard |
| Rv* (varies) | Audit Logs Admin | Access audit logs and exports |
| Rv* (varies) | Content Admin | Review flagged content, manage custom emoji, Slackbot, DLP |
| Rv* (varies) | Roles Admin | Create custom roles and manage system role assignments |
| Rv* (varies) | User Lifecycle Admin | Manage sessions and deactivate accounts |
| Rv* (varies) | Workflows Admin | Manage workflows |

Note: Slack role IDs are workspace-specific. Your workspace may have different IDs.

### B. OIG Entitlement Best Practices

**Naming Convention:**
- Use descriptive names: "Slack System Role - {Role Name}"
- Include source system in name for clarity
- Keep attribute names consistent: "slack_system_role"

**Metadata:**
- Add descriptions to explain what each role grants access to
- Document when entitlements were created (use tables)
- Track which workflows manage which entitlements

**Access Certification:**
- Schedule regular certification campaigns (quarterly recommended)
- Include all Slack system role entitlements in scope
- Assign appropriate reviewers (Slack admins or workspace owners)

### C. Security Considerations

**API Token Security:**
- Store tokens in Workflows connections (encrypted at rest)
- Rotate tokens every 90 days minimum
- Use dedicated service account for API tokens
- Audit token usage regularly

**Least Privilege:**
- Slack app should only request needed scopes
- API token should have minimum required permissions
- Consider using OAuth 2.0 for Slack instead of user token

**Audit Trail:**
- Execution logs provide audit trail of all changes
- OIG maintains grant history automatically
- Keep execution logs for 1 year minimum

**Failure Safety:**
- Differential sync prevents mass access removal
- Failed executions don't modify existing state
- Monitor for repeated failures (may indicate compromise)

### D. Performance Optimization

**Current Implementation:**
- Expected execution time: 1-5 minutes
- Suitable for up to 1,000 users with system roles
- API calls scale linearly with user count

**For Large Environments (>1,000 users):**
- Consider batching API calls
- Implement pagination for Slack API responses
- Use parallel helper flows for user lookups
- Cache entitlement mappings to reduce OIG API calls

**Rate Limit Handling:**
- Slack: 100+ requests/minute for most endpoints
- OIG: Check specific limits in documentation
- Implement exponential backoff for rate limit errors

---

## Support and Resources

### Documentation
- [Slack Admin API - Managing Users](https://docs.slack.dev/admins/managing-users)
- [Okta Identity Governance API](https://developer.okta.com/docs/api/iga/)
- [OIG Entitlement Management](https://help.okta.com/en-us/content/topics/identity-governance/em/entitlement-mgt.htm)

### Related Articles
- [Managing Custom Entitlements in OIG](https://iamse.blog/2024/01/26/managing-custom-entitlements-or-bring-your-own-byo-entitlements-using-okta-identity-governance-oig-entitlement-management/)
- [Okta Workflows Best Practices](https://help.okta.com/en-us/content/topics/workflows/workflows-best-practices.htm)

### Getting Help
- Okta Community Forums
- Okta Support Portal
- Your Okta Customer Success Manager

---

**Document Version History:**

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | Oct 30, 2025 | Initial implementation guide |

---

*This guide is maintained by Solutions Engineering and will be updated as requirements change or native support becomes available.*
