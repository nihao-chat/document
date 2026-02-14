# Event & Messaging Specification

| Date       | Description                                |
| ---------- | ------------------------------------------ |
| 2026-02-14 | FEAT-USER-001: User Account Management     |

## 1. Overview

This document defines the event-driven messaging architecture for nihao-chat backend services. The system uses Apache Kafka as the message broker for asynchronous communication between services. Currently, event-driven messaging is used for notification delivery (email and SMS), decoupling the auth-service from external email and SMS providers.

## 2. Message Broker

### 2.1 Kafka Cluster Configuration

| Setting                | Value           | Description                                         |
| ---------------------- | --------------- | --------------------------------------------------- |
| Broker count           | 3               | Minimum brokers for production cluster              |
| Replication factor     | 3               | Number of replicas per partition                    |
| Min in-sync replicas   | 2               | Minimum ISR for producer `acks=all`                 |
| Default partitions     | 3               | Default partition count for new topics              |
| Log retention (default)| 7 days          | Default message retention period                    |
| Message max bytes      | 1 MB            | Maximum message size                                |
| Compression            | snappy          | Producer-side compression codec                     |
| Security protocol      | SASL_SSL        | Authentication and encryption for Kafka connections |
| SASL mechanism         | SCRAM-SHA-256   | SASL authentication mechanism                       |

### 2.2 Topic Naming Convention

Topics follow the pattern: `{domain}.{channel}`

| Segment   | Description                        | Example        |
| --------- | ---------------------------------- | -------------- |
| `domain`  | Business domain or service area    | `notification` |
| `channel` | Delivery channel or sub-category   | `email`, `sms` |

Dead-letter topics append `.dlq`: `{domain}.{channel}.dlq`

### 2.3 Environment-Specific Overrides

| Setting              | Development | Staging | Production |
| -------------------- | ----------- | ------- | ---------- |
| Broker count         | 1           | 3       | 3          |
| Replication factor   | 1           | 3       | 3          |
| Min in-sync replicas | 1           | 2       | 2          |
| Default partitions   | 1           | 3       | 3          |
| Log retention        | 1 day       | 3 days  | 7 days     |

## 3. Topic Registry

| Topic Name               | Description                                   | Partitions | Retention | Cleanup Policy |
| ------------------------- | --------------------------------------------- | ---------- | --------- | -------------- |
| `notification.email`      | Email notification delivery requests          | 3          | 7 days    | delete         |
| `notification.sms`        | SMS notification delivery requests            | 3          | 7 days    | delete         |
| `notification.email.dlq`  | Dead-letter queue for failed email deliveries | 1          | 14 days   | delete         |
| `notification.sms.dlq`    | Dead-letter queue for failed SMS deliveries   | 1          | 14 days   | delete         |

## 4. Event Catalog

| Topic                | Event Type                | Producer     | Consumer     | Description                                                              |
| -------------------- | ------------------------- | ------------ | ------------ | ------------------------------------------------------------------------ |
| `notification.email` | `verification_code.email` | auth-service | notification | Send verification code via email (registration, password reset)          |
| `notification.sms`   | `verification_code.sms`   | auth-service | notification | Send verification code via SMS (registration, password reset)            |

## 5. Event Schemas

### 5.1 verification_code.email

**Event Type:** `verification_code.email`
**Version:** `1.0`
**Topic:** `notification.email`

**Schema:**

```json
{
  "event_id": "string (UUID)",
  "event_type": "verification_code.email",
  "version": "1.0",
  "timestamp": "string (ISO 8601)",
  "source": "auth-service",
  "data": {
    "recipient_email": "string",
    "verification_code": "string",
    "purpose": "string (registration | password_reset)",
    "expires_in": "integer",
    "locale": "string"
  }
}
```

**Field Table:**

| Field                    | Type    | Required | Description                                            |
| ------------------------ | ------- | -------- | ------------------------------------------------------ |
| `event_id`               | string  | Yes      | Unique event identifier (UUID v4)                      |
| `event_type`             | string  | Yes      | Event type: `verification_code.email`                  |
| `version`                | string  | Yes      | Schema version: `1.0`                                  |
| `timestamp`              | string  | Yes      | Event creation time (ISO 8601)                         |
| `source`                 | string  | Yes      | Producing service: `auth-service`                      |
| `data.recipient_email`   | string  | Yes      | Recipient email address                                |
| `data.verification_code` | string  | Yes      | 6-digit verification code (plaintext for delivery)     |
| `data.purpose`           | string  | Yes      | Code purpose: `registration` or `password_reset`       |
| `data.expires_in`        | integer | Yes      | Code validity duration in seconds (600)                |
| `data.locale`            | string  | No       | Preferred language for the email template (default: `en`) |

**Example Payload:**

