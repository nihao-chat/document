# Error Handling

| Date       | Description                                |
| ---------- | ------------------------------------------ |
| 2026-02-14 | FEAT-USER-001: User Account Management     |

## 1. Overview

This document defines the error handling strategy for nihao-chat backend services. It establishes a structured error code registry, maps between REST and gRPC error representations, and provides guidelines for consistent error handling across all services.

All REST API errors follow the standard response envelope:

```json
{
  "code": 1001,
  "data": null,
  "msg": "Invalid credentials",
  "request_id": "req-abc123"
}
```

## 2. Error Code Registry

### 2.1 Code Range Allocation

| Range       | Category            | Service        |
| ----------- | ------------------- | -------------- |
| 1000–1999   | Authentication      | auth-service   |
| 2000–2999   | User Management     | user-service   |
| 9000–9999   | General / System    | All services   |

### 2.2 Authentication Errors (1000–1999)

| Code | Name                          | HTTP Status | gRPC Status        | Description                                                |
| ---- | ----------------------------- | ----------- | ------------------ | ---------------------------------------------------------- |
| 1001 | INVALID_CREDENTIALS           | 401         | UNAUTHENTICATED    | Email/phone or password is incorrect                       |
| 1002 | ACCOUNT_LOCKED                | 423         | PERMISSION_DENIED  | Account locked due to excessive failed login attempts      |
| 1003 | IDENTIFIER_ALREADY_EXISTS     | 409         | ALREADY_EXISTS     | Email or phone number is already registered                |
| 1004 | INVALID_VERIFICATION_CODE     | 400         | INVALID_ARGUMENT   | Verification code is invalid, expired, or max attempts exceeded |
| 1006 | WEAK_PASSWORD                 | 400         | INVALID_ARGUMENT   | Password does not meet strength requirements               |
| 1007 | INVALID_TOKEN                 | 401         | UNAUTHENTICATED    | Token is invalid, expired, or revoked                      |
| 1008 | TOKEN_EXPIRED                 | 401         | UNAUTHENTICATED    | Access or refresh token has expired                        |
| 1009 | SOCIAL_AUTH_FAILED            | 401         | UNAUTHENTICATED    | OAuth provider authentication failed                       |
| 1010 | VERIFICATION_CODE_RATE_LIMITED| 429         | RESOURCE_EXHAUSTED | Too many verification code requests                        |
| 1011 | PASSWORD_SAME_AS_CURRENT      | 400         | INVALID_ARGUMENT   | New password cannot be the same as the current password     |

### 2.3 User Management Errors (2000–2999)

| Code | Name                          | HTTP Status | gRPC Status        | Description                                                |
| ---- | ----------------------------- | ----------- | ------------------ | ---------------------------------------------------------- |
| 2001 | USER_NOT_FOUND                | 404         | NOT_FOUND          | User with the specified ID does not exist                  |
| 2002 | INVALID_NICKNAME              | 400         | INVALID_ARGUMENT   | Nickname is empty or exceeds 30 characters                 |
| 2003 | INVALID_BIO                   | 400         | INVALID_ARGUMENT   | Bio exceeds 200 characters                                 |
| 2004 | INVALID_AVATAR_FORMAT         | 400         | INVALID_ARGUMENT   | Avatar image format not supported (allowed: jpeg, png, webp)|
| 2005 | AVATAR_TOO_LARGE              | 400         | INVALID_ARGUMENT   | Avatar image exceeds 5 MB size limit                       |
| 2006 | AVATAR_NOT_FOUND              | 404         | NOT_FOUND          | User does not have an avatar to delete                     |
| 2007 | PROFILE_NOT_FOUND             | 404         | NOT_FOUND          | User profile does not exist                                |

### 2.4 General / System Errors (9000–9999)

