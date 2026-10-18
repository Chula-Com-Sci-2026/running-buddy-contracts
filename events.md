# Broker contracts

## Buddy notification events, version 1

Machine-readable payload contract: [buddy-events.schema.json](buddy-events.schema.json).

Exchange `cra.events`, type `topic`, durable. Routing key equals `eventType`.

```json
{
  "eventId": "UUID",
  "eventType": "BuddyRequestSent",
  "version": 1,
  "occurredAt": "ISO-8601 UTC timestamp",
  "correlationId": "UUID",
  "payload": {
    "requestId": "UUID",
    "senderId": "UUID",
    "recipientId": "UUID",
    "status": "pending"
  }
}
```

`BuddyRequestAnswered` has the same shape and `status` is `accepted` or `rejected`. Notification targets `recipientId` for sent events and `senderId` for answered events. Delivery is at least once. Notification's unique `(event_id, recipient_id)` constraint deduplicates after retries. Acknowledgement occurs only after database commit. Invalid messages are rejected to exchange `cra.dead` / queue `notifications.dead`; database failures close/reconnect the channel with a three-second retry delay so unacknowledged messages requeue.

Discovery writes outbox events atomically with request creation/response and retries publishing every two seconds. Failed or unrouted publishes leave `published_at` null. The local publisher uses RabbitMQ's management publish API, persistent messages and positive routing acknowledgement. Consumers must be provisioned before events can leave the outbox.

## Team Identity email jobs

Machine-readable payload contract: [identity-email-job.schema.json](identity-email-job.schema.json).

Team Identity sends persistent JSON messages directly to a durable queue using RabbitMQ's default exchange. `NOTIFICATION_QUEUE` selects the queue: the service default is `notification_queue`, while the team-identity integration override sets `identity.email.v1`. Its dedicated email worker consumes the same queue. These jobs do not use the `cra.events` exchange or the buddy event envelope, and the Notification service's buddy consumer does not consume them.

```json
{
  "channelType": "EMAIL",
  "target": "runner@example.com",
  "subject": "OTP for E-mail Address Verification",
  "templateType": "EMAIL_VERIFICATION",
  "otpCode": "123456"
}
```

The implemented email producers use `EMAIL_VERIFICATION` (five-minute OTP expiry) or `PASSWORD_RESET` (15-minute expiry). The code is six digits. There is no event ID, version or deduplication key in this job format. The worker acknowledges after delivery; on parsing or delivery failure it rejects without requeue. The queue declaration does not configure a dead-letter exchange. Publishing uses persistent messages but does not wait for publisher confirms. Do not assume the buddy consumer's deduplication or retry guarantees apply to these jobs.

The worker also has an SMS branch accepting `channelType`, `target` and `message`, but the current phone OTP flow calls the SMS/console adapter directly rather than publishing an SMS job. The email schema describes the jobs actually emitted by the current producers. `EMAIL_DELIVERY=console` logs the email code instead of sending email; the phone adapter is configured separately.
