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

Notification declares its broker topology when its consumer starts, so publishing fails until it has run once: exchange `cra.events`; durable queue `notifications.events` with `x-dead-letter-exchange: cra.dead`, bound with routing keys `BuddyRequestSent` and `BuddyRequestAnswered`; durable topic exchange `cra.dead`; and durable queue `notifications.dead`, bound to it with `#`. A message whose routing key differs from its `eventType` counts as invalid. Payload fields outside this contract are ignored and not stored. An event in a category its target runner turned off is acknowledged and not stored.

Discovery writes outbox events atomically with request creation/response and retries publishing every two seconds. Failed or unrouted publishes leave `published_at` null. The local publisher uses RabbitMQ's management publish API, persistent messages and positive routing acknowledgement. Consumers must be provisioned before events can leave the outbox.

## Profile events, version 1

Machine-readable payload contract: [profile-events.schema.json](profile-events.schema.json).

Profile publishes `ProfileUpdated` after every change to a runner's profile: a save, a visibility change, a photo attached or removed. Exchange `cra.events`, type `topic`, durable; Profile declares it on every connect. Routing key equals `eventType`.

```json
{
  "eventId": "UUID",
  "eventType": "ProfileUpdated",
  "version": 1,
  "occurredAt": "ISO-8601 UTC timestamp",
  "correlationId": "UUID",
  "payload": {
    "id": "UUID",
    "name": "Ray Attajak",
    "area": "Lumphini area",
    "areaCell": { "latitude": 13.73, "longitude": 100.54 },
    "photoUrl": null,
    "bio": null,
    "paceMin": null,
    "paceMax": null,
    "experience": "Intermediate",
    "availability": "Tue & Thu · 18:30",
    "usualDistance": "5–10 km",
    "gear": null,
    "revision": 3
  }
}
```

The payload is the runner's public card plus `revision`. Every key is always present: a field the runner hid or never set is `null`, `gear` included (never `[]`). In the example, Ray hid his pace and has no bio, gear or photo. `id`, `name`, `area`, `areaCell` and `revision` are never null. The exact point and the list of hidden fields never appear: the point is sent only as its cell, rounded to a 0.01° grid. A hidden field is not used for matching either: a runner is left out of searches that filter on a field they hid, and included when that filter is off.

Profile writes the event in the same transaction as the change (a transactional outbox) and relays it every two seconds, oldest first, over AMQP with publisher confirms. Messages are persistent and `mandatory`, with `contentType` `application/json` and `messageId` equal to `eventId`. An event counts as published only once the broker confirms it and did not return it as unroutable. Events therefore wait in the outbox until a queue is bound for `ProfileUpdated`, as they do while the broker is unreachable. Profile connects as its own broker user, `profile`, which may configure and write `cra.events` and nothing else.

Delivery is at least once: deduplicate by `eventId`. Each event is a full snapshot, so a consumer replaces its copy rather than merging. It applies an event whose `payload.revision` is at least the one it holds, and ignores a lower one. `npm run events:republish` in Profile (`node dist/republish-events.js` in its image) sends every profile's current card again under its unchanged revision; run it when a consumer's copy must be built or rebuilt. Revisions restart at 1 when Profile's database is reset or re-seeded, so a consumer's copy must be reset with it.

A consumer's queue, bound to `cra.events` with routing key `ProfileUpdated`, must be durable: RabbitMQ 4.3 refuses a queue that is neither durable nor exclusive, and an exclusive queue disappears with the connection that declared it.

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