```json
{
  "event_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "event_type": "verification_code.email",
  "version": "1.0",
  "timestamp": "2026-02-14T10:30:00Z",
  "source": "auth-service",
  "data": {
    "recipient_email": "user@example.com",
    "verification_code": "123456",
    "purpose": "registration",
    "expires_in": 600,
    "locale": "en"
  }
}
```

**Schema Versioning:** Backward compatible. New optional fields may be added in future versions. Consumers must ignore unknown fields.

### 5.2 verification_code.sms

**Event Type:** `verification_code.sms`
**Version:** `1.0`
**Topic:** `notification.sms`

**Schema:**

```json
{
  "event_id": "string (UUID)",
  "event_type": "verification_code.sms",
  "version": "1.0",
  "timestamp": "string (ISO 8601)",
  "source": "auth-service",
  "data": {
    "recipient_phone": "string",
    "verification_code": "string",
    "purpose": "string (registration | password_reset)",
    "expires_in": "integer",
    "locale": "string"
  }
}
```

**Field Table:**

| Field                    | Type    | Required | Description                                            |
| ------------------------ | ------- | -------- | ------------------------------------------------------ |
| `event_id`               | string  | Yes      | Unique event identifier (UUID v4)                      |
| `event_type`             | string  | Yes      | Event type: `verification_code.sms`                    |
| `version`                | string  | Yes      | Schema version: `1.0`                                  |
| `timestamp`              | string  | Yes      | Event creation time (ISO 8601)                         |
| `source`                 | string  | Yes      | Producing service: `auth-service`                      |
| `data.recipient_phone`   | string  | Yes      | Recipient phone number in E.164 format                 |
| `data.verification_code` | string  | Yes      | 6-digit verification code (plaintext for delivery)     |
| `data.purpose`           | string  | Yes      | Code purpose: `registration` or `password_reset`       |
| `data.expires_in`        | integer | Yes      | Code validity duration in seconds (600)                |
| `data.locale`            | string  | No       | Preferred language for the SMS template (default: `en`) |

**Example Payload:**

```json
{
  "event_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "event_type": "verification_code.sms",
  "version": "1.0",
  "timestamp": "2026-02-14T10:30:00Z",
  "source": "auth-service",
  "data": {
    "recipient_phone": "+886912345678",
    "verification_code": "654321",
    "purpose": "password_reset",
    "expires_in": 600,
    "locale": "zh"
  }
}
```

**Schema Versioning:** Backward compatible. New optional fields may be added in future versions. Consumers must ignore unknown fields.

## 6. Producer & Consumer Contracts

### 6.1 auth-service (Producer)

**Events Produced:**

| Event Type                | Topic                | Trigger Condition                                       |
| ------------------------- | -------------------- | ------------------------------------------------------- |
| `verification_code.email` | `notification.email` | Registration or password reset code requested for email |
| `verification_code.sms`   | `notification.sms`   | Registration or password reset code requested for phone |

**Delivery Guarantee:** At-least-once

**Producer Configuration:**

| Setting                | Value  | Description                                          |
| ---------------------- | ------ | ---------------------------------------------------- |
| `acks`                 | `all`  | Wait for all in-sync replicas to acknowledge         |
| `retries`              | 3      | Number of retry attempts on transient failures       |
| `retry.backoff.ms`     | 100    | Backoff between retries                              |
| `enable.idempotence`   | `true` | Ensure exactly-once delivery to the broker           |

### 6.2 notification (Consumer)

**Events Consumed:**

| Event Type                | Topic                | Consumer Group           | Processing Description                                   |
| ------------------------- | -------------------- | ------------------------ | --------------------------------------------------------- |
| `verification_code.email` | `notification.email` | `notification-email-cg`  | Render email template and send via external email provider |
| `verification_code.sms`   | `notification.sms`   | `notification-sms-cg`    | Render SMS template and send via external SMS provider     |

**Delivery Guarantee:** At-least-once

**Idempotency Strategy:** The notification consumer uses the `event_id` as a deduplication key. Before processing, the consumer checks if the `event_id` has already been processed (tracked via an in-memory LRU cache with 1-hour TTL). If already processed, the event is acknowledged without reprocessing. Duplicate deliveries are acceptable but minimized through this mechanism.

**Consumer Configuration:**

| Setting                  | Value      | Description                                          |
| ------------------------ | ---------- | ---------------------------------------------------- |
| `enable.auto.commit`     | `false`    | Manual offset commit after successful processing     |
| `auto.offset.reset`      | `earliest` | Start from earliest unprocessed message              |
| `max.poll.records`       | 10         | Maximum records per poll                             |
| `session.timeout.ms`     | 30000      | Consumer session timeout                             |
| `heartbeat.interval.ms`  | 10000      | Heartbeat interval                                   |

## 7. Retry & Dead-Letter

### 7.1 Retry Policy

| Topic                | Max Retries | Backoff Strategy    | Initial Delay | Max Delay  |
| -------------------- | ----------- | ------------------- | ------------- | ---------- |
| `notification.email` | 3           | Exponential backoff | 1 second      | 30 seconds |
| `notification.sms`   | 3           | Exponential backoff | 1 second      | 30 seconds |

