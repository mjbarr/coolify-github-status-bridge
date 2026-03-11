# Technical Specification: Coolify Log Bridge

## Architecture Overview

The Coolify Log Bridge is a service that fetches deployment logs from Coolify and publishes them to GitHub PRs via Check Runs. It operates as a webhook handler that reacts to Coolify deployment events.

```
┌─────────────┐     Webhook      ┌──────────────────┐     API      ┌─────────┐
│  Coolify    │ ───────────────► │  Bridge Service  │ ──────────► │ GitHub  │
│  Instance   │                  │                  │              │  API    │
└─────────────┘                  │  - Fetch logs    │              └─────────┘
                                 │  - Redact secrets│
                                 │  - Create Check  │
                                 │    Run            │
                                 └──────────────────┘
```

## Components

### 1. Webhook Receiver
- **Purpose**: Receive deployment events from Coolify
- **Endpoint**: `POST /webhook/coolify`
- **Coolify Events**: `deployment_started`, `deployment_done`, `deployment_failed`

### 2. Log Fetcher
- **Purpose**: Retrieve deployment logs from Coolify API
- **API Endpoint**: `GET /api/v4/deployments/{id}/logs`
- **Filtering**: By deployment ID, branch, or commit SHA

### 3. Secret Redactor
- **Purpose**: Remove sensitive information before publishing to GitHub
- **Implementation**: Regex-based pattern matching + replacement
- **Patterns**: See Redaction Rules section

### 4. GitHub Reporter
- **Purpose**: Create/update Check Runs on GitHub
- **API Endpoint**: `POST /repos/{owner}/{repo}/check-runs`
- **Output**: Check Run with annotations and external URL

## Data Flow

```
1. Developer pushes commit → GitHub CI runs → Deploys to Coolify
2. Coolify completes deployment → sends webhook to Bridge
3. Bridge receives webhook:
   a. Extracts deployment ID, status, branch, commit SHA
   b. Calls Coolify API to fetch logs
   c. Runs logs through Redactor
   d. Creates/updates GitHub Check Run with:
      - Status (success/failure)
      - Title: "Coolify Deployment"
      - Summary: Redacted log excerpt (first 500 chars)
      - Annotation: Link to full Coolify logs
      - External URL: Direct link to Coolify deployment
4. Developer sees results in GitHub PR
```

## API Specification

### Webhook Payload (from Coolify)

```json
{
  "event": "deployment_done",
  "deployment": {
    "id": " deployment_abc123",
    "status": "success",
    "branch": "main",
    "commit": "abc123def456",
    "build_id": "build_xyz789",
    "application": {
      "name": "my-app",
      "id": "app_123"
    },
    "created_at": "2024-01-15T10:30:00Z",
    "finished_at": "2024-01-15T10:35:00Z"
  }
}
```

### GitHub Check Run Output

```json
{
  "name": "Coolify Deployment",
  "status": "completed",
  "conclusion": "success",
  "output": {
    "title": "Deployment to production successful",
    "summary": "Deployment completed in 5m 23s\n\n**Branch:** main\n**Commit:** abc123d\n\n**Log Excerpt:**\n```\n[10:30:01] Starting deployment...\n[10:30:15] Building Docker image...\n[10:32:45] Pushing to registry...\n[10:35:00] Deployment complete!\n```\n\n[View full logs in Coolify](https://coolify.example.com/deployment/abc123)",
    "annotations": [
      {
        "path": "deployment",
        "start_line": 1,
        "end_line": 1,
        "annotation_level": "notice",
        "message": "Deployment successful - [View logs in Coolify](https://coolify.example.com/deployment/abc123)"
      }
    ]
  },
  "external_id": "deployment_abc123",
  "external_url": "https://coolify.example.com/deployment/abc123"
}
```

## Redaction Rules

The Redactor applies the following rules in order:

### Rule 1: Environment Variable Patterns
Redact values for keys matching:
- `/^(AWS_|AZURE_|GCP_|GOOGLE_|KUBERNETES_)/i`
- `/(KEY|SECRET|PASSWORD|TOKEN|API_KEY|PRIVATE_KEY|CREDENTIAL|AUTH)/i`

**Before**: `AWS_SECRET_ACCESS_KEY=abcdef123456`
**After**: `AWS_SECRET_ACCESS_KEY=[REDACTED]`

### Rule 2: URI Credentials
Redact credentials in URIs:
- `://([^:]+):([^@]+)@` → `://[REDACTED]:[REDACTED]@`

**Before**: `https://user:password@api.example.com`
**After**: `https://[REDACTED]:[REDACTED]@api.example.com`

### Rule 3: Bearer Tokens
Redact Bearer tokens:
- `/Bearer\s+([a-zA-Z0-9_\-\.]+)/i` → `Bearer [REDACTED]`

### Rule 4: Private Keys
Redact PEM private keys:
- `/-----BEGIN\s+(RSA\s+)?PRIVATE KEY-----/` → `[REDACTED PRIVATE KEY]`

### Rule 5: Database Connection Strings
Redact database credentials:
- `/(postgres|mysql|mongodb|redis):\/\/[^@]+@/` → `://[REDACTED]@/`

## Configuration

### Environment Variables

```env
# Required
COOLIFY_API_URL=https://coolify.example.com
COOLIFY_API_KEY=<coolify-api-key>
GITHUB_REPO=owner/repo
GITHUB_TOKEN=<github-pat>

# Optional
BRIDGE_PORT=3000
LOG_REDACTION_ENABLED=true
MAX_LOG_LENGTH=500
WEBHOOK_SECRET=<hmac-secret>
```

### GitHub Secrets (Recommended)

Store sensitive values as GitHub Secrets:
- `COOLIFY_API_KEY`: Coolify API token
- `COOLIFY_WEBHOOK_SECRET`: HMAC secret for webhook verification

## Deterministic Mapping Strategy

To map GitHub commit SHA → Coolify deployment:

1. **Primary**: Use deployment's `commit` field (if available from Coolify)
2. **Secondary**: Use branch name + timestamp matching
3. **Tertiary**: Query Coolify API for deployments by branch, find closest commit

The webhook approach is preferred because Coolify sends the commit SHA directly.

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Coolify API timeout | Retry 3x with exponential backoff (1s, 2s, 4s) |
| GitHub API rate limit | Retry after specified wait time |
| Invalid webhook signature | Return 401, log warning |
| Deployment not found | Create Check Run with error message |
| Redaction failure | Abort and log full error (don't leak) |

## Security Considerations

1. **Webhook Verification**: Verify HMAC signature on incoming webhooks
2. **Token Storage**: Use GitHub Secrets, never commit to repo
3. **Network**: Bridge should run in same network as Coolify or use VPN
4. **Audit Logging**: Log all access (who/what/when) without sensitive data
5. **Least Privilege**: Bridge only needs `read` on Coolify, `checks:write` on GitHub

## Deployment

The bridge can be deployed as:
- **Docker container**: Recommended for isolation
- **Serverless function**: For low-traffic scenarios
- **Existing Coolify service**: If supported

## Testing Strategy

1. **Unit Tests**: Redaction rules, mapping logic
2. **Integration Tests**: Coolify API mock, GitHub API mock
3. **E2E Tests**: Real webhook flow with test Coolify instance

## Future Enhancements (Out of Scope)

- Multi-Coolify support
- Real-time log streaming
- Deployment diffs
- Slack/Discord notifications
