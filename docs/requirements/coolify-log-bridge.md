# Requirements: Coolify Log Bridge

## Overview

This document outlines the requirements for the Coolify↔GitHub bridge that surfaces Coolify build/deploy logs into GitHub PRs.

## Problem Statement

Currently, the team lacks visibility into Coolify deploy/build failures from GitHub PRs. When a PR build/deploy fails, there is no easy way to access the relevant Coolify logs directly from the GitHub interface.

## Goals

1. **Primary Goal**: Enable developers to view Coolify deployment logs directly from GitHub PRs without navigating away
2. **Secondary Goal**: Provide actionable failure information (links + excerpts) while maintaining security
3. **Tertiary Goal**: Support both success and failure states with appropriate visibility

## Functional Requirements

### FR-1: Log Retrieval
- The bridge must be able to fetch Coolify deployment/build logs via Coolify API
- Must support both build logs and deployment logs
- Must support filtering by commit SHA and/or branch name
- Must support retrieving logs for completed (success/failure) deployments

### FR-2: GitHub Integration
- Must attach deployment status to GitHub PRs
- Preferred mechanism: GitHub Check Run (provides rich UI, inline annotations)
- Alternative: PR comment or commit status (less preferred but simpler)
- Must show:
  - Deployment status (pending, success, failure)
  - Link to full Coolify logs
  - Short excerpt (first ~500 chars of relevant logs)

### FR-3: Deterministic Mapping
- Must deterministically map GitHub commit SHA → Coolify deployment
- Must handle branch-based deployments (main, feature branches)
- Must handle the case where multiple deployments exist for same commit

### FR-4: Security - Secret Redaction
- **Critical**: Must NOT leak secrets or deployment configuration in GitHub artifacts
- Must implement redaction rules for:
  - Environment variables containing `KEY`, `SECRET`, `PASSWORD`, `TOKEN`, `API_KEY`
  - AWS/Azure/GCP credentials
  - Database connection strings
  - Private keys (PEM, SSH)
  - Bearer tokens in headers/urls
- Redaction must happen BEFORE data reaches GitHub

### FR-5: Authentication
- Must support Coolify API key authentication
- Must support GitHub token (PAT or App token)
- Tokens must be stored as GitHub secrets, never in repo code
- Bridge must handle token rotation gracefully

## Non-Functional Requirements

### NFR-1: Performance
- Log retrieval should complete within 10 seconds
- Bridge should handle rate limiting gracefully

### NFR-2: Reliability
- Must handle Coolify API unavailability gracefully
- Must implement retries with exponential backoff
- Must have proper logging for debugging

### NFR-3: Maintainability
- Code must be well-documented
- Configuration should be externalized
- Redaction rules should be easily extensible

## Out of Scope (Explicit Non-Goals)

- Triggering Coolify deployments from GitHub (read-only)
- Supporting multiple Coolify instances (single instance for now)
- Real-time log streaming (polling is acceptable)
- Storing logs long-term (just fetch and display)

## Acceptance Criteria

1. ✅ For a failed deploy/build, a GitHub PR shows a link to the matching Coolify logs
2. ✅ For a successful deploy/build, PR shows success + optional link
3. ✅ No secrets in GitHub artifacts; sensitive log lines are redacted
4. ✅ Mapping between GitHub commit SHA and Coolify deployment is deterministic
5. ✅ Architecture document specifies all components and data flows

## Related Issues

- Issue #4: Spec: Surface Coolify build/deploy logs back into GitHub PRs

## Dependencies

- Coolify API access (v4 API)
- GitHub API access (Checks API or PR Comments API)
- GitHub Secrets for token storage
