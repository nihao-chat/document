# gRPC Service Specification

| Date       | Description                                                          |
| ---------- | -------------------------------------------------------------------- |
| 2026-02-20 | Replace custom ErrorDetail with Google standard error detail types   |
| 2026-02-14 | FEAT-USER-001: User Account Management                               |

## 1. Overview

This document defines the gRPC service interfaces for nihao-chat backend services. All service definitions use proto3 syntax. Internal communication between the gateway and backend services, as well as inter-service calls, use gRPC.

## 2. Common Types

**File:** `proto/common/v1/common.proto`

```protobuf
syntax = "proto3";

package common.v1;

import "google/protobuf/timestamp.proto";

// Identifier type enum
enum IdentifierType {
  IDENTIFIER_TYPE_UNKNOWN = 0;
  IDENTIFIER_TYPE_EMAIL = 1;
  IDENTIFIER_TYPE_PHONE = 2;
}

// Verification code purpose enum
enum VerificationPurpose {
  VERIFICATION_PURPOSE_UNKNOWN = 0;
  VERIFICATION_PURPOSE_REGISTRATION = 1;
  VERIFICATION_PURPOSE_PASSWORD_RESET = 2;
}

// OAuth provider enum
enum OAuthProvider {
  OAUTH_PROVIDER_UNKNOWN = 0;
  OAUTH_PROVIDER_GOOGLE = 1;
  OAUTH_PROVIDER_APPLE = 2;
}
```

### Google Standard Error Detail Types

