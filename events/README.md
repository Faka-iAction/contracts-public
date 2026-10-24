# FAKA iAction event contracts

Status: draft for review, revised with API contract `1.0.0-alpha.2` (2026-10-15). The Symfony reference implementation (`api-symfony`) now emits many of the names below through a transactional outbox; where this file says a name is emitted, that is a statement about that code, not a claim that it is tested in production. Payload schema files do not exist yet; the minimal payloads of the names added in alpha.2 are described in prose below.

This folder holds the contracts for domain events. `envelope.schema.json` (JSON Schema 2020-12) defines the envelope that every event uses. Payload schemas, one per event type and version, will be added as each event is defined.

Sources: master architecture v2.10 (Sections 23, 41 Event Infrastructure, 62.9), the v2.11 draft, TIS v0.3 (Sections 8.4, 8.5) and the product decisions in the product decision record (not published). If this file and the architecture disagree, the architecture wins.

## What events are for

Events drive asynchronous work: notifications, search indexing, realtime publication, integrations and audit workflows.

Events are not the source of truth. The shared database, and for money the financial ledger, are authoritative. All backends share one database, so events are never used to copy business data from one backend to another. A consumer that needs the current state reads it; it does not reconstruct it from events.

An event is written to the outbox in the same database transaction as the change it records, and published afterwards. If the transaction rolls back, there is no event.

## Envelope

Every event has these fields. See `envelope.schema.json` for types and limits.

| Field | Meaning |
| --- | --- |
| `eventId` | Unique opaque ID of this event. The deduplication key. |
| `eventType` | Name of the event, for example `MemberRegistered`. |
| `schemaVersion` | Version of the payload schema for this event type. Integer, starts at 1. |
| `aggregateType` | Type of the thing the event is about, for example `Member`. |
| `aggregateId` | Opaque public ID of that thing. |
| `aggregateVersion` | Version of the aggregate after this change. Increases by one per event for the same aggregate. |
| `occurredAt` | When the change happened. UTC, ISO 8601, `Z` suffix. |
| `correlationId` | Correlation ID of the request or job that led to the event. Same value as the API's `X-Correlation-Id`. |
| `causationId` | `eventId` of the event that directly caused this one, or `null`. |
| `producer` | Component that produced the event, for example `api-symfony`. |
| `payload` | Event-specific data. |

The envelope is closed: no other top-level field is allowed. Anything event-specific goes in `payload`.

## Naming rules

