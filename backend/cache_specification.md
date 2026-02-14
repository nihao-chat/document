# Cache Strategy

| Date       | Description                                |
| ---------- | ------------------------------------------ |
| 2026-02-14 | FEAT-USER-001: User Account Management     |

## 1. Overview

This document defines the Redis caching strategy for nihao-chat backend services. Redis is used for session management, account lockout tracking, verification code storage, and rate limiting. All cache items are designed with appropriate TTLs and key patterns to support the authentication and user management features.

## 2. Cache Items

### 2.1 Session Management

| Key Pattern                          | Value                                                    | TTL   | Service      | Purpose                                  |
| ------------------------------------ | -------------------------------------------------------- | ----- | ------------ | ---------------------------------------- |
| `session:{user_id}:{token_id}`       | `{"user_id": "...", "created_at": "...", "user_agent": "..."}` | 7 days | auth-service | Refresh token session metadata           |
| `user_sessions:{user_id}`           | Set of `token_id` strings                                      | None (managed explicitly) | auth-service | Track all active session token IDs for a user |

### 2.2 Account Lockout

| Key Pattern                          | Value                                                    | TTL    | Service      | Purpose                                  |
| ------------------------------------ | -------------------------------------------------------- | ------ | ------------ | ---------------------------------------- |
| `lockout:{identifier}`               | Hash — `attempts`: N, `locked_at`: "..."                 | 15 min | auth-service | Track failed login attempts and lockout  |

### 2.3 Verification Codes

| Key Pattern                          | Value                                                    | TTL    | Service      | Purpose                                  |
| ------------------------------------ | -------------------------------------------------------- | ------ | ------------ | ---------------------------------------- |
| `verify:{identifier}:{purpose}`      | `{"code_hash": "...", "identifier_type": "email\|phone", "attempts": N}` | 10 min | auth-service | Store verification code (self-contained, no DB) |

### 2.4 Rate Limiting

| Key Pattern                          | Value   | TTL     | Service | Purpose                                        |
| ------------------------------------ | ------- | ------- | ------- | ---------------------------------------------- |
| `ratelimit:{endpoint}:{ip}`          | Counter | Varies  | gateway | Track request counts per endpoint per IP       |

### 2.5 Idempotency Keys

| Key Pattern                          | Value                                                    | TTL      | Service | Purpose                                          |
| ------------------------------------ | -------------------------------------------------------- | -------- | ------- | ------------------------------------------------ |
| `idempotency:{idempotency_key}`      | `{"status_code": N, "headers": {...}, "body": {...}}`    | 24 hours | gateway | Store response for duplicate request detection   |

## 3. Operations by Feature

### 3.1 Login

| Step                              | Redis Operation                                                                 |
| --------------------------------- | ------------------------------------------------------------------------------- |
| Check account lockout             | `GET lockout:{identifier}`                                                      |
| Record failed attempt             | `HINCRBY lockout:{identifier} attempts 1` + `EXPIRE lockout:{identifier} 900`  |
| Trigger lockout (5th failure)     | `HSET lockout:{identifier} locked_at {timestamp}` + `EXPIRE lockout:{identifier} 900` |
| Clear lockout on success          | `DEL lockout:{identifier}`                                                      |
| Store session on login            | `SET session:{user_id}:{token_id} {metadata} EX 604800`                        |
| Track session in user set         | `SADD user_sessions:{user_id} {token_id}`                                      |

### 3.2 Token Refresh

| Step                              | Redis Operation                                                                 |
| --------------------------------- | ------------------------------------------------------------------------------- |
| Validate refresh token exists     | `GET session:{user_id}:{token_id}`                                              |
| Delete old session                | `DEL session:{user_id}:{token_id}`                                              |
| Remove old token from user set    | `SREM user_sessions:{user_id} {token_id}`                                       |
| Store new session                 | `SET session:{user_id}:{new_token_id} {metadata} EX 604800`                    |
| Track new session in user set     | `SADD user_sessions:{user_id} {new_token_id}`                                  |

### 3.3 Logout

| Step                              | Redis Operation                                                                 |
| --------------------------------- | ------------------------------------------------------------------------------- |
| Delete session                    | `DEL session:{user_id}:{token_id}`                                              |
| Remove token from user set        | `SREM user_sessions:{user_id} {token_id}`                                       |

### 3.4 Registration — Send Verification Code

| Step                              | Redis Operation                                                                 |
| --------------------------------- | ------------------------------------------------------------------------------- |
| Store verification code           | `SET verify:{identifier}:registration {code_hash, identifier_type, attempts: 0} EX 600` |

