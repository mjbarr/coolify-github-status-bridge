# ADR-001: Bridge Approach for Coolify↔GitHub Integration

## Status

**Accepted** - 2024-01-15

## Context

We need to integrate Coolify deployment logs with GitHub PRs to improve developer visibility. The key decision is HOW to connect Coolify's deployment events to GitHub.

### Options Considered

1. **Polling-based Bridge** (pull model)
   - Background job periodically queries Coolify API for recent deployments
   - Matches commits to deployments
   - Updates GitHub Check Runs

2. **Webhook-based Bridge** (push model)
   - Coolify sends webhooks on deployment events
   - Bridge receives and processes immediately
   - Updates GitHub Check Runs

3. **GitHub Actions + Coolify API** (no custom service)
   - Use GitHub Actions to query Coolify API post-deploy
   - Simpler but tighter coupling to CI

4. **GitHub App with Coolify Integration**
   - Full-featured GitHub App
   - Most complex to build and maintain

## Decision

**We chose Option 2: Webhook-based Bridge** as a standalone service.

### Rationale

| Factor | Webhook | Polling | Actions | GitHub App |
|--------|---------|---------|---------|------------|
| Latency | Low (immediate) | High (poll interval) | Medium | Low |
| Complexity | Medium | Low | Low | High |
| Reliability | Medium | High | High | Medium |
| Cost | Low | Low | GitHub minutes | App hosting |
| Real-time | ✅ | ❌ | ✅ | ✅ |
| No CI dependency | ✅ | ✅ | ❌ | ✅ |

**Key reasons:**
1. **Real-time**: Developers see results immediately after deployment
2. **Decoupled**: Not tied to GitHub Actions execution time
3. **Efficient**: Only processes events when they happen
4. **Scalable**: Can handle multiple repos/projects through configuration

## Consequences

### Positive

- Fast feedback loop (minutes, not polling intervals)
- Independent of CI system (works with any deploy method)
- Centralized logic (one bridge, multiple repos)
- Rich integration via Check Runs

### Negative

- Requires Coolify webhook configuration
- Additional service to deploy/maintain
- Need webhook security (HMAC verification)
- Bridge becomes a single point of failure (mitigate with retries)

### Mitigation

- Bridge will have health checks and automatic restarts
- Webhook retry queue for failed GitHub updates
- Comprehensive logging for debugging

## Implementation Notes

- Bridge will be a simple Node.js/Express service (or Python FastAPI)
- Use the existing `coolify-github-status-bridge` repo as base
- Deploy alongside existing infrastructure or as Docker container

## Alternatives Considered

### Webhook + PR Comment (Rejected)
- Simpler than Check Runs
- Less visibility (not in the PR summary)
- More noisy (comment clutter)

### Polling (Rejected)
- Would require 1-2 minute intervals for "real-time" feel
- Wastes API calls
- Latency unacceptable for dev workflow

## Related ADRs

- ADR-002: Secret Redaction Strategy (future)
- ADR-003: GitHub Integration Method (future)
