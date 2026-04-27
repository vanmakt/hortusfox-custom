# Security Policy

## About security vulnerabilities

Security vulnerabilities describe major issues in a supported version of this product that can compromise your workspace. These exploits need to be dealt with promptly and may not be disclosed in public.

## Reporting a Vulnerability

Please report any security vulnerability to security@hortusfox.com confidently. Security vulnerabilities will be addressed as soon as possible in order to provide a fix. 

## Security update log

### 2026-04-27

- API authentication now supports `Authorization: Bearer <token>` to reduce dependency on query-string credentials.
- API routes were tightened from broad `ANY` methods to explicit `GET` (read endpoints) and `POST` (mutating endpoints).
- Added lightweight API audit logging (endpoint, status, token fingerprint, source IP, user-agent, token source) for security visibility.
- Query-string token usage is now marked as deprecated using `X-Api-Token-Deprecated: query`.