- An event type is `<Aggregate><PastTenseVerb>` in PascalCase: `MemberRegistered`, `CreatorProfileUpdated`. This follows the names used in the architecture (`ActionCreated`, `ActionFulfilled`).
- An event states a fact that has already happened and has been committed. It is never a command or a request (`SendReceipt` is not an event).
- The name starts with the `aggregateType`. `CreatorProfileCreated` has `aggregateType` `CreatorProfile`. An event about one part of an aggregate may start with that part's name instead: `BidRaised` and `BidRankChanged` have `aggregateType` `Action`.
- Where the event records a state change, the verb is the **canonical target state** from `../domain/state-machines.md`, so that the event and the state can never disagree: `PaymentConfirmed` (state `CONFIRMED`), `ActionFulfilled` (state `FULFILLED`), `CashReceiptConfirmed` (state `CASH_CONFIRMED_BY_CREATOR`). The dotted names in the state-machine tables (`payment.confirmed`) are indicative; the names here are authoritative.
- Aggregate names: `Action` is a participant Action (one participant's Request, Vote, Bid, Shout-out or Tip). `CreatorAction` is the definition a creator publishes.
- Use the canonical vocabulary of the architecture: Member, Creator, Recipient, Creator Action, Follow. Do not name an event after a vertical; FAKA iAction is the platform and FAKA iRequest is one vertical on it.
- Name the business fact, not the table or the screen. `CreatorHandleChanged`, not `CreatorProfilesRowUpdated`.
- Prefer a specific event to a generic one when consumers react differently. A handle change has its own event because QR and link consumers care about it and nothing else in a profile update.
- A released name is never reused with a different meaning.
- Payload field names are `camelCase`. Identifiers, money and timestamps follow the API conventions (`../openapi/CONVENTIONS.md`): opaque string IDs, `Money` as integer minor units plus ISO 4217 currency, UTC timestamps.

## Versioning rules

- `schemaVersion` belongs to the event type. `MemberRegistered` version 2 and `CreatorFollowed` version 2 are unrelated.
- Compatible payload changes do not change `schemaVersion`: adding an optional field, or adding a value to a value list that the schema marks as open.
- Incompatible payload changes increase `schemaVersion` by one: removing or renaming a field, changing a type or a meaning, making an optional field required.
- Consumers ignore payload fields they do not know.
- A consumer that receives a `schemaVersion` it does not support must not guess. It sets the event aside for an operator (dead letter) and does not acknowledge it as processed.
- When a version is increased, the producer and the consumers must be able to run through a period in which both versions exist, because the four backends and the consumers are not deployed at the same instant. How that period is run (publish both, or upgrade consumers first) is decided per event when it happens.
- The envelope has its own version, in the `$id` of `envelope.schema.json`. Changing the envelope incompatibly is a change for every consumer and needs an architecture decision.
- Each payload schema will live in this folder as `<EventType>.v<schemaVersion>.schema.json`.

## Delivery expectations

**At-least-once.** An event may be delivered more than once. It is never guaranteed to be delivered exactly once.

**Consumers are idempotent.** A consumer records the `eventId` values it has processed and applies each event at most once. Processing an event twice must never duplicate a payment, a fulfilment, a ledger posting, a settlement, a refund or a notification.

**No reliance on wall-clock order.** Events can arrive out of order, and clocks on different machines disagree. A consumer must not order events by `occurredAt` and must not assume that an event with a later timestamp happened later.

- To order the events of one aggregate, use `aggregateVersion`. It is the event's sequence number among its aggregate's events, kept in the outbox (see "Aggregate version as built" below). An event whose `aggregateVersion` is not greater than the last one applied is a duplicate or is stale and is skipped. A gap means an earlier event has not arrived yet.
- There is no ordering between different aggregates. If a consumer needs facts from two aggregates, it reads the current state from the database.
- When in doubt, a consumer re-checks the current state before acting, and applies only a transition that is valid from that state.

**Retries.** A consumer that fails on a transient error retries with bounded exponential backoff and jitter, up to a maximum number of attempts. An event that still fails, or fails permanently, goes to a dead-letter store for an operator to review. It is not dropped.

**External side effects.** A consumer never blindly retries an external side effect whose outcome is unknown, above all a payment-provider call. It queries the provider and reconciles first.

**Replay.** Events may be replayed, for example to rebuild a search index. Replay must be safe: because consumers are idempotent, a replayed event changes nothing that was already applied.

**Privacy.** A payload carries the minimum a consumer needs. It does not carry passwords, tokens, payment credentials or contact details that the consumer does not need. Sensitive data is referenced by ID and read by a consumer that is entitled to it. An event that is published to a public realtime topic uses a separate public projection with an explicit allowlist, not the internal payload.

## Aggregate version as built

Settled 2026-10-15 (open point 2), as `api-symfony` does it. `aggregateVersion` is **not** a version column on the business entity. The outbox assigns it: one more than the highest version already stored, or written in the same transaction, for the same `(aggregateType, aggregateId)`. A unique constraint on `(aggregate_type, aggregate_public_id, aggregate_version)` in `outbox_events` is the guard: two transactions that emit for one aggregate at the same moment cannot both commit, and the loser is rolled back with its change. Blocks that hold the aggregate's row lock while they emit never meet that case.

Consequences:

- Every aggregate that emits has a gap-free sequence starting at 1, without a counter column on its table.
- It is unrelated to the entities' internal optimistic-lock version (`lock_version`) and to `state_version` in the state history. A consumer must not compare them.
- An aggregate with no table of its own (a follow, which has no public ID; a country, identified by its ISO code) still has a sequence.
- All backends must assign it the same way. Spring Boot, ASP.NET Core and FastAPI do not write to the production database in MVP1, so in MVP1 only Symfony assigns it.

## How `api-symfony` publishes

Recorded as a fact about the code, not as a test result. Each event is written to `outbox_events` in the transaction of its change. A relay command claims rows with a lease, publishes them in per-aggregate order, retries with backoff and moves a row to a dead state after the last attempt. Delivery is at least once. Consumers record `(consumer_name, event_id)` in `processed_events` and skip duplicates. An operator can see counts and dead events, and requeue one, from the web admin; each requeue is audited. A refused change emits nothing, because the event is recorded only after every refusal has been decided.

## MVP1 events

Names only. Payload schemas follow.

The identity, profile, follow and QR names below belong to the scope of this skeleton. They are proposals for review.

Members and capabilities:

- `MemberRegistered`
- `MemberProfileUpdated`
- `MemberCapabilityChanged`
- `MemberIdentifierAdded`
- `MemberIdentifierRemoved`
- `MemberSignedIn`
- `MemberSessionRevoked`

Creator profiles:

- `CreatorProfileCreated`
- `CreatorProfileUpdated`
- `CreatorHandleChanged`
- `CreatorProfileSuspended` (emitted by moderation; payload below)
- `CreatorProfileReinstated` (not emitted yet: no reinstatement path exists)

Follows:

- `CreatorFollowed`
- `CreatorUnfollowed`

QR resources:

- `QrResourceCreated`
- `QrResourceActivated`
- `QrResourceSuspended`
- `QrResourceRevoked`
- `QrResourceReplaced` (added in alpha.2; payload below)
- `QrResourceExpired` (not emitted: expiry is computed when a code is resolved and is not a stored transition)
- `QrResourceRetargeted` (not emitted yet: an ordinary replacement does not change the target)

Notes on the names above, following the product decisions of 2026-10-12 (the product decision record (not published)):

- `MemberRegistered` is emitted when a member is created: when a registration challenge is completed with its one-time code, or on the first use of a social login identity. Starting a registration emits nothing, because no member exists yet.
- `MemberIdentifierAdded` is emitted when a sign-in identifier (email, phone or social login identity) becomes verified and attached to a member, including the first one at registration. `MemberIdentifierRemoved` is emitted when one is detached. Their payloads must not carry the email address, phone number or provider subject; they reference the identifier by its opaque ID and state its type.
- Sending a one-time code and a failed verification are not domain events. They belong in the security audit log.
- `CreatorHandleChanged` covers a rename. The old handle stays reserved for the same creator and redirects, so consumers that index or cache by handle keep the old handle pointing at the same creator rather than deleting it.
- A member has at most one creator profile, so `CreatorProfileCreated` occurs at most once per member.

The names below follow the state machines in `../domain/state-machines.md` and the operations added to the API contract on 2026-10-13. A name marked (P) depends on a transition that the state machines mark as a proposal.

Creator Actions (aggregate `CreatorAction`):

- `CreatorActionCreated`
- `CreatorActionConfigChanged` (a new configuration version, decision 19)
- `CreatorActionUpdated` (added in alpha.2 as a proposal; **not emitted by `api-symfony`**: a change to the title, description, start or end time that does not create a configuration version currently emits nothing, so a search index or a cached page would not learn of it. Payload: `creatorActionId`, `changedFields` (names only, from `title`, `description`, `startsAt`, `endsAt`). A change of `endsAt` that reopens an expired Creator Action also emits `CreatorActionActivated`.)
- `CreatorActionPublished`
- `CreatorActionActivated` (also on resume, and on reopening an expired one by moving its end time (P, DN-02))
- `CreatorActionPaused`
- `CreatorActionExpired`
- `CreatorActionDisabled`

Participant Actions (aggregate `Action`):

- `ActionCreated`
- `ActionInitiated` (a payment or cash report was started)
- `ActionSubmitted`
- `ActionAccepted`
- `ActionRejected`
- `ActionFulfilled`
- `ActionCompleted` (P, DN-06; for a Vote, decided by decision 20: emitted for each counted vote when voting closes)
- `ActionCancelled`
- `ActionExpired`
- `ActionFailed` (P, DN-07)
- `BidRanked` (a bid's first payment was confirmed and it entered the ranking)
- `BidRaised`
- `BidRankChanged` (one per bid whose rank moved; drives "your position changed" notifications)

Payments (aggregate `Payment`):

- `PaymentCreated`
- `PaymentPending`
- `PaymentConfirmed`
- `PaymentFailed`
- `PaymentExpired`
- `PaymentRefunded`
- `PaymentDisputed`
- `PaymentDisputeResolved` (P, DN-22: dispute decided for the creator; the payment returns to `CONFIRMED`)
- `PaymentReversed` (P, DN-22)

Refunds (aggregate `Refund`):

- `RefundRequested`
- `RefundFailed` (raises a reconciliation exception; P, DN-23)

Cash in person (aggregate `CashReceipt`):

- `CashReceiptReported`
- `CashReceiptConfirmed`
- `CashReceiptVoided` (P, DN-27)
- `CashReceiptCorrected` (the compensating record was created)

Settlement (aggregate `Settlement`):

- `SettlementEntitlementRecorded`
- `SettlementEligible`
- `SettlementHeld`
- `SettlementHoldReleased` (P, DN-29)
- `SettlementProcessing`
- `SettlementSettled`
- `SettlementFailed`
- `SettlementCancelled`

These per-entitlement names are **not emitted by `api-symfony`** (contract 1.0.0-alpha.3). Under decision 26 a creator's payable entitlement is derived from the ledger when asked and paid out in pooled payouts, one per creator and currency, so there is no per-payment settlement record to change state; the `Payout` events below carry the facts. Whether these names are kept for a later per-entitlement settlement record is open.

Settlement Profile (aggregate `SettlementProfile`):

- `SettlementProfileCreated` (P, DN-34; emitted since alpha.3, when a member first gives payout details)
- `SettlementProfileVerificationStarted` (emitted since alpha.3: details given, waiting for an operator's review)
- `SettlementProfilePayoutEnabled` (emitted since alpha.3: verified by the provider or by an operator)
- `SettlementProfileVerificationFailed` (P, DN-36; emitted since alpha.3, when an operator could not verify the destination)
- `SettlementProfileDestinationChanged` (P, DN-36; emitted since alpha.3, with the payout hold)
- `SettlementProfilePayoutSuspended` (P, DN-36; not emitted yet: no suspension path exists)

Payouts (aggregate `Payout`; added in alpha.3 as proposals, decision 26):

- `PayoutCreated` (a payout was recorded: its amount left the creator's entitlement for "payout in transit")
- `PayoutProcessing` (the provider accepted the transfer)
- `PayoutPaid` (the provider reported the transfer successful)
- `PayoutFailed` (refused or failed at the provider, or returned because the destination changed before it was sent; the amount is back in the entitlement)
- `PayoutReversed` (the provider returned the transfer; the amount is back in the entitlement)

Guests (aggregate `GuestSession`):

- `GuestSessionCreated`
- `GuestSessionConverted` (added in alpha.2; payload below)

Country configuration (aggregate `Country`):

- `CountryConfigurationUpdated` (was a placeholder; given an aggregate and a payload in alpha.2)

Administration and trust and safety (added in alpha.2; emitted by `api-symfony`; payloads below):

- `AdminRoleGranted`, `AdminRoleRevoked` (aggregate `Member`)
- `ReportFiled` (aggregate `Report`)
- `ModerationActionTaken` (aggregate `ModerationCase`)
- `ActionMessageHidden` (aggregate `ActionMessage`)

Still placeholders (referrals):

- `ReferralRewardCreated`
- `ReferralRewardReversed`

### Payloads of the names added or given a payload in alpha.2

Every payload is minimal and privacy-reviewed: public identifiers, codes and states only. None carries a token, a contact detail, a reason typed by an operator or a participant, or the text of a message or report. Those stay in the audit log or the source record. All are `schemaVersion` 1.

| Event | `aggregateType` | `aggregateId` | Payload fields |
| --- | --- | --- | --- |
| `QrResourceReplaced` | `QrResource` | the QR identifier (short code) | `creatorId`, `kind`, `targetType`, `previousState`, `state` (`REPLACED`), `successorQrId` |
| `CountryConfigurationUpdated` | `Country` | the ISO 3166-1 alpha-2 code | `country`, `domain`, `key`, `version`, `mandatoryPolicy` (boolean), `effectiveFrom`, `effectiveUntil` (nullable). The value itself is not carried; a consumer reads it from the configuration. |
| `GuestSessionConverted` | `GuestSession` | the guest session's public ID | `guestSessionId`, `memberId`, `linkedActions` (a count) |
| `AdminRoleGranted` / `AdminRoleRevoked` | `Member` | the member's public ID | `memberId`, `role` (`ADMIN_SUPPORT`, `ADMIN_MODERATOR`, `ADMIN_OPERATIONS`) |
| `ReportFiled` | `Report` | the report's public ID | `reportId`, `state` (`REPORTED`), `category` (the reason code), `kind` (the subject type), `targetId` (the subject's public ID), `caseId` |
| `ModerationActionTaken` | `ModerationCase` | the case's public ID | `caseId`, `decision` (`START_REVIEW`, `DISMISS`, `ESCALATE`, `SUSPEND_PROFILE`, `SUSPEND_QR`, `REVOKE_QR`, `HIDE_MESSAGE`), `kind` (the subject type), `status` (the case's status after the decision), `moderatorId` |
| `CreatorProfileSuspended` | `CreatorProfile` | the profile's public ID | `creatorId`, `previousStatus`, `status` |
| `ActionMessageHidden` | `ActionMessage` | the message's public ID | `actionId`, `creatorActionId`, `hidden` (`true`) |

The other QR events (`QrResourceCreated`, `QrResourceActivated`, `QrResourceSuspended`, `QrResourceRevoked`) carry the same fields as `QrResourceReplaced` without `successorQrId`.

### Payloads of the settlement and payout events (alpha.3)

Proposed by `api-symfony` for decision 26; all `schemaVersion` 1. **No settlement or payout event carries an account number, a bank code, a provider recipient or transfer reference, the account holder's name or an identity document**; a consumer that needs the masked destination reads it from the Settlement Profile.

| Event | `aggregateType` | `aggregateId` | Payload fields |
| --- | --- | --- | --- |
| `SettlementProfileCreated`, `SettlementProfileVerificationStarted`, `SettlementProfilePayoutEnabled`, `SettlementProfileVerificationFailed`, `SettlementProfileDestinationChanged` | `SettlementProfile` | the profile's public ID | `settlementProfileId`, `memberId`, `payoutEligibilityState`, `railType` (nullable), `payoutHoldUntil` (nullable timestamp) |
| `PayoutCreated`, `PayoutProcessing`, `PayoutPaid`, `PayoutFailed`, `PayoutReversed` | `Payout` | the payout's public ID | `payoutId`, `creatorId`, `amount` (Money: what is sent), `status` (the contract's `PayoutStatus`) |

Notes:

- A first submission emits `SettlementProfileCreated` and then `SettlementProfileVerificationStarted` or `SettlementProfilePayoutEnabled`, in one transaction. A change of destination emits `SettlementProfileDestinationChanged` and then one of those two. Giving the same details again emits nothing.
- A payout returned because the destination changed before it was sent emits `PayoutFailed`.
- A payout blocked by the legal gate of decision 26 emits nothing: nothing was recorded or sent. The attempt is in the audit log.

Notes on these names:

- `QrResourceReplaced` follows the naming rule (target state `REPLACED`). `QrResourceRetargeted` remains for a change of destination, which is a different fact.
- The admin role events are on the `Member` aggregate, so a consumer sees them in order with the member's other events. Granting a role a member already holds, or revoking one they do not hold, changes nothing and emits nothing. Roles are granted and revoked by console command only.
- An enforcement decision emits `ModerationActionTaken` and also the event of its effect, in the same transaction: `CreatorProfileSuspended`, `QrResourceSuspended`, `QrResourceRevoked` or `ActionMessageHidden`.
- The names `ReportFiled`, `ModerationActionTaken` and the decision codes are `api-symfony`'s proposal. The architecture names report states (Section 28) but no events.

### Listed but not emitted by `api-symfony` yet

So that consumers do not wait for them: `MemberProfileUpdated`, `MemberCapabilityChanged`, `MemberSignedIn`, `MemberSessionRevoked` (see open point 3), `CreatorProfileReinstated`, `QrResourceExpired`, `QrResourceRetargeted`, `CreatorActionUpdated`, `ActionFailed` (no path reaches `FAILED` without payments), `BidRanked`, `BidRaised`, `BidRankChanged`, and every payment, refund, cash, settlement (per entitlement) and referral event, and `SettlementProfilePayoutSuspended`. Since alpha.3 `api-symfony` emits the other Settlement Profile events and the `Payout` events above.

Notes:

- **`PaymentConfirmed`, not `PaymentCompleted`.** Architecture Section 62.9 uses `PaymentCompleted`, `RefundCompleted` and `SettlementCompleted` as examples. The canonical states are `CONFIRMED`, `REFUNDED` and `SETTLED` (architecture 13.3), so the events are `PaymentConfirmed`, `PaymentRefunded` and `SettlementSettled`. The `...Completed` names are not used for payments, refunds or settlements. `ActionCompleted` is used because `COMPLETED` is an Action state.
- A refund in progress is not a payment state. `RefundRequested` and `RefundFailed` cover the refund record; success is `PaymentRefunded`.
- A cash receipt never causes a payment, settlement or referral event.
- Outbidding produces `BidRankChanged` only. It never produces a refund event.
- `ActionCancelled` caused by `CreatorActionDisabled` carries the disabling event's `eventId` as its `causationId`, and so does each resulting `RefundRequested`.

## Open points

1. **Payment event names.** Settled 2026-10-13 in favour of the canonical state names: `PaymentConfirmed`, `PaymentRefunded`, `SettlementSettled` (see the notes above). The next architecture version should change the examples in Section 62.9 to match.
2. **`aggregateVersion` for every aggregate.** Settled 2026-10-15 as built: the field stays required, and the outbox assigns it per aggregate, so no aggregate needs a version counter on its own table (see "Aggregate version as built").
3. **Sign-in events.** `MemberSignedIn` and `MemberSessionRevoked` are security signals more than domain facts. They may belong in the audit log rather than the event stream.
4. **`producer` detail.** The envelope carries the component name only. Whether it should also carry the component's release version, for tracing a bad event to a build, is open.
5. **Event ID scheme.** Opaque string; the generation scheme follows the identifier decision in the API conventions.
6. **Retention.** How long published events and processed-event records are kept is not specified.
7. **Events marked (P).** Each depends on a transition the state machines mark as a proposal. If the proposal is rejected, the event is removed or renamed.
8. **Payload schema files.** The payloads in this file are prose. A `<EventType>.v1.schema.json` per emitted event is still to be written, and the conformance suite should check emitted payloads against them.
9. **Creator Action detail changes.** Confirm `CreatorActionUpdated`, or decide that consumers re-read the Creator Action on other events only.