| Code | Name                          | HTTP Status | gRPC Status        | Description                                                |
| ---- | ----------------------------- | ----------- | ------------------ | ---------------------------------------------------------- |
| 9001 | INTERNAL_ERROR                | 500         | INTERNAL           | Unexpected server error                                    |
| 9002 | VALIDATION_ERROR              | 400         | INVALID_ARGUMENT   | Request payload failed validation                          |
| 9003 | RATE_LIMITED                  | 429         | RESOURCE_EXHAUSTED | Too many requests; rate limit exceeded                     |
| 9004 | UNAUTHORIZED                  | 401         | UNAUTHENTICATED    | Missing or invalid authentication token                    |
| 9005 | FORBIDDEN                     | 403         | PERMISSION_DENIED  | Insufficient permissions for the requested action          |
| 9006 | NOT_FOUND                     | 404         | NOT_FOUND          | Requested resource does not exist                          |
| 9007 | SERVICE_UNAVAILABLE           | 503         | UNAVAILABLE        | Downstream service is temporarily unavailable              |
| 9008 | REQUEST_TIMEOUT               | 504         | DEADLINE_EXCEEDED  | Request processing timed out                               |
| 9009 | IDEMPOTENCY_CONFLICT          | 409         | ALREADY_EXISTS     | Duplicate request with same idempotency key                |

## 3. gRPC Error Propagation

### 3.1 Translation Rules

1. gRPC services return errors using `google.golang.org/grpc/status` with the Rich Error Model.
2. Each gRPC error includes:
   - **gRPC status code** (e.g., `INVALID_ARGUMENT`, `NOT_FOUND`)
   - **Error detail** containing the application error `code` (integer) and `message` (string)
3. The gateway translates gRPC errors to REST responses using the mapping below.
4. If a gRPC error has no application error detail, the gateway maps the gRPC status code to a general error.

### 3.2 gRPC Status to HTTP Mapping

| gRPC Status        | HTTP Status | Default Error Code |
| ------------------ | ----------- | ------------------ |
| OK                 | 200         | —                  |
| INVALID_ARGUMENT   | 400         | 9002               |
| UNAUTHENTICATED    | 401         | 9004               |
| PERMISSION_DENIED  | 403         | 9005               |
| NOT_FOUND          | 404         | 9006               |
| ALREADY_EXISTS     | 409         | 9009               |
| RESOURCE_EXHAUSTED | 429         | 9003               |
| INTERNAL           | 500         | 9001               |
| UNAVAILABLE        | 503         | 9007               |
| DEADLINE_EXCEEDED  | 504         | 9008               |

> When the gRPC error includes an application-specific error code (e.g., `1001`), that code takes precedence over the default mapping.

## 4. Error Handling Guidelines

### 4.1 Gateway Layer

- Validate request format and required fields before forwarding to gRPC services
- Translate all gRPC errors to the standard REST JSON envelope
- Never expose internal error details (stack traces, internal IPs) to clients
- Include `request_id` in every error response for traceability
- Apply rate limiting errors (9003) before forwarding to backend services
- Return 401 with code 9004 for missing/invalid JWT tokens on protected endpoints

### 4.2 Auth Service

- Return specific error codes (1001–1011) for authentication-related failures
- Never disclose whether an identifier exists in error messages for login failures (use generic `INVALID_CREDENTIALS`)
- For password reset send-code, always return success to prevent user enumeration
- Log failed authentication attempts with identifier (hashed) and IP for security auditing
- On account lockout, return remaining lockout time in the error detail

### 4.3 User Service

- Return specific error codes (2001–2007) for user management failures
- Validate input constraints (nickname length, bio length, avatar format/size) and return appropriate error codes
- Return 404 (`USER_NOT_FOUND`) when the requested user does not exist
- Never expose internal user IDs in error messages to unauthorized requesters

### 4.4 Notification Service

- Log delivery failures but do not propagate errors to the client (asynchronous processing)
- Implement retry with exponential backoff for transient email/SMS provider failures
- Alert on persistent delivery failures via metrics/logging

## 5. Logging & Observability

| Log Level | When to Use                                                                                 |
| --------- | ------------------------------------------------------------------------------------------- |
| ERROR     | Unexpected failures: database errors, external service failures, unhandled exceptions        |
| WARN      | Expected but notable events: failed login attempts, rate limit hits, account lockouts        |
| INFO      | Successful operations: user registered, login successful, password changed, token refreshed  |
| DEBUG     | Detailed processing information: request/response payloads (sanitized), cache hits/misses    |

**Logging Rules:**

- Every log entry must include `request_id`, `service`, and `timestamp`
- Authentication logs must include `identifier` (hashed), `ip_address`, and `user_agent`
- Never log passwords, tokens, verification codes, or other secrets in plaintext
- Structured JSON logging format for all services
- Log rotation and retention configured per environment