### 3.5 Registration — Verify Code

| Step                              | Redis Operation                                                                 |
| --------------------------------- | ------------------------------------------------------------------------------- |
| Get verification code             | `GET verify:{identifier}:registration`                                          |
| Increment attempt on failure      | Update JSON `attempts` field and `SET verify:{identifier}:registration {updated_value} KEEPTTL` |
| Delete code on success            | `DEL verify:{identifier}:registration`                                          |
| Store session after registration  | `SET session:{user_id}:{token_id} {metadata} EX 604800`                        |
| Track session in user set         | `SADD user_sessions:{user_id} {token_id}`                                      |

### 3.6 Password Reset — Send Code

| Step                              | Redis Operation                                                                 |
| --------------------------------- | ------------------------------------------------------------------------------- |
| Store verification code           | `SET verify:{identifier}:password_reset {code_hash, identifier_type, attempts: 0} EX 600` |

### 3.7 Password Reset — Reset Password

| Step                              | Redis Operation                                                                 |
| --------------------------------- | ------------------------------------------------------------------------------- |
| Get verification code             | `GET verify:{identifier}:password_reset`                                        |
| Increment attempt on failure      | Update JSON `attempts` field and `SET verify:{identifier}:password_reset {updated_value} KEEPTTL` |
| Delete code on success            | `DEL verify:{identifier}:password_reset`                                        |
| Get all user session tokens       | `SMEMBERS user_sessions:{user_id}`                                              |
| Delete each session               | `DEL session:{user_id}:{token_id}` for each token_id                           |
| Delete user sessions set          | `DEL user_sessions:{user_id}`                                                   |

### 3.8 Change Password

| Step                              | Redis Operation                                                                 |
| --------------------------------- | ------------------------------------------------------------------------------- |
| Get all user session tokens       | `SMEMBERS user_sessions:{user_id}`                                              |
| Delete other sessions             | For each `token_id` != `current_token_id`: `DEL session:{user_id}:{token_id}` + `SREM user_sessions:{user_id} {token_id}` |

### 3.9 Rate Limiting

| Step                              | Redis Operation                                                                 |
| --------------------------------- | ------------------------------------------------------------------------------- |
| Increment request count           | `INCR ratelimit:{endpoint}:{ip}`                                                |
| Set TTL on first request          | `EXPIRE ratelimit:{endpoint}:{ip} {window_seconds}` (only if TTL not set)      |
| Check rate limit                  | `GET ratelimit:{endpoint}:{ip}` and compare against limit                       |

### 3.10 Idempotency Check

| Step                              | Redis Operation                                                                 |
| --------------------------------- | ------------------------------------------------------------------------------- |
| Check for existing key            | `GET idempotency:{idempotency_key}`                                             |
| Store response on first request   | `SET idempotency:{idempotency_key} {response} EX 86400`                        |

## 4. Eviction & Memory Policy

| Setting              | Value                | Description                                               |
| -------------------- | -------------------- | --------------------------------------------------------- |
| `maxmemory-policy`   | `volatile-ttl`       | Evict keys with TTL set, starting with shortest remaining |
| `maxmemory`          | Configured per environment | Maximum memory allocation for Redis instance         |

> All cache keys used by nihao-chat have explicit TTLs. The `volatile-ttl` policy ensures that if memory pressure occurs, keys closest to expiration are evicted first, preserving longer-lived session data.

## 5. Failure Strategy

| Failure Scenario               | Behavior                                                                              |
| ------------------------------ | ------------------------------------------------------------------------------------- |
| Redis unavailable (session)    | Reject token refresh requests; access tokens continue to work until expiry            |
| Redis unavailable (lockout)    | Allow login attempts (fail-open); log warning for monitoring                          |
| Redis unavailable (verify)     | Reject verification attempt (fail-closed); log warning for monitoring                 |
| Redis unavailable (rate limit) | Allow requests through (fail-open); log warning and alert ops team                    |
| Redis unavailable (idempotency) | Process request normally (fail-open); log warning — risk of duplicate processing     |
| Key missing unexpectedly       | Treat as expired/invalid; require user to re-authenticate or re-request code          |
| Redis connection timeout       | Retry once with 100ms timeout; if still failing, apply fail-open/fail-closed per item |

## 6. Key Design

### 6.1 Avoiding SCAN

Avoid using `SCAN` in production request paths. All keys must be directly addressable; use index structures (Set, List) to track related keys.