### 7.2 DLQ Topic Naming Convention

Dead-letter topics follow the pattern: `{original_topic}.dlq`

| Original Topic       | DLQ Topic                |
| -------------------- | ------------------------ |
| `notification.email` | `notification.email.dlq` |
| `notification.sms`   | `notification.sms.dlq`   |

### 7.3 DLQ Processing Guidelines

- Events are moved to the DLQ after exhausting all retry attempts
- DLQ messages retain the original event payload plus additional metadata:
  - `original_topic`: The source topic
  - `failure_reason`: Description of the last failure
  - `retry_count`: Number of retries attempted
  - `failed_at`: Timestamp of final failure
- DLQ retention is 14 days to allow manual investigation
- Ops team should monitor DLQ depth and set up alerts for non-empty DLQs
- DLQ messages can be replayed to the original topic after root cause resolution

## 8. Ordering & Partitioning

### 8.1 Partition Key Strategy

| Topic                | Partition Key     | Ordering Guarantee             | Rationale                                                  |
| -------------------- | ----------------- | ------------------------------ | ---------------------------------------------------------- |
| `notification.email` | `recipient_email` | Per-recipient email ordering   | Prevents duplicate/out-of-order codes to same recipient    |
| `notification.sms`   | `recipient_phone` | Per-recipient phone ordering   | Prevents duplicate/out-of-order codes to same recipient    |

### 8.2 Ordering Guarantees and Trade-offs

- **Within a partition:** Messages are strictly ordered. Using the recipient as partition key ensures all verification codes for the same user are processed sequentially.
- **Across partitions:** No ordering guarantee. Different recipients' notifications can be processed in parallel.
- **Trade-off:** Using recipient as partition key may cause uneven partition distribution if certain recipients receive significantly more notifications. This is acceptable given the rate limiting on verification code endpoints (3 requests per minute).

## 9. Observability

### 9.1 Metrics

| Metric Name                              | Type      | Labels                          | Description                                            |
| ---------------------------------------- | --------- | ------------------------------- | ------------------------------------------------------ |
| `kafka_producer_messages_total`          | Counter   | `topic`, `event_type`, `status` | Total messages produced (status: success, error)       |
| `kafka_producer_latency_seconds`         | Histogram | `topic`                         | Time to publish message to Kafka                       |
| `kafka_consumer_messages_total`          | Counter   | `topic`, `event_type`, `status` | Total messages consumed (status: success, error, dlq)  |
| `kafka_consumer_processing_seconds`      | Histogram | `topic`, `event_type`           | Time to process a consumed message                     |
| `kafka_consumer_lag`                     | Gauge     | `topic`, `consumer_group`       | Consumer lag (messages behind latest offset)           |
| `kafka_dlq_messages_total`              | Counter   | `topic`                         | Total messages sent to DLQ                             |
| `notification_delivery_total`            | Counter   | `channel`, `purpose`, `status`  | Notification delivery attempts (channel: email/sms)    |
| `notification_delivery_latency_seconds`  | Histogram | `channel`, `purpose`            | External provider response time                        |

### 9.2 Logging Guidelines

| Event                          | Log Level | Fields                                                   |
| ------------------------------ | --------- | -------------------------------------------------------- |
| Message published              | INFO      | `event_id`, `event_type`, `topic`, `partition`, `offset` |
| Message consumed               | INFO      | `event_id`, `event_type`, `topic`, `partition`, `offset` |
| Delivery successful            | INFO      | `event_id`, `channel`, `purpose`, `provider_response_id` |
| Delivery failed (retryable)    | WARN      | `event_id`, `channel`, `purpose`, `error`, `retry_count` |
| Delivery failed (moved to DLQ) | ERROR     | `event_id`, `channel`, `purpose`, `error`, `retry_count` |
| Duplicate event skipped        | DEBUG     | `event_id`, `event_type`                                 |

**Rules:**

- Never log verification codes, recipient email addresses, or phone numbers in plaintext
- Mask recipient identifiers in logs (e.g., `u***@example.com`, `+886****5678`)
- Include `event_id` in all log entries for end-to-end tracing
- Use structured JSON logging format consistent with other services

### 9.3 Alerting Thresholds

| Alert                      | Condition                                  | Severity |
| -------------------------- | ------------------------------------------ | -------- |
| High consumer lag          | Consumer lag > 1000 messages for 5 minutes | Warning  |
| DLQ non-empty              | Any message in DLQ topic                   | Warning  |
| DLQ depth increasing       | DLQ depth increases for 30 minutes         | Critical |
| Delivery failure rate high | > 10% delivery failures in 5-minute window | Critical |
| Producer publish failures  | Any publish failure                        | Warning  |
| Consumer group inactive    | No heartbeat for > 60 seconds              | Critical |