Error details use the standard types from [`google/rpc/error_details.proto`](https://github.com/googleapis/googleapis/blob/master/google/rpc/error_details.proto). The project uses the following types:

| Type             | Fields                                         | Usage                                   |
| ---------------- | ---------------------------------------------- | --------------------------------------- |
| `BadRequest`     | `field_violations[]` → `field`, `description`  | Field-level validation errors           |
| `ErrorInfo`      | `reason`, `domain`, `metadata`                 | Application-specific error reason       |
| `ResourceInfo`   | `resource_type`, `resource_name`, `owner`, `description` | Resource not found or already exists |
| `QuotaFailure`   | `violations[]` → `subject`, `description`      | Rate limit or quota exceeded            |
| `RetryInfo`      | `retry_delay`                                  | Suggested retry delay                   |
| `DebugInfo`      | `stack_entries[]`, `detail`                    | Internal debug information              |

## 3. Naming Conventions

| Element          | Convention          | Example                        |
| ---------------- | ------------------- | ------------------------------ |
| File             | `snake_case.proto`  | `auth_service.proto`           |
| Package          | `domain.subdomain.v1` | `auth.v1`                    |
| Message          | `PascalCase`        | `LoginRequest`                 |
| Field            | `snake_case`        | `user_id`                      |
| Service          | `PascalCase`        | `AuthService`                  |
| RPC method       | `PascalCase`        | `Login`                        |
| Enum name        | `PascalCase`        | `IdentifierType`               |
| Enum value       | `CAPS_SNAKE_CASE`   | `IDENTIFIER_TYPE_EMAIL`        |
| First enum value | `*_UNKNOWN = 0`     | `IDENTIFIER_TYPE_UNKNOWN = 0`  |

## 4. Request/Response

Every RPC must have a unique `Request` and `Response` message. Message reuse across different RPCs is strictly prohibited. This ensures each RPC can evolve independently without breaking changes.

## 5. Compatibility

- Never change existing field numbers
- Use `reserved` keyword for deprecated field tags and names
- Add new fields with new field numbers only
- Enum values must never be reordered or removed; add `reserved` for deprecated values

## 6. Pagination

### 6.1 Cursor-Based Pagination

Used for real-time or frequently updated lists (e.g., messages, notifications):

| Field              | Type   | Description                                              |
| ------------------ | ------ | -------------------------------------------------------- |
| `page_token`       | string | Opaque token for the next page (from previous response)  |
| `page_size`        | int32  | Number of items per page (default: 20, max: 100)         |
| `next_page_token`  | string | Token for the next page (empty if no more pages)         |

```protobuf
// Include in list requests
string page_token = 1;
int32 page_size = 2;

// Include in list responses
repeated Item items = 1;
string next_page_token = 2;
```

### 6.2 Offset-Based Pagination

Used for stable, sortable lists (e.g., admin user lists, search results):

| Field        | Type   | Description                                |
| ------------ | ------ | ------------------------------------------ |
| `page`       | int32  | Page number (1-based, default: 1)          |
| `page_size`  | int32  | Number of items per page (default: 20, max: 100) |
| `total`      | int32  | Total number of items                      |

```protobuf
// Include in list requests
int32 page = 1;
int32 page_size = 2;

// Include in list responses
repeated Item items = 1;
int32 total = 2;
int32 page = 3;
int32 page_size = 4;
```

## 7. Error Handling

### 7.1 Overview

Errors are returned via `google.golang.org/grpc/status` with the [Google Rich Error Model](https://cloud.google.com/apis/design/errors#error_model). Each error includes:

1. A gRPC status code (e.g., `INVALID_ARGUMENT`, `NOT_FOUND`)
2. A status message (human-readable description)
3. Zero or more standard detail types from `google/rpc/error_details.proto` attached via `status.WithDetails()`

No custom error detail messages are used. All error details use the Google standard types defined in Section 2.

### 7.2 Error Detail Type Mapping

| gRPC Status        | HTTP | Google Detail Type | Usage                                       |
| ------------------ | ---- | ------------------ | ------------------------------------------- |
| INVALID_ARGUMENT   | 400  | `BadRequest`       | Field-level validation errors (FieldViolation) |
| UNAUTHENTICATED    | 401  | `ErrorInfo`        | Authentication failure reason               |
| PERMISSION_DENIED  | 403  | `ErrorInfo`        | Insufficient permissions reason             |
| NOT_FOUND          | 404  | `ResourceInfo`     | Information about the missing resource      |
| ALREADY_EXISTS     | 409  | `ResourceInfo`     | Information about the conflicting resource  |
| RESOURCE_EXHAUSTED | 429  | `QuotaFailure`     | Quota or rate limit violation               |
| INTERNAL           | 500  | `DebugInfo`        | Debug information (never exposed to client) |
| UNAVAILABLE        | 503  | `RetryInfo`        | Suggested retry delay                       |
| DEADLINE_EXCEEDED  | 504  | —                  | Status message only                         |

### 7.3 Go Examples

**BadRequest — field validation errors:**

```go
st := status.New(codes.InvalidArgument, "Validation failed")
br := &errdetails.BadRequest{}
br.FieldViolations = append(br.FieldViolations,
    &errdetails.BadRequest_FieldViolation{
        Field:       "password",
        Description: "Password must be at least 8 characters",
    },
    &errdetails.BadRequest_FieldViolation{
        Field:       "nickname",
        Description: "Nickname must not be empty",
    },
)
st, _ = st.WithDetails(br)
return nil, st.Err()
```

**ErrorInfo — application-specific error:**

```go
st := status.New(codes.Unauthenticated, "Invalid credentials")
ei := &errdetails.ErrorInfo{
    Reason: "Email or password is incorrect",
    Domain: "auth.nihao.chat",
}
st, _ = st.WithDetails(ei)
return nil, st.Err()
```

## 8. Auth Service

### 8.1 Service Definition

**File:** `proto/auth/v1/auth_service.proto`

```protobuf
syntax = "proto3";

package auth.v1;

import "common/v1/common.proto";

service AuthService {
  // Registration
  rpc SendVerificationCode(SendVerificationCodeRequest) returns (SendVerificationCodeResponse);
  rpc Register(RegisterRequest) returns (RegisterResponse);

  // Login
  rpc Login(LoginRequest) returns (LoginResponse);
  rpc SocialLogin(SocialLoginRequest) returns (SocialLoginResponse);

  // Token
  rpc RefreshToken(RefreshTokenRequest) returns (RefreshTokenResponse);

  // Logout
  rpc Logout(LogoutRequest) returns (LogoutResponse);

  // Password
  rpc ChangePassword(ChangePasswordRequest) returns (ChangePasswordResponse);
  rpc SendPasswordResetCode(SendPasswordResetCodeRequest) returns (SendPasswordResetCodeResponse);
  rpc ResetPassword(ResetPasswordRequest) returns (ResetPasswordResponse);
}
```

#### SendVerificationCode

Sends a verification code to the specified email or phone number.

| gRPC Status        | Description                         |
| ------------------ | ----------------------------------- |
| RESOURCE_EXHAUSTED | Too many verification code requests |

#### Register

Registers a new user account with verification code and password.

| gRPC Status      | Description                                                       |
| ---------------- | ----------------------------------------------------------------- |
| ALREADY_EXISTS   | Email or phone number is already registered                       |
| INVALID_ARGUMENT | Verification code is invalid, expired, or max attempts exceeded   |
| INVALID_ARGUMENT | Password does not meet strength requirements                      |

#### Login

Authenticates a user with email/phone and password.

| gRPC Status       | Description                                           |
| ----------------- | ----------------------------------------------------- |
| UNAUTHENTICATED   | Email/phone or password is incorrect                  |
| PERMISSION_DENIED | Account locked due to excessive failed login attempts |

#### SocialLogin

Authenticates a user via Google or Apple OAuth.

| gRPC Status     | Description                            |
| --------------- | -------------------------------------- |
| UNAUTHENTICATED | OAuth provider authentication failed   |

#### RefreshToken

Issues a new access token using a refresh token.

| gRPC Status     | Description                            |
| --------------- | -------------------------------------- |
| UNAUTHENTICATED | Token is invalid, expired, or revoked  |

#### Logout

Revokes the user's session token.

_No service-specific errors._

#### ChangePassword

Changes the authenticated user's password.

| gRPC Status      | Description                                                |
| ---------------- | ---------------------------------------------------------- |
| INVALID_ARGUMENT | New password cannot be the same as the current password    |
| INVALID_ARGUMENT | Password does not meet strength requirements               |

#### SendPasswordResetCode

Sends a password reset verification code.

| gRPC Status        | Description                         |
| ------------------ | ----------------------------------- |
| RESOURCE_EXHAUSTED | Too many verification code requests |

#### ResetPassword

Resets the user's password using a verification code.

| gRPC Status      | Description                                                       |
| ---------------- | ----------------------------------------------------------------- |
| INVALID_ARGUMENT | Verification code is invalid, expired, or max attempts exceeded   |
| INVALID_ARGUMENT | Password does not meet strength requirements                      |

### 8.2 Message Definitions

#### Registration Messages

```protobuf
message SendVerificationCodeRequest {
  string identifier = 1;
  common.v1.IdentifierType identifier_type = 2;
  common.v1.VerificationPurpose purpose = 3;
}

message SendVerificationCodeResponse {
  int32 expires_in = 1; // Seconds until code expires
}

message RegisterRequest {
  string identifier = 1;
  common.v1.IdentifierType identifier_type = 2;
  string code = 3;
  string password = 4;
  string nickname = 5;
}

message RegisterResponse {
  string user_id = 1;
  string access_token = 2;
  string refresh_token = 3;
  int32 expires_in = 4; // Access token TTL in seconds
}
```

#### Login Messages

```protobuf
message LoginRequest {
  string identifier = 1;
  common.v1.IdentifierType identifier_type = 2;
  string password = 3;
}

message LoginResponse {
  string user_id = 1;
  string access_token = 2;
  string refresh_token = 3;
  int32 expires_in = 4;
}

message SocialLoginRequest {
  common.v1.OAuthProvider provider = 1;
  string id_token = 2;
  string authorization_code = 3; // Required for Apple, optional for Google
}

message SocialLoginResponse {
  string user_id = 1;
  string access_token = 2;
  string refresh_token = 3;
  int32 expires_in = 4;
  bool is_new_user = 5;
}
```

#### Token Messages

```protobuf
message RefreshTokenRequest {
  string refresh_token = 1;
}

message RefreshTokenResponse {
  string access_token = 1;
  string refresh_token = 2;
  int32 expires_in = 3;
}
```

#### Logout Messages

```protobuf
message LogoutRequest {
  string user_id = 1;
  string token_id = 2;
}

message LogoutResponse {}
```

#### Password Messages

```protobuf
message ChangePasswordRequest {
  string user_id = 1;
  string current_password = 2;
  string new_password = 3;
}

message ChangePasswordResponse {}

message SendPasswordResetCodeRequest {
  string identifier = 1;
  common.v1.IdentifierType identifier_type = 2;
}

message SendPasswordResetCodeResponse {
  int32 expires_in = 1;
}

message ResetPasswordRequest {
  string identifier = 1;
  common.v1.IdentifierType identifier_type = 2;
  string code = 3;
  string new_password = 4;
}

message ResetPasswordResponse {}
```

## 9. User Service

### 9.1 Service Definition

**File:** `proto/user/v1/user_service.proto`

```protobuf
syntax = "proto3";

package user.v1;

import "google/protobuf/timestamp.proto";
import "common/v1/common.proto";

service UserService {
  // User CRUD
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc GetUserByIdentifier(GetUserByIdentifierRequest) returns (GetUserByIdentifierResponse);

  // Profile
  rpc UpdateProfile(UpdateProfileRequest) returns (UpdateProfileResponse);
  rpc GetProfile(GetProfileRequest) returns (GetProfileResponse);

  // Avatar
  rpc GenerateAvatarPresignedPost(GenerateAvatarPresignedPostRequest) returns (GenerateAvatarPresignedPostResponse);
  rpc DeleteAvatar(DeleteAvatarRequest) returns (DeleteAvatarResponse);
}
```

#### CreateUser

Creates a new user record (called by auth-service during registration).

| gRPC Status      | Description                                |
| ---------------- | ------------------------------------------ |
| INVALID_ARGUMENT | Nickname is empty or exceeds 30 characters |

#### GetUser

Retrieves a user by ID.

| gRPC Status | Description                               |
| ----------- | ----------------------------------------- |
| NOT_FOUND   | User with the specified ID does not exist |

#### GetUserByIdentifier

Retrieves a user by email or phone number.

| gRPC Status | Description                                       |
| ----------- | ------------------------------------------------- |
| NOT_FOUND   | User with the specified identifier does not exist |

#### UpdateProfile

Updates the user's profile fields.

| gRPC Status      | Description                                |
| ---------------- | ------------------------------------------ |
| NOT_FOUND        | User profile does not exist                |
| INVALID_ARGUMENT | Nickname is empty or exceeds 30 characters |
| INVALID_ARGUMENT | Bio exceeds 200 characters                 |

#### GetProfile

Retrieves a user's public profile.

| gRPC Status | Description                 |
| ----------- | --------------------------- |
| NOT_FOUND   | User profile does not exist |

#### GenerateAvatarPresignedPost

Generates a presigned POST URL for avatar upload.

| gRPC Status      | Description                          |
| ---------------- | ------------------------------------ |
| INVALID_ARGUMENT | Avatar image format not supported    |
| INVALID_ARGUMENT | Avatar image exceeds 5 MB size limit |

#### DeleteAvatar

Deletes the user's avatar.

| gRPC Status | Description                            |
| ----------- | -------------------------------------- |
| NOT_FOUND   | User does not have an avatar to delete |

### 9.2 Message Definitions

#### User CRUD Messages

```protobuf
message CreateUserRequest {
  string identifier = 1;
  common.v1.IdentifierType identifier_type = 2;
  string nickname = 3;
}

message CreateUserResponse {
  string user_id = 1;
}

message GetUserRequest {
  string user_id = 1;
}

message GetUserResponse {
  string user_id = 1;
  string email = 2;
  string phone = 3;
  string nickname = 4;
  string status = 5;
  string role = 6;
  string avatar_url = 7;
  string bio = 8;
  google.protobuf.Timestamp created_at = 9;
}

message GetUserByIdentifierRequest {
  string identifier = 1;
  common.v1.IdentifierType identifier_type = 2;
}

message GetUserByIdentifierResponse {
  string user_id = 1;
  string email = 2;
  string phone = 3;
  string nickname = 4;
  string status = 5;
  string role = 6;
}
```

#### Profile Messages

```protobuf
message UpdateProfileRequest {
  string user_id = 1;
  optional string nickname = 2;
  optional string bio = 3;
}

message UpdateProfileResponse {
  string user_id = 1;
  string nickname = 2;
  string bio = 3;
}

message GetProfileRequest {
  string user_id = 1;
}

message GetProfileResponse {
  string user_id = 1;
  string nickname = 2;
  string avatar_url = 3;
  string bio = 4;
}
```

#### Avatar Messages

```protobuf
message GenerateAvatarPresignedPostRequest {
  string user_id = 1;
  string content_type = 2;
  int64 content_length = 3;
}

message GenerateAvatarPresignedPostResponse {
  string url = 1;
  map<string, string> fields = 2;
}

message DeleteAvatarRequest {
  string user_id = 1;
}

message DeleteAvatarResponse {}
```

## 10. Inter-Service Communication

| Caller         | Callee         | RPC                    | When                                                          |
| -------------- | -------------- | ---------------------- | ------------------------------------------------------------- |
| gateway        | auth-service   | SendVerificationCode   | Client requests registration verification code                |
| gateway        | auth-service   | Register               | Client submits registration form                              |
| gateway        | auth-service   | Login                  | Client logs in with email/phone + password                    |
| gateway        | auth-service   | SocialLogin            | Client logs in with Google or Apple                           |
| gateway        | auth-service   | RefreshToken           | Client refreshes access token                                 |
| gateway        | auth-service   | Logout                 | Client logs out                                               |
| gateway        | auth-service   | ChangePassword         | Client changes password                                       |
| gateway        | auth-service   | SendPasswordResetCode  | Client requests password reset code                           |
| gateway        | auth-service   | ResetPassword          | Client resets password with verification code                 |
| gateway        | user-service   | GetUser                | Client requests own profile (GET /users/me)                   |
| gateway        | user-service   | UpdateProfile          | Client updates profile (PATCH /users/me)                      |
| gateway        | user-service   | GetProfile             | Client views another user's public profile                    |
| gateway        | user-service   | GenerateAvatarPresignedPost | Client requests avatar upload URL                        |
| gateway        | user-service   | DeleteAvatar           | Client deletes avatar                                         |
| auth-service   | user-service   | CreateUser             | During registration after code verification                   |
| auth-service   | user-service   | GetUserByIdentifier    | During login to look up user by email/phone                   |
| auth-service   | user-service   | GetUserByIdentifier    | During social login to check if user exists                   |
| auth-service   | user-service   | GetUserByIdentifier    | During password reset to find user                            |
