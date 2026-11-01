# Running Buddy Contracts

This repository contains contracts for every implemented backend service and the API gateway, including both Identity implementations. Contract files are included here so consumers can use the catalog without following paths into sibling repositories.

| Surface | Contract | Coverage / origin |
|---|---|---|
| Identity (Python fallback) | [identity.openapi.json](identity.openapi.json) | Exported FastAPI snapshot; register, verify OTP, login, refresh, logout |
| Identity (team Node.js / Express) | [team-identity.openapi.json](team-identity.openapi.json) | Derived from mounted routes, controllers and session code; includes email/phone verification and password reset |
| Profile | [profile.openapi.json](profile.openapi.json) | JSON copy of `profile-service`'s `docs/openapi.yaml`; own profile, field visibility, photo uploads, public profiles and a temporary runner listing |
| Run Conditions | [conditions.openapi.json](conditions.openapi.json) | Exported FastAPI snapshot; weather/air-quality conditions and readiness |
| Run Scheduling | [scheduling.openapi.json](scheduling.openapi.json) | Exported FastAPI snapshot; runs, membership and internal attendance confirmation |
| Notification | [notification.openapi.json](notification.openapi.json) | Live export from `notification-service`; list notifications, mark read and category preferences |
| Buddy Discovery (gRPC) | [buddy_discovery.proto](buddy_discovery.proto) | Copy of the service's versioned protobuf; seven RPCs for search, buddies and requests |
| Buddy Discovery (HTTP) | [discovery.openapi.json](discovery.openapi.json) | Derived from Rails routes; health endpoint only |
| Reputation | [reputation.openapi.yaml](reputation.openapi.yaml) | Existing handwritten contract; records, attendance, ratings and history |
| API Gateway / BFF | [gateway.openapi.json](gateway.openapi.json) | Exported FastAPI snapshot; see routing matrix below for forwarded APIs |
| Buddy broker events | [events.md](events.md), [buddy-events.schema.json](buddy-events.schema.json) | Discovery publisher to Notification consumer |
| Profile broker events | [events.md](events.md), [profile-events.schema.json](profile-events.schema.json) | Profile publisher; intended for Buddy Discovery |
| Identity email jobs | [events.md](events.md), [identity-email-job.schema.json](identity-email-job.schema.json) | Team Identity publisher to its email worker |

Trust & Safety has no implemented service or contract in this stack. It remains unimplemented; this catalog does not invent routes for it. The frontend consumes the gateway and is not a separate backend API. PostgreSQL, Redis, RabbitMQ and monitoring tools are infrastructure, not domain service APIs.

## Contract scope and updates

The exported FastAPI files are existing snapshots, not newly verified live exports, except Notification, which is exported from its running service with complete response schemas. A service's actual `/openapi.json` is authoritative for its implemented HTTP routes. Gateway catch-all proxy routes are generic in OpenAPI; downstream contracts define forwarded resource schemas. Some existing responses have unconstrained schemas; use the service behavior as the authority until those schemas are completed. The Reputation contract is handwritten and currently omits some response bodies, health and the Rails PUT update alias. The Profile contract is a JSON copy of `profile-service`'s `docs/openapi.yaml`, which that service serves at `/openapi.yaml`.

The team Identity contract was derived from its mounted Express routes and controller/session behavior. It is a separate implementation contract, not a replacement for the Python fallback contract. The team implementation is selected by the team-identity Compose override. Its additional email, phone and password-reset routes are **not exposed by the current gateway**; calling those through the gateway requires explicit routing changes. Neither implementation exposes a public account-lock operation.

When changing a FastAPI service, export its `/openapi.json` again and review the diff. When changing Discovery, update the protobuf here from the service's versioned definition and regenerate client stubs. Update the handwritten contracts and broker schemas alongside changes to their producers and consumers. These files document implemented interfaces; they do not make unimplemented features available. JSON schemas specify producer payloads; consumers do not necessarily enforce every schema constraint.

## Authentication and shared conventions

All protected HTTP requests carry `Authorization: Bearer <JWT>`. gRPC carries the same value in lowercase `authorization` metadata. Tokens use issuer `cant-run-alone-local`, audience `cant-run-alone-services`, HS256 and a 15-minute expiry. Refresh tokens are HttpOnly, SameSite=Lax cookies scoped to `/api/v1/auth`; rotating/revoked state is stored in Identity. The signing key is local configuration and is never part of this catalog.

Notification and Profile log and return a UUID `X-Correlation-Id` request header, and create one when it is missing or invalid. A gateway should forward it along with `Authorization`.

## Gateway routing

- `POST /api/v1/auth/{register,verify-otp,login,refresh,logout}` -> Identity; request/response schemas in Identity contract.
- `GET /api/v1/runners` -> Discovery `SearchRunners`, whose JSON payload contains public Profile runner views. For now, Discovery answers it by calling Profile's temporary search. Filters: `radiusKm`, `experience`, `pace`, `availability`, `verifiedOnly`. In the local implementation a nonempty pace filter other than `Any` requests overlap with the caller's saved pace range. Trust badges are deferred, so `verifiedOnly=true` returns no approved badges.
- `GET /api/v1/runners/{uuid}`, `GET/PUT /api/v1/me/profile`, `PUT /api/v1/me/profile/visibility`, `POST /api/v1/me/profile/photo-uploads` and `PUT/DELETE /api/v1/me/profile/photo` -> Profile. Photo files go between the browser and object storage directly, not through the gateway.
- `GET/POST /api/v1/buddy-requests` -> Discovery list/send; send body `{ "runnerId": "UUID" }`. Gateway derives sender from token. Optional old frontend `note` is not persisted by the current protobuf contract.
- `PATCH /api/v1/buddy-requests/{uuid}` -> Discovery response; body `{ "decision": "accepted" }` or `rejected`. `DELETE` cancels a sender-owned pending request.
- `GET /api/v1/buddies` -> Discovery; returns array of buddy UUIDs.
- `/api/v1/runs` and run membership operations -> Scheduling.
- `GET /api/v1/conditions` -> Run Conditions.
- `/api/v1/notifications`, `PATCH /api/v1/notifications/{uuid}/read` and `GET/PATCH /api/v1/notifications/preferences` -> Notification.
- `/api/v1/running_records`, `/api/v1/me/history`, `POST /api/v1/runs/{uuid}/attendance`, `POST /api/v1/runs/{uuid}/ratings`, and `GET /api/v1/runners/{uuid}/rating-summary` -> Reputation.
- `GET /api/v1/dashboard` aggregates current profile, cumulative record distance and the next scheduled run, which may be null. The total is cumulative history, not a weekly metric.

Use UUID IDs, ISO-8601 timestamps with timezones, metres/seconds for stored running records and kilometres for run plans. Run form date/time is interpreted as Asia/Bangkok. Ratings are integers 1–5. HTTP statuses: 401 unauthenticated, 403 forbidden, 404 missing, 409 conflicting lifecycle/capacity/duplicate (some Rails validation duplicates return 422), 422 invalid input, 503 unavailable collaborator/provider. Rails record errors preserve the original D3 nested error envelope; newer errors use `code`/`message`. FastAPI body validation uses its native `detail` list. Profile answers every error with `code`, `message` and `correlationId`, and on a 422 adds `fieldErrors` when a field can be named. Consumers should tolerate these documented shapes; a unified error envelope remains future cleanup.

Search locations are area/distance only. Unjoined run pins are rounded; participant IDs/confirmed IDs and exact coordinates are visible only to joined runners. Phone verification is distinct from a moderation-approved badge.
