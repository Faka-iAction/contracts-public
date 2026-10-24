# FAKA iAction API conventions

Status: draft for review (milestone M1), contract version `1.0.0-alpha.5`. This file makes no claim about what is implemented or tested.

These conventions apply to every operation in `faka-v1.yaml` and to every operation added later. They are written for the four backend implementations (Symfony first, then Spring Boot, ASP.NET Core and Python/FastAPI) and for API clients. A client must be able to talk to any compliant backend without changing its requests.

Sources: master architecture v2.10 (approved 2026-10-13; Sections 4 to 9, 12.5.3, 13, 14, 22, 23, 28.1, 41, 43, 62.3), TIS v0.2 (Sections 6, 7.1, 8, 9, 10, 11.1), the product decisions of 2026-10-12, 2026-10-13, 2026-10-15 and 2026-10-16 (the product decision record (not published), decisions 1 to 27; decisions 17 to 27 are not yet in the approved architecture), and the state machines and domain model in `../domain/`, which are authoritative for state names and transitions. If this file and the architecture disagree, the architecture wins and this file is wrong. Where the product decisions and an earlier draft disagree, the decisions win.

For shapes (fields, types, status codes), `faka-v1.yaml` is authoritative. This file is authoritative for behaviour that OpenAPI cannot express.

## Changes in 1.0.0-alpha.5

alpha.5 (2026-10-24) adds the creator's profile image. Nothing else changed.

- **Profile image:** `putOwnCreatorImage` (`PUT /me/creator-profile/image`, the image as the request body: `image/jpeg`, `image/png` or `image/webp`, at most 5 MB) and `deleteOwnCreatorImage` (`DELETE /me/creator-profile/image`). The server redraws the image as a JPEG of at most 1024 pixels a side and serves it from its own host; `imageUrl` changes whenever the image does. Where the file is stored (local disk, Amazon S3 or Google Cloud Storage) is a deployment setting, not part of the contract.

## Changes in 1.0.0-alpha.4

alpha.4 (2026-10-24) adds the operations the Flutter app needs to match the web on reporting, QR codes and guest history (items 87, 89 and 90). Nothing else changed. The changes:

- **Reporting (item 87):** `fileReport` (`POST /reports`), by a member or a guest, about a public creator profile or Creator Action. Schemas `ReportRequest`, `ReportReason` (the reason codes `api-symfony` proposed) and `FiledReport`; new tag Trust and Safety.
- **Guest history (item 90):** `convertGuestHistory` (`POST /me/guest-history`): the signed-in member sends the guest session's token in `X-Guest-Token`, and the session's Actions become theirs. Schema `GuestHistoryResult`.
- **The creator's QR code (item 89):** `getOwnQrCode` and `issueOwnQrCode` (`GET` and `POST /me/creator-profile/qr`), `withdrawOwnQrCode` (`POST /me/creator-profile/qr/withdraw`), `getOwnQrImage` (`GET /me/creator-profile/qr/image`, SVG or PNG) and `getOwnQrSheet` (`GET /me/creator-profile/qr/sheet`, an A4 PDF). Schemas `CreatorQrCode` and `QrWithdrawRequest`. A creator cannot replace a code that FAKA iAction suspended. Listing every code, suspending and reinstating one, and replacement that keeps the old code working are still web-admin only.
## Changes in 1.0.0-alpha.3

alpha.3 (2026-10-16) applies product decisions 26 and 27 to the settlement, payout and money sections only. Nothing else changed. The changes:

- **Payout details are given to this API**, with `submitSettlementBankAccount` (`PUT /me/settlement-profile/bank-account`). The provider-hosted onboarding operations and schemas are removed: `startSettlementOnboarding`, `StartSettlementOnboardingRequest`, `SettlementOnboarding` and `SettlementOnboardingStep`. Sections 13.2 and 13.3 are rewritten to match.
- **Payout methods (decision 27):** a South African bank account. `SettlementProfile.railType` reserves `MOBILE_MONEY`.
- **What is kept about a destination:** the provider's recipient reference, the bank's name, the last four digits of the account number and the holder's name. `SettlementProfile` gains `bankAccount`, `verificationMethod` and `payoutHoldUntil`. The full account number, the bank code and any identity document number are never stored, returned, logged, evented or audited.
- **Recent sign-in and payout hold** for giving or changing payout details (Section 13.4). New error code `RECENT_SIGN_IN_REQUIRED` (`403`).
- **Verification:** with the owner's identity or registration number the provider verifies the account at once; otherwise an operator reviews it. New error codes `SETTLEMENT_DETAILS_REJECTED` and `SETTLEMENT_DETAILS_NOT_VERIFIED` (`422`).
- **Pending balance:** the figures are defined for payouts (Section 13.4), and `CurrencyBalance` gains `inCoolOff`, the part of `notYetEligible` still in the cool-off.
- **Payouts:** a new read-only operation, `listOwnPayouts` (`GET /me/payouts`), with the schemas `Payout`, `PayoutStatus` and `PayoutPage`. `LedgerEntry` gains `payoutId`; `LedgerEntryType` gains `PAYOUT_FEE`. There is still no operation that requests, withdraws or moves money.

## Changes in 1.0.0-alpha.2

alpha.2 (2026-10-15) settles points where alpha.1 was silent, in the way the Symfony reference implementation (`api-symfony`) already behaves, and applies product decisions 17 to 25. Where the implementation and a decision disagreed, the decision was followed; no such case was found. The changes:

- **Decisions 17 to 25 applied** and the matching items in Section 15 marked decided: published and active are different states (item 49); disabling cancels free Actions too (item 51); a configuration change makes a new version for new participants (item 52); a vote completes when voting closes (item 58); tallies are shown once voting closes (item 60); a guest session lasts 30 days and is renewed on use (item 62); Shout-out screening before submission and creator review after (item 64); free Actions have a fulfilment deadline (item 55); Paystack is the MVP1 provider (item 67).
- **Vote tallies.** Two new operations, `listPublicVoteTally` (public, after voting closes) and `listOwnVoteTally` (the owning creator, at any time), and a new error code, `VOTE_TALLY_NOT_AVAILABLE`.
- **Rejection reason.** `ParticipantAction.rejectionReason` carries the creator's free-text reason to the participant. It is optional in alpha.2 (Section 15, item 86).
- **Paid submissions without a provider.** `submitRequest`, `submitBid`, `submitShoutOut` and `submitTip` list `503 PAYMENT_PROVIDER_UNAVAILABLE`.
- **Inactive countries.** `becomeCreator` and `updateOwnCreatorProfile` state the `422` field codes for `country` (`UNKNOWN_VALUE`, `NOT_AVAILABLE`); `createCreatorAction` states `409 CONFLICT` for a creator whose country is not active (Section 15, item 88); participation in such a Creator Action is declined with `403 PARTICIPATION_DECLINED`.
- **QR resolution.** A replaced code resolves as its successor; there is no `replaced` public status (Section 15, item 89).
- **Concurrent changes** (item 36): defined in Section 9.1.
- **Malformed optional `Idempotency-Key`**: `400 INVALID_PARAMETER` (Section 9).
- **Which error wins** when a request has several problems: Section 8.2.
- **Identifier formats.** `SignInIdentifier` and `EmailIdentifier` now state the same email rule (the HTML "valid e-mail address" syntax), and `SignInIdentifier` states the E.164 rule (`^\+[1-9][0-9]{6,14}$`), both as `api-symfony` validates them. They are stated in prose: a type-dependent schema condition made the Redocly example check report false warnings.
- **Idempotent replay is byte for byte** (Section 9).
- **Administration is web-only** and reporting, QR management and guest conversion have no API operation yet (Section 14).
- A duplicated line in Section 14 is removed.

## 1. Versioning and compatibility

- The base path is `/api/v1`. The major version is in the path. An incompatible change goes to `/api/v2`; it is never made silently inside `/api/v1`.
- `info.version` in the OpenAPI file is the contract version (semantic versioning). Contract releases are tagged in this repository.
- The host name is environment configuration and is not part of the contract.

Compatible changes, which may be made within `/api/v1`:

- adding an operation;
- adding an optional request field, query parameter or header;
- adding a field to a response;
- adding a value to an open value list (see below);
- adding an error code.

Incompatible changes, which need a new major version:

- removing or renaming an operation, field, parameter or error code;
- changing a field's type, format or meaning;
- making an optional request field required;
- tightening validation so that previously valid requests fail;
- changing the status code or error code returned for an existing situation.

What clients must tolerate:

- unknown fields in responses (ignore them);
- unknown values in value lists that the contract marks as open ("further values may be added"), using the fallback the field describes;
- unknown error codes (fall back to the HTTP status).

What servers must do:

- reject unknown fields in request bodies with `422` and field code `UNKNOWN_FIELD`. Request schemas are closed (`additionalProperties: false`) so that a typo is not silently ignored.
- return only listed fields in public projections. Schemas marked as an allowlist (`PublicCreatorSummary`, `PublicCreatorProfile`, `QrResolution`, `QrDestination`, `PublicCreatorAction`, `PublicActionOption`, `VoteTallyEntry`) are closed. Adding a field to one of them is a contract change that needs a privacy review, even though it is technically compatible for clients.

The deprecation policy (notice period, how deprecation is signalled) is not yet defined. See Decisions needed.

## 2. Requests and responses

- Bodies are JSON, UTF-8. Requests send `Content-Type: application/json`.
- Successful responses use `application/json`. Error responses use `application/problem+json`.
- Field names are `camelCase`. Path segments are `kebab-case`. Error codes and capability codes are `UPPER_SNAKE_CASE`.
- Value lists: the values introduced with Creator Actions, participation, payments, cash and the money views are `UPPER_SNAKE_CASE`, because they are the canonical names of the architecture and the domain model (`SUBMITTED`, `CONFIRMED`, `CASH_CONFIRMED_BY_CREATOR`, `PAYOUT_ENABLED`, `BID_RAISE`). The identity, profile and QR values that came first (`email`, `google`, `unverified`, `active`) are `lower_snake_case` and stay so, because changing them would break implemented clients.
- A field that has no value is sent as `null`, not omitted, when the schema lists it as required. Clients may rely on required fields being present.
- **`null` is not the same as absent.** A field the schema allows to be `null` may be sent as `null`; in a `PATCH` that clears it. A field the schema does not allow to be `null` (an optional field such as `displayName` in `updateOwnMember`, for example) is rejected when sent as `null`: `422 VALIDATION_FAILED` with field code `INVALID_TYPE`. A required field sent as `null` gets field code `REQUIRED`. The server never treats `null` as "leave unchanged".
- `PATCH` is a partial update: an absent field is left unchanged; a field sent as `null` is cleared, where the schema allows `null`. Arrays are replaced as a whole. A `PATCH` with no fields is rejected with `422 VALIDATION_FAILED` and one field error `{"field": "$", "code": "EMPTY_UPDATE"}`.
- A field of the wrong JSON type inside a valid JSON object (a number where a string is expected, an object where an array is expected) is rejected with `422 VALIDATION_FAILED` and field code `INVALID_TYPE`. A body that is not valid JSON, or is valid JSON but not an object, is `400 BAD_REQUEST`.
- `/me` always means the authenticated member. A member's private record is never addressable by ID.
- All input is validated on the server. Client-side validation is a convenience only.

## 3. Identifiers

- Every public identifier is an opaque string (`Id` schema: 1 to 64 characters from `A-Z a-z 0-9 _ -`).
- Clients store and compare identifiers as strings. They must not parse them, sort by them, or infer creation time, type or order from them.
- A sequential database key is never exposed as a public identifier.
- Identifiers are stable for the life of the resource. A creator's `id` does not change when the handle is renamed, which is why follows use the `id` and not the handle.
- A handle is a public, human-chosen name, not an identifier. It can change.
- A sign-in identifier (an email address, a phone number or a social login identity) is how a member proves who they are. It is private, it can be added and removed, and it is never used to address a resource. Each one has its own opaque `id`.
- A QR identifier is the short opaque string encoded in a printed code (`/q/{id}`). It is a separate identifier space from `Id` and is case-sensitive.

### Renamed handles

Decided 2026-10-12: after a rename the old handle stays reserved for the same creator and redirects to the new one. It is never released for another creator to claim.

How the API expresses the redirect: `getPublicCreatorByHandle` called with an old handle returns **`200` with the creator's current profile**, not an HTTP redirect.

- `handle` and `canonicalPath` carry the current handle.
- `redirectedFrom` carries the old handle that was requested. It is `null` when the requested handle is the current one.
- A client that sees a non-null `redirectedFrom` replaces any stored handle, link or cache key with the current handle.
- An old handle resolves straight to the current handle, however many renames lie between them. There are no chains for a client to follow.
- If the creator's profile is not publicly visible, an old handle returns the same `404 CREATOR_NOT_FOUND` as the current one.
- A handle that is reserved in this way is unavailable to everyone else: `becomeCreator` and `updateOwnCreatorProfile` return `409 HANDLE_TAKEN` for it, with no indication of whether it is a current or a former handle.

A `200` with a field was chosen over a `308` with a `Location` header for three reasons. HTTP clients differ in whether they follow redirects and whether they keep headers when they do, so four backends and several clients would be harder to keep consistent. A permanent redirect can be cached by intermediaries, which would be wrong if the creator later renames back. And the conformance tests can assert on one response shape. The server-rendered web route `/@oldhandle` is a different interface and does use an HTTP redirect; that is outside this contract.

The examples in the contract look like ULIDs. That is illustrative; the generation scheme is not decided. See Decisions needed.

## 4. Money

- Money is always the `Money` object: `amountMinor` (integer, minor units of the currency) and `currency` (ISO 4217 alphabetic code, upper case). R25.00 is `{"amountMinor": 2500, "currency": "ZAR"}`.
- Money is never a floating-point number, a decimal number or a formatted string, in requests, responses or events.
- An amount is never sent without its currency, and amounts in different currencies are never added.
- The number of minor units per major unit comes from ISO 4217 for the currency. Clients format amounts for display; the API does not send formatted amounts.
- Whether a particular field may be negative is stated on that field.

- **Amounts that decide what is charged are computed by the server.** A client never supplies an authoritative amount. The only amounts a client sends are the ones the participant chooses: a tip amount, a bid amount and a bid raise. The server validates each against the configured limits (`BELOW_MINIMUM`, `ABOVE_MAXIMUM`, `NOT_IN_INCREMENT`, `CURRENCY_MISMATCH`). When the creator confirms cash, the creator restates the amount and the server checks that it matches (`AMOUNT_MISMATCH`).
- Fee percentages, transaction limits, the cool-off length, the checkout window and the guest-session lifetime are configuration. None of them is a constant in this contract. Where a client must show one, the server returns it in a field.

## 5. Time

- Every timestamp is an ISO 8601 / RFC 3339 instant in UTC with the `Z` suffix, for example `2026-10-12T09:15:30Z`. Fractional seconds are optional, up to six digits.
- The server never sends a local offset. A request that sends one is rejected with field code `INVALID_FORMAT`.
- Timestamp fields end in `At` (`createdAt`, `followedAt`).
- Durations are integers in seconds, with the unit in the description (`expiresIn`).
- Clients convert to local time for display. The server does not send local time.
- Clients must not use timestamps to decide the order of events or state changes. See the events README.

## 6. Pagination

Every operation that returns a collection is paginated. There is no unpaginated collection.

Small, server-bounded arrays inside a single resource are not collections and are not paginated: for example the payment methods for one Action, the payments of one Action, or the currencies of a balance summary. Each has a `maxItems` in the schema.

- Request: `limit` (1 to 100, default 20) and `cursor` (opaque string, omitted for the first page).
- Response envelope:

  ```json
  {
    "items": [],
    "page": { "limit": 20, "hasMore": false, "nextCursor": null }
  }
  ```

- To get the next page, send `page.nextCursor` as `cursor` with the same filters. When `hasMore` is `false`, `nextCursor` is `null`.
- A cursor is opaque. Clients must not build, change or decode one. A cursor is valid only with the filters it was issued for.
- A `limit` outside the range, or an invalid or expired cursor, returns `400` (`INVALID_PARAMETER` or `INVALID_CURSOR`). The server does not silently clamp `limit`.
- No total count is returned.
- The order of a list is fixed by the server and described on the operation. Items added or removed while a client pages may cause an item to be skipped or repeated; clients must tolerate a repeated item.

## 7. Filtering and sorting

- Filters are query parameters named after the field they filter (`category`, `country`, `city`, `language`). Free-text search uses `q`.
- Filters combine with AND. Each filter takes one value.
- Each operation lists the query parameters it accepts. **This applies to every operation, including those that declare no query parameters at all:** a query parameter that the operation does not declare is rejected with `400 INVALID_PARAMETER` and a field error naming the parameter with code `UNKNOWN_FIELD`. A misspelt filter therefore never silently returns unfiltered results. A query parameter of the wrong type or out of range is also `400 INVALID_PARAMETER` (field code `INVALID_TYPE` or `OUT_OF_RANGE`), because the problem is in the URL.
- An invalid filter value (for example an unknown category code) returns `400`, not an empty list.
- There is no client-selectable sort yet. When one is added it will be a `sort` parameter with a per-operation allowlist of values.

## 8. Errors

### 8.1 Envelope

Every error response, from every operation, has this body and the media type `application/problem+json`:

```json
{
  "type": "about:blank",
  "title": "Validation failed",
  "status": 422,
  "code": "VALIDATION_FAILED",
  "detail": "One or more fields are invalid.",
  "correlationId": "01JA8Q2M5T7W9X3B6C4D8E0F1G",
  "errors": [
    { "field": "displayName", "code": "REQUIRED" }
  ]
}
```

| Field | Required | Meaning |
| --- | --- | --- |
| `type` | yes | Problem type URI. `about:blank` for now. |
| `title` | yes | Short summary of the problem class. Fixed per code (see below). Not for branching. |
| `status` | yes | The HTTP status code, repeated. |
| `code` | yes | Stable machine-readable code from the catalogue below. This is what clients branch on. |
| `detail` | no | Human-readable explanation that is safe to show to the user. |
| `correlationId` | yes | The request's correlation ID (Section 11). |
| `errors` | no | Field-level problems: `field`, `code`, optional `detail`. |

Rules:

- Clients branch on `code`, and on `errors[].code` for fields. They never parse `title` or `detail`.
- `title` is fixed by the code: it is the standard HTTP reason phrase of the status (`Bad Request`, `Unauthorized`, `Forbidden`, `Not Found`, `Method Not Allowed`, `Conflict`, `Unprocessable Content`, `Too Many Requests`, `Internal Server Error`, `Service Unavailable`), with exactly two exceptions: `VALIDATION_FAILED` has the title `Validation failed` and `VERIFICATION_FAILED` has the title `Verification failed`. `title` is not localised; `detail` may be.
- An error never contains protected data, a stack trace, SQL, an internal host name, a secret or a payment-provider payload. A field error never echoes the rejected value.
- `errors[].field` uses dot notation for nested fields (`identifier.value`) and brackets for array items (`links[0].url`). For a query or header parameter it is the parameter name. `$` names the request body as a whole, for a problem that belongs to no single field (for example `EMPTY_UPDATE`).
- Errors raised before the application runs (by a proxy or gateway) should use the same envelope where the deployment allows. Clients must still cope with a non-JSON error body.

### 8.2 Status codes

| Status | Used when |
| --- | --- |
| `400` | The request cannot be understood: the body is not valid JSON or not a JSON object, a header is invalid, a required `Idempotency-Key` is missing, a query parameter is invalid or unknown, or a cursor is invalid. |
| `401` | No valid credential was presented, or sign-in failed. |
| `403` | The caller is authenticated but not allowed to do this. |
| `404` | The resource does not exist, or exists but is not visible to the caller. The two are not distinguished. |
| `405` | The path exists but does not support the HTTP method. Sent with an `Allow` header. Applies to every path and is not repeated on each operation. |
| `409` | The request conflicts with current state (duplicate, name taken), or an idempotency key was misused. |
| `422` | The body is a JSON object but a field is invalid: wrong JSON type, `null` where not allowed, missing, out of range, or not allowed with the rest of the request. `errors` lists the fields. |
| `429` | Rate limit exceeded. |
| `500` | Unexpected server failure. |
| `503` | The service or a dependency needed for this operation is temporarily unavailable. |

The line between `400` and `422`: if the body is not a JSON object, or the problem is in the URL or the headers, it is `400`. If the body is a JSON object and any field in it is wrong, including a field of the wrong JSON type, it is `422`.

Business rules that depend on the current state of a resource (an Action that can no longer be cancelled, a slug already taken, a Creator Action that is not open) are `409`, not `422`. Rules about the request itself (an amount below the minimum, a method not offered) are `422` when they are about one field's value, and `409` when they are about the state of the world.

Protected resources return `404` rather than `403` when confirming that the resource exists would itself disclose something.

**Which problem is reported.** A request can have more than one problem. Clients must not depend on which one is reported, with these fixed points:

- On an operation that needs a member's bearer token, a request with no valid credential gets `401` before anything in the URL, the headers or the body is looked at. An unknown or invalid query parameter on such a request therefore gives `401`, not `400`.
- On a participation operation, a request that carries neither a bearer token nor a guest token gets `401` first. A guest token that is unknown or expired is checked after the URL, so a request with both a bad guest token and an unknown query parameter may get `400 INVALID_PARAMETER` rather than `401 GUEST_TOKEN_INVALID`. (This is what `api-symfony` does; a backend may also check the guest token first.)
- Problems in the URL and headers (`400`) are reported before problems in the body (`400` or `422`), and those before any rule that depends on stored state (`404`, `409`).

### 8.3 Error code catalogue

The title of every code follows the rule in Section 8.1. Codes are stable once released. New codes are added over time; a code is never reused with a different meaning.

`IDENTIFIER_ALREADY_REGISTERED`, which the first draft returned from `registerMember`, has been removed before any release. It told any caller whether an identifier was registered.

General:

| Code | Status | Meaning |
| --- | --- | --- |
| `BAD_REQUEST` | 400 | Malformed request (for example, the body is not valid JSON). |
| `INVALID_PARAMETER` | 400 | A query or header parameter is invalid or unknown. `errors` names it. |
| `INVALID_CURSOR` | 400 | The pagination cursor is invalid, expired or was issued for different filters. |
| `UNAUTHENTICATED` | 401 | Access token missing, malformed or revoked. |
| `ACCESS_TOKEN_EXPIRED` | 401 | Access token expired. The client should refresh. |
| `FORBIDDEN` | 403 | Not allowed. |
| `CAPABILITY_REQUIRED` | 403 | The caller lacks a capability this operation needs. |
| `NOT_FOUND` | 404 | Resource not found, where no more specific code applies. |
| `CONFLICT` | 409 | Conflict with current state, where no more specific code applies. |
| `IDEMPOTENCY_KEY_REUSED` | 409 | The idempotency key was already used with a different request. |
| `IDEMPOTENCY_REQUEST_IN_PROGRESS` | 409 | A request with this idempotency key is still being processed. |
| `VALIDATION_FAILED` | 422 | One or more fields are invalid. `errors` lists them. |
| `RATE_LIMITED` | 429 | Too many requests. |
| `INTERNAL_ERROR` | 500 | Unexpected server failure. |
| `SERVICE_UNAVAILABLE` | 503 | Temporarily unavailable. |
| `METHOD_NOT_ALLOWED` | 405 | The path does not support this HTTP method. The `Allow` header lists those it does. |
| `IDEMPOTENCY_KEY_REQUIRED` | 400 | The operation requires an `Idempotency-Key` header and it is missing or malformed. |

`CONFLICT` is also returned by `createCreatorAction` when the creator profile's country is not active, because no more specific code exists yet (Section 15, item 88).

Identity and profiles:

| Code | Status | Meaning |
| --- | --- | --- |
| `INVALID_CREDENTIALS` | 401 | Sign-in failed, by any method: one-time code, password or social login. The same code for every cause. |
| `REFRESH_TOKEN_INVALID` | 401 | The refresh token is unknown, expired, revoked or already used. Sign in again. |
| `VERIFICATION_FAILED` | 422 | A one-time code or provider credential presented by a signed-in member could not be verified. The same code for every cause. |
| `IDENTIFIER_NOT_AVAILABLE` | 409 | The identifier belongs to another member. Returned only after the caller has proved control of the identifier. |
| `LAST_IDENTIFIER` | 409 | The member's last sign-in identifier cannot be removed. |
| `IDENTIFIER_NOT_FOUND` | 404 | The caller has no sign-in identifier with this `id`. |
| `CREATOR_PROFILE_EXISTS` | 409 | The member already has a creator profile. |
| `HANDLE_TAKEN` | 409 | The handle is not available: it is another creator's current handle, or a former handle that stays reserved for them. |
| `CREATOR_PROFILE_NOT_FOUND` | 404 | The caller has no creator profile. |
| `CREATOR_NOT_FOUND` | 404 | No publicly visible creator matches. |
| `QR_NOT_FOUND` | 404 | The QR identifier is unknown or malformed. |
| `SOCIAL_PROVIDER_UNAVAILABLE` | 503 | The social login provider is not configured on this server or cannot be reached. The member can use another way to sign in. |

Creator Actions:

| Code | Status | Meaning |
| --- | --- | --- |
| `CREATOR_ACTION_NOT_FOUND` | 404 | No such Creator Action, or it is not visible to the caller. |
| `CREATOR_ACTION_STATE_CONFLICT` | 409 | The Creator Action cannot make this change from its current state. |
| `CREATOR_ACTION_NOT_PUBLISHABLE` | 409 | The configuration is incomplete or breaks a publish rule. `errors` names the fields. |
| `ACTION_SLUG_TAKEN` | 409 | The creator already has a Creator Action with this slug. |
| `OPTION_NOT_FOUND` | 404 | No such option on this Creator Action. |
| `OPTION_IN_USE` | 409 | The option or playlist item has been chosen by a participant and cannot be deleted. |
| `PLAYLIST_NOT_FOUND` | 404 | No such playlist on the caller's profile. |
| `PLAYLIST_ITEM_NOT_FOUND` | 404 | No such item in the playlist. |
| `PLAYLIST_NAME_TAKEN` | 409 | The caller already has a playlist with this name. |
| `PLAYLIST_IN_USE` | 409 | A Creator Action's current configuration uses the playlist. |
| `VOTE_TALLY_NOT_AVAILABLE` | 409 | Voting on this Vote has not closed, or the Vote was disabled, so its result is not shown (decision 21). Added in alpha.2. |

Participation:

| Code | Status | Meaning |
| --- | --- | --- |
| `GUEST_TOKEN_INVALID` | 401 | The guest token is unknown or expired. |
| `GUEST_NOT_ALLOWED` | 403 | The Creator Action does not allow guests. |
| `PARTICIPATION_DECLINED` | 403 | A risk or policy control declined the participation or payment, including participation in a Creator Action whose creator's country is not active. No reason is given. |
| `TRANSACTION_LIMIT_EXCEEDED` | 403 | The amount is over a transaction limit that applies to this participant, creator or Action. The limit itself is not disclosed. |
| `CREATOR_ACTION_NOT_OPEN` | 409 | The Creator Action is not open for participation (not `ACTIVE`, outside its window, or the creator's profile is not public). |
| `ACTION_CONFIG_CHANGED` | 409 | The `configVersion` the participant saw is no longer current. |
| `PARTICIPATION_LIMIT_REACHED` | 409 | A vote limit, quantity limit, duplicate rule or maximum number of bids is reached. |
| `OPTION_UNAVAILABLE` | 409 | The chosen option or playlist item is no longer available. |
| `ACTION_NOT_FOUND` | 404 | No such participant Action for this caller. |
| `ACTION_STATE_CONFLICT` | 409 | The participant Action cannot make this change from its current state. |
| `ACTION_NOT_CANCELLABLE` | 409 | The Action can no longer be cancelled: payment or cash is confirmed, or it is a Vote. |
| `BID_NOT_RAISABLE` | 409 | The bid is not ranked, or the Creator Action is not open. |

Payments and cash:

| Code | Status | Meaning |
| --- | --- | --- |
| `ACTION_NOT_PAYABLE` | 409 | The Action needs no payment now: it is free, already paid, or closed. |
| `PAYMENT_ALREADY_IN_PROGRESS` | 409 | Another payment for the Action is `CREATED` or `PENDING`, or its outcome is not yet known. |
| `PAYMENT_METHOD_NOT_AVAILABLE` | 409 | The method is not offered for this Action now. |
| `PAYMENT_NOT_FOUND` | 404 | No such payment for this caller. |
| `RECEIPT_NOT_AVAILABLE` | 409 | The payment has never been confirmed, so it has no receipt. |
| `PAYMENT_PROVIDER_UNAVAILABLE` | 503 | The payment provider is not configured or cannot be reached. Nothing was charged. |
| `CASH_NOT_AVAILABLE` | 409 | Cash in person cannot be used for this Action (a Bid, a free Action, cash not enabled, or a profile that is not self-managed). |
| `CASH_RECEIPT_NOT_FOUND` | 404 | No such cash receipt on the caller's profile. |
| `CASH_RECEIPT_STATE_CONFLICT` | 409 | The cash receipt cannot make this change from its current state, or a pending cash report blocks an online payment. |
| `SETTLEMENT_ONBOARDING_NOT_AVAILABLE` | 409 | Payout details cannot be given for this member now (no payout method is offered for the member's country, or the profile is suspended). Kept under its alpha.2 name; returned by `submitSettlementBankAccount` since alpha.3. |
| `RECENT_SIGN_IN_REQUIRED` | 403 | Giving or changing payout details needs a recent sign-in, and the access token's sign-in is too old. The member signs in again and repeats the request. Added in alpha.3. |
| `SETTLEMENT_DETAILS_REJECTED` | 422 | The payment provider did not accept the payout details (for example an unknown bank). Nothing was kept. Added in alpha.3. |
| `SETTLEMENT_DETAILS_NOT_VERIFIED` | 422 | The payment provider could not verify the bank account against the owner's document. Nothing was kept. Added in alpha.3. |

Field codes (in `errors[].code`):

| Code | Meaning |
| --- | --- |
| `REQUIRED` | A required field is missing or `null`. |
| `INVALID_FORMAT` | The value does not match the required format or pattern. |
| `INVALID_TYPE` | The value has the wrong JSON type, or is `null` where `null` is not allowed. |
| `TOO_SHORT` | Shorter than the minimum length, or fewer items than the minimum. |
| `TOO_LONG` | Longer than the maximum length, or more items than the maximum. |
| `OUT_OF_RANGE` | A number outside the permitted range. |
| `UNKNOWN_VALUE` | Not one of the permitted values (for example, an unknown category code). |
| `UNKNOWN_FIELD` | The field or parameter is not part of the contract. |
| `NOT_AVAILABLE` | The value is valid but cannot be used (for example, a handle that is taken). |
| `EMPTY_UPDATE` | A partial update contained no fields. Always with `field` `$`. |
| `NOT_ALLOWED` | The field is valid in general but not in combination with the rest of this request (for example, a password with a phone identifier, `minimumBid` on a Request, or a message that fails screening). |
| `BELOW_MINIMUM` | An amount below the configured minimum (minimum bid, bid increment for a raise, minimum tip). |
| `ABOVE_MAXIMUM` | An amount above the configured maximum (maximum tip). |
| `NOT_IN_INCREMENT` | A bid or raise that is not a whole number of increments. |
| `CURRENCY_MISMATCH` | An amount in a different currency from the Creator Action's. |
| `AMOUNT_MISMATCH` | An amount that must equal a recorded amount does not (confirming cash). |

## 9. Idempotency

Purpose: a client that does not know whether a state-changing request succeeded (timeout, dropped connection) can send it again without it being done twice.

- The client sends an `Idempotency-Key` header: 16 to 128 characters from `A-Z a-z 0-9 _ - : .`. A random UUID is recommended. The client generates a new key for each new intended action and reuses the key only when retrying that same action.
- A key is scoped to the caller and the operation. The caller is the member, or for a guest the guest session. The same key used by two different callers, or on two different operations, is two unrelated keys.
- On an operation that does not accept the header, an `Idempotency-Key` is **ignored**: it is never an error, nothing is stored and it has no effect.
- On an operation that requires it, a request without it, or with a malformed one, is refused with `400 IDEMPOTENCY_KEY_REQUIRED` and nothing is done.
- On an operation where it is optional, a malformed key (outside the length or the character set above) is refused with `400 INVALID_PARAMETER`, with a field error naming `Idempotency-Key` and the code `TOO_SHORT`, `TOO_LONG` or `INVALID_FORMAT`. Nothing is done. A malformed key is never silently ignored where the header is accepted, because the client would then believe its retry was protected.
- **Same key, same request:** the server returns the original result again, with the same status code and the same body, and performs nothing a second time. This holds whether the original result was a success or a client error (`4xx`). The replayed body is byte for byte the original: a server stores it as exact text, not in a form that may reorder JSON object keys (MySQL's `JSON` type does reorder them).
- **Same key, different request:** the server returns `409` with `IDEMPOTENCY_KEY_REUSED` and performs nothing. "Different" means the method, the path or the request body differs in substance; differences in JSON whitespace or key order are not differences.
- **Same key, first request still running:** the server returns `409` with `IDEMPOTENCY_REQUEST_IN_PROGRESS`. The client retries after a short wait.
- A request that failed with `5xx`, or was rejected with `429`, did not record a result. The key may be retried and the request is processed again.
- Keys are retained for a defined period. After it, a key is forgotten and a retry is treated as a new request. The period is not yet decided; see Decisions needed.
- The stored result must be recorded in the same database transaction as the state change it describes, so that a crash cannot leave one without the other.

Each operation states whether the header is optional, required or not accepted:

| Kind of operation | `Idempotency-Key` |
| --- | --- |
| `GET`, `PUT`, `DELETE` | Not accepted. These are idempotent by definition; the header is ignored. The exception is `submitSettlementBankAccount` (a `PUT`), below. |
| `POST` that creates something with natural uniqueness (`becomeCreator`, `createCreatorAction`) | Optional. Without it, a retry returns the natural `409`. With it, a retry returns the original `201`. |
| `POST` that sends a one-time code (`registerMember`, `startSignIn`, `addOwnIdentifier`) | Not accepted. A repeat is a deliberate "send me a new code": it issues a new challenge and invalidates the earlier one. Rate limits, not idempotency keys, protect these. |
| `POST` that consumes a one-time proof (`signIn`, `signInWithSocial`, `confirmOwnIdentifier`) | Not accepted. The code or provider credential is single-use; a retry after a lost response fails and the client starts again. |
| `POST` on tokens (`refreshToken`, `signOut`) | Not accepted. See below. |
| `PATCH` | Not accepted. Re-sending the same partial update gives the same state. |
| Participation and money: `submitRequest`, `submitVote`, `submitBid`, `submitShoutOut`, `submitTip`, `raiseBid`, `createPayment`, `reportCashPayment`, `confirmCashReceipt`, `correctCashReceipt`, `submitSettlementBankAccount` | **Required.** `submitSettlementBankAccount` registers the account with the payment provider, which a blind retry would repeat. Its stored request fingerprint leaves out the account number and document number, so the key store holds nothing they could be recovered from. |
| State transitions that are idempotent by state: `publishCreatorAction`, `pauseCreatorAction`, `resumeCreatorAction`, `disableCreatorAction`, `cancelAction`, `voidCashReceipt`, `acceptAction`, `rejectAction`, `fulfilAction` | Not accepted. Repeating the call when the target state is already reached returns `200` with the current resource and changes nothing; from any other state it returns `409`. |
| Other `POST` that creates a sub-resource (`addCreatorActionOption`, `createPlaylist`, `addPlaylistItem`, `createGuestSession`) | Not accepted. A retry may create a second one; the client checks the list before retrying. |

Token operations are a special case. `refreshToken` consumes the refresh token, so a retry after a lost response presents a token that has already been used. How that is handled (a short grace window, or revoking the session) is a security decision that has not been taken; see Decisions needed.

Idempotency keys do not replace concurrency control; see Section 9.1.

### 9.1 Concurrent changes

Resolved in alpha.2 (item 36), as `api-symfony` behaves. Resources carry no version that a client sends back: there is no `If-Match`, no `ETag` and no `version` field. The server serialises changes to one resource with row locks and keeps an internal version (`lock_version`) on Creator Actions and participant Actions.

- **Creator profile updates** (`updateOwnCreatorProfile`): two updates at once are applied one after the other under the profile's row lock. Each changes only the fields it names; on a field both name, the later one wins. Handle history, languages and links are never written twice. `updateOwnMember` is last write wins in the same way.
- **Creator Action updates** (`updateOwnCreatorAction`): if another change to the same Creator Action is committed after the server read it and before this update writes, the update is refused with `409 CREATOR_ACTION_STATE_CONFLICT` and nothing is written. The client reloads and tries again.
- **State changes** (publish, pause, resume, disable, accept, reject, fulfil, cancel): the transition is decided on the locked, current state. A request that lost a race either finds the target state already reached (`200`, nothing changed) or gets the operation's own state-conflict `409` (`CREATOR_ACTION_STATE_CONFLICT`, `ACTION_STATE_CONFLICT`, `ACTION_NOT_CANCELLABLE`).
- **Limits** (vote limits, quantity limits, duplicate rules): checked with locking reads, so two submissions cannot both pass a limit of one; the second gets `409 PARTICIPATION_LIMIT_REACHED`.
- **Uniqueness** (a handle, a slug, a second creator profile): the database's unique constraints settle a race; the request that loses gets the same `409` as one that arrives second.

There is no separate "concurrent modification" error code. If one is ever wanted, it is a new code, and changing an existing response to it is an incompatible change.

## 10. Rate limiting

- Every operation is rate limited. Limits are stricter on registration, requesting a one-time code, sign-in, token refresh, QR resolution, guest-session creation and every participation and payment operation, because those are where credential guessing, code guessing, message flooding and identifier scanning happen.
- Operations that send a one-time code are limited per identifier as well as per caller, so that one person cannot be flooded with messages. A request that is over the per-identifier limit must still be indistinguishable, to the caller, between a registered and an unregistered identifier.
- A request over the limit returns `429` with `RATE_LIMITED` and a `Retry-After` header giving a number of seconds. The client waits at least that long, and adds its own jitter before retrying.
- A `429` has no side effect and records no idempotency result.
- `503` may also carry `Retry-After`.
- The limits themselves, what they are keyed on (member, IP address, identifier), and whether remaining-quota headers are exposed are not part of this contract yet. See Decisions needed.

## 11. Correlation IDs

- Every response, success or error, carries an `X-Correlation-Id` header. An error also carries the same value in `correlationId`.
- A client may send its own `X-Correlation-Id` (8 to 128 characters from `A-Z a-z 0-9 _ - : .`). If it is present and well-formed the server uses it. If it is absent or malformed the server generates one; a malformed value is ignored, not rejected.
- The server writes the correlation ID to its logs and copies it into every domain event the request causes (the `correlationId` of the event envelope), so that a request can be traced through asynchronous work.
- A correlation ID is not a secret and not a credential. It must not contain personal data.
- Support and error reports quote the correlation ID.

## 12. Authentication and authorization

### 12.1 Tokens

- The API uses bearer tokens: `Authorization: Bearer <access token>`.
- A successful sign-in, by any method, issues a short-lived access token and a refresh token. `refreshToken` exchanges a refresh token for a new pair and invalidates the one presented (rotation). `signOut` revokes the session the access token belongs to.
- Tokens are opaque to clients. A client must not decode a token or rely on its format. The client learns lifetimes from `expiresIn` and `refreshExpiresIn`, and must not hard-code them.
- Responses that carry tokens, one-time code challenges, sign-in identifiers, guest tokens, payment details or money figures carry a `Cache-Control` header that **contains the `no-store` directive**. Other directives may accompany it (`no-store, private` is fine); clients and tests check for the directive, not for an exact header string. Tokens, codes and provider credentials never appear in URLs and are never logged.
- Web sessions are a separate mechanism used by the server-rendered web experience. They are not part of this contract. A session cookie is not accepted as an API credential and an API token cannot be turned into a web session.

### 12.2 Ways to sign in

Decided 2026-10-12 and 2026-10-13 (decisions 10, 13 and 14): a member can register and sign in with an email address, with a phone number, or through a social login provider; the providers are Google and Apple; email sign-in offers a password and a one-time code; phone sign-in uses a one-time code; a social-only member has no password. The verification rules and account linking are still open (items 43 and 44).

| Method | Operations | Status |
| --- | --- | --- |
| One-time code to an email address or phone number | `registerMember` or `startSignIn`, then `signIn` with `grantType: verification_code` | Decided. The verification rules (code length, lifetime, attempts, resend) are still open. |
| Email and password | `signIn` with `grantType: password` | Decided (decision 14). The password policy is still open. |
| Social login (Google, Apple) | `signInWithSocial` | Decided (decision 13). Account linking is still open. |

One-time code flow:

1. The client calls `registerMember` (new member, with profile data) or `startSignIn` (existing member). The server sends a code to the identifier and returns a challenge: `verificationId`, `expiresIn`, `resendAfter`.
2. The client calls `signIn` with the `verificationId` and the code. The server returns tokens, the member and `memberCreated`.

A member is created only when a registration challenge is completed, or on the first use of a social identity. `startSignIn` never creates a member, and nothing else in the platform creates one as a side effect.

Social login: the client obtains a credential from the provider (an ID token or an authorization code) and sends it with the provider name to `signInWithSocial`. The server validates the credential with the provider and never trusts the client's own claim about the user. If the provider identity is already attached to a member, that member is signed in. Otherwise a member is created. `provider` is an enum of `google` and `apple` (decision 13). If neither the provider nor the request supplies a display name, sign-in still succeeds and the member is created with `displayName` `null`; the client asks for a name afterwards and sets it with `updateOwnMember`. A provider that is not configured on the server, or cannot be reached, gives `503 SOCIAL_PROVIDER_UNAVAILABLE`.

Passwords: a member has a password only if they set one. A member who registered with a phone number or through social login has none, and `Member.hasPassword` is `false`. Password sign-in is for email identifiers only: the `password` grant takes an `EmailIdentifier`, and a phone identifier in it is rejected with `422`, field `identifier.type`, code `NOT_ALLOWED`.

### 12.3 Account enumeration

The API must never tell a caller whether an email address or phone number is registered, unless the caller has proved control of it.

- `registerMember` and `startSignIn` return the same `202` challenge whether or not the identifier is registered. Status, headers, body shape, the range of values in the body, and response time must not differ. The work that does differ (sending a message, creating a pending record) is done so that it cannot be timed from outside.
- When `registerMember` is called for an identifier that is already registered, no second member is created. The code sent to the identifier signs its owner in to the existing member, and the profile data in the request is discarded. Only someone who can read the code learns this.
- When `startSignIn` is called for an identifier that is not registered, a challenge is still returned. It can never be completed.
- Every sign-in failure returns `401 INVALID_CREDENTIALS` and nothing else: unknown identifier, wrong or expired code, unknown `verificationId`, too many attempts, wrong password, no password set, invalid provider credential. Refresh failure always returns `REFRESH_TOKEN_INVALID`.
- `addOwnIdentifier` for an email or phone returns the same `202` whether or not the identifier belongs to another member. Only `confirmOwnIdentifier`, after the correct code, can return `409 IDENTIFIER_NOT_AVAILABLE`.
- A challenge allows a limited number of attempts and then stops working, whether or not it could ever have succeeded.

### 12.4 Managing sign-in identifiers

A signed-in member can list their identifiers (`listOwnIdentifiers`), add one (`addOwnIdentifier`, then `confirmOwnIdentifier` for email or phone; a social identity is verified by its provider credential in the one call) and remove one (`removeOwnIdentifier`).

- An identifier can be used to sign in only once it is verified. Unverified identifiers are not listed.
- One identifier belongs to at most one member.
- The last remaining identifier cannot be removed: `409 LAST_IDENTIFIER`. A password is not an identifier and does not count.
- The provider's own subject identifier for a social identity is never returned.

### 12.5 Authorization

- Operations marked `security: []` in the contract are public and need no token: `registerMember`, `startSignIn`, `signIn`, `signInWithSocial`, `refreshToken`, `searchCreators`, `getPublicCreatorByHandle`, `resolveQr`, `listPublicCreatorActions`, `getPublicCreatorAction`, `listPublicActionOptions`, `listPublicVoteTally`, `createGuestSession`.
- Participation, payment and participant status operations accept either a member's bearer token or a guest token (Section 12.6). Every other operation needs a member's access token.
- Authorization is decided on the server for every protected operation, from the authenticated member, their capabilities and their relationship to the resource. `getOwnCapabilities` exists to help a client choose what to show; hiding a control is not authorization.
- A member has at most one creator profile **in MVP1** (decisions 12 and 15). Multiple creator profiles are a later capability under organisations; `/me/creator-profile` is singular for MVP1. `becomeCreator` returns `409 CREATOR_PROFILE_EXISTS` for a member who already has one.
- Capability codes are `MEMBER`, `CREATOR` and `CREATOR_VERIFIED`. Every member holds `MEMBER`. The contract does not use `FAN`; the domain model, which currently proposes `FAN` (OQ-04), must follow the contract.
- Creator operations under `/me/creator-actions`, `/me/playlists`, `/me/incoming-actions`, `/me/cash-receipts`, `/me/balance` and `/me/ledger-entries` need the `CREATOR` capability (`403 CAPABILITY_REQUIRED` otherwise) and act only on the caller's own creator profile. Nobody acts on a creator's behalf in MVP1. Cash can be confirmed only by the creator who owns the profile.
- The Settlement Profile belongs to the member, not to the creator profile, because any member may need one to receive a referral reward. It needs no creator capability. `listOwnPayouts` needs the `CREATOR` capability, like the other creator money views.

### 12.6 Guests

A guest takes part without registering. A guest is a limited principal, not a member, and taking part never creates a member account.

1. The client calls `createGuestSession` (public, rate limited) and receives a `guestToken` and the session's `expiresAt`. A guest session lasts 30 days and is renewed on use: every request that presents the token moves its expiry to 30 days after that request (decision 23).
2. The client sends the token in the `X-Guest-Token` header on participation, payment and status operations. It is shown once; the client stores it for as long as the visitor may want to read their Actions.
3. With the token the guest can submit, pay, report cash, cancel before payment, and read their own Actions, payments and receipts by `id`. With it the guest can do nothing else: it is never accepted where a member's bearer token is required.

- The token is opaque, secret, never put in a URL and never logged. A lost token cannot be recovered through the API.
- An unknown or expired token gives `401 GUEST_TOKEN_INVALID`.
- If a request carries both a valid bearer token and a guest token, the bearer token is used and the guest token is ignored.
- Idempotency keys of a guest are scoped to the guest session.
- **Conversion.** A member can attach a guest session's Actions to their account, explicitly, after proving they hold the guest session (architecture 5.2). The original guest reference stays on each record; nothing is copied. This exists in the web experience (`api-symfony`) and emits `GuestSessionConverted`; there is no API operation for it yet (Section 15, item 90).
- A guest's own Action or payment is addressed by its `id`; an `id` that belongs to someone else gives the same `404` as one that does not exist.
- Public operations return public projections only. A public response never contains a member identifier, a sign-in identifier, a contact detail, settlement information or an internal status.

## 13. Actions, payments and money

### 13.1 Three separate states

A participant Action, its payment and its settlement each have their own state machine, linked by reference (`../domain/state-machines.md`). The contract keeps them apart:

- `ParticipantAction.state` is the Action state. An Action is never "refunded" or "disputed".
- `Payment.state` is the payment state: `CREATED`, `PENDING`, `CONFIRMED`, `FAILED`, `EXPIRED`, `REFUNDED`, `DISPUTED`, `REVERSED`.
- Cash has its own two states, `CASH_REPORTED_PENDING_CREATOR_CONFIRMATION` and `CASH_CONFIRMED_BY_CREATOR`, and is never a payment.
- Settlement states are internal to the platform; the creator sees their effect as the figures of the pending balance.

A confirmed payment does not mean the Action was fulfilled. A fulfilled Action does not mean the creator has been paid out.

### 13.2 Rules the contract encodes

- **Published and active.** A `PUBLISHED` Creator Action is visible but not open; an `ACTIVE` one is open for participation (decision 17).
- **Configuration versions.** A change to a Creator Action's configuration creates a new version for new participants only (decision 19). A participant submits the `configVersion` they were shown; a stale version gives `409 ACTION_CONFIG_CHANGED`.
- **Free Actions complete without payment.** A paid Action is `CREATED` until its payment or cash is confirmed. A free Action has a fulfilment deadline under the same rule as a paid one, with no payment effect (decision 24).
- **Votes** complete when voting closes (decision 20). Their tally is shown to participants once voting closes and not before (decision 21); the owning creator sees it at any time.
- **Shout-out messages** are screened automatically before submission (`422`, field code `NOT_ALLOWED`, nothing charged) and reviewed by the creator after; a rejected paid message is refunded in full (decision 22).
- **The server computes what is charged.** See Section 4.
- **A provider redirect is never proof of payment.** Only `getPayment` says whether a payment succeeded. The server learns outcomes from the provider through a signed, provider-specific notification endpoint or a server-to-server query. That endpoint is outside this client contract.
- **Refunds are never requested by a client.** They are automatic (fulfilment deadline passed, creator rejected, Creator Action disabled) and always for the full amount of the payment (decision 8). The contract exposes refund status only.
- **A participant cannot cancel after payment is confirmed** (decision 7). `cancelAction` works only in `CREATED` and `INITIATED`.
- **A vote cannot be changed** (decision 9). There is no operation to change or withdraw one.
- **Bids:** any amount at or above the minimum in increment steps joins the ranking (decision 1); a raise is at least one increment; rank is shown but not enforced (decision 2); no cash for bids (decision 3).
- **Disabling a Creator Action** cancels every unfulfilled Action on it, paid (decision 6) or free (decision 18), and refunds every confirmed online payment of them in full. Pausing or expiry does not.
- **A Tip has no fulfilment step** and no fulfilment deadline.
- **Cash in person** is only for self-managed creators, never for a Bid, always has a platform fee of zero, and counts only when the owning creator confirms it. A confirmed cash receipt is never edited: a mistake is corrected with a compensating record.
- **There is no wallet.** The creator money views are read-only and derived from the ledger. No operation withdraws, transfers or spends a balance. Cash is shown separately and never included in a balance.
- **A Settlement Profile is not needed to take paid Actions.** Nothing is paid out until the profile is `PAYOUT_ENABLED` and the amount is payable (Section 13.4). Payout details are given to this API (`submitSettlementBankAccount`, decision 26); only a masked form of them is ever returned.
- **Payouts are made by the platform, never requested by a client.** `listOwnPayouts` shows them; no operation starts, cancels, speeds up or withdraws one.

### 13.3 Dependence on the payment provider

**Paystack is the payment provider for MVP1** (decision 25). Payouts are made to the creator's verified payout destination, a South African bank account (decisions 26 and 27). Live payouts are switched off by deployment configuration until they are approved; until then a server makes no live transfer and every refused attempt is logged and audited. Sandbox payouts may run. Nothing in this contract changes when live payouts are switched on.

To limit the effect of provider changes, the money operations stay provider-neutral:

- No provider's API, field, state or reference is modelled. Provider states are mapped by the server onto the canonical payment and payout states.
- Online payment method codes are decided by the server (`PaymentMethodOption`) and are not fixed by the contract; only `CASH_IN_PERSON` is.
- Checkout is described by a neutral instruction (`CheckoutInstruction`) whose only type today is "open this URL". Payout details are plain fields of the destination (`SubmitSettlementBankAccountRequest`), not of a provider.
- Provider notifications (webhooks) are a separate, provider-specific, signed endpoint that is not part of this contract.
- A server reconciles its ledger against the provider's records and lists any difference for an operator.

### 13.4 Settlement and payouts

The rules a server follows for decision 26. The figures are those of `getOwnPendingBalance`.

- **Payable.** An amount from an online payment becomes payable (`eligible`) when the mandatory cool-off after the payment's confirmation has elapsed (its length is configuration; architecture 13.5), its Action is fulfilled (a Tip once paid; a Bid at any rank once fulfilled), no refund or dispute is open on the payment, and the member's Settlement Profile is `PAYOUT_ENABLED` with no payout hold. Before that it is `notYetEligible` (cool-off, fulfilment, or no enabled profile) or `held` (a dispute, an open refund, a payout hold).
- **Cash is never payable** and never part of any figure but `cash`.
- **Refunds and chargebacks after a payout** leave the creator's entitlement negative (`owed`). Later collections cover it first; nothing more is paid out while it lasts. No refund or chargeback is taken from the platform's own money by default.
- **Payout runs.** The platform pays each creator's whole payable amount of a currency once it reaches a minimum payout amount (configuration). One payout per creator and currency is in transit at a time. Running a payout run twice pays once.
- **Payout outcome.** A payout is `PENDING` until sent and `PROCESSING` until the provider reports it. `PAID` is final. `FAILED` and `REVERSED` return the amount to the creator's entitlement for a later run. A payout whose outcome is unknown (a timeout) is verified with the provider and never sent again on its own.
- **Payout fee.** The provider's payout fee is borne by the creator unless the server is configured otherwise; it shows as `fee` on the payout and as a `PAYOUT_FEE` ledger entry.
- **Changing payout details** needs a recent sign-in (`403 RECENT_SIGN_IN_REQUIRED`) and starts a payout hold (`payoutHoldUntil`, length configuration). A payout recorded for the old destination and not yet sent goes back to the pending balance. Giving the same details again changes nothing.
- **Verification** is by the provider against the owner's identity or registration number, when the member gives it, or by an operator's review. A destination that an operator cannot verify is dropped and the profile returns to `DESTINATION_PENDING`.
- **No full account number** is ever stored, returned, logged, evented or audited. Only the bank's name, the last four digits and the holder's name are shown.

## 14. Deliberately out of scope so far

This contract covers conventions, identity, profiles, discovery, follows, QR resolution, Creator Actions, participation, payments, cash in person, the creator's queue, the creator's money views and the Settlement Profile. The following are absent on purpose, not by oversight:

- Provider notification (webhook) endpoints. They are provider-specific and signed, and are not part of the client contract.
- Guest receipt contact (email or phone for a guest's receipt), and guest-to-member conversion over the API. Conversion exists in the web experience (Section 12.6).
- Live sessions, live queues and realtime topics. The `AT_SESSION_END` deadline rule depends on live sessions.
- Disputes as a participant-facing flow; a dispute is raised with the payment provider and shows only as payment state `DISPUTED`.
- Paid votes, donations and organisation-managed profiles (later phase).
- Re-enabling a disabled Creator Action, and deleting a Creator Action.
- Referral codes, attribution and rewards.
- Credential management beyond sign-in identifiers: setting, changing and resetting a password, account recovery when every identifier is lost, and step-up authentication other than the recent sign-in that payout details need (Section 13.4).
- Merging two member accounts.
- Session management beyond sign-out: listing sessions, signing out everywhere.
- Consent records and policy acceptance at registration.
- Account closure, data export, deletion and suppression.
- Profile image upload and other media.
- Reference data: listing creator categories, countries, languages and country configuration.
- QR management: creating, activating, suspending, revoking and replacing QR resources, and QR assets. Only resolution is here. `api-symfony` issues one active code when a member becomes a creator and has the full lifecycle internally, with no API operation (Section 15, item 89).
- Creator verification workflow, public statistics on profiles, and whether the viewer follows a creator.
- Notifications and overlays.
- Organisations, teams, agencies and managed profiles (later phase).
- **Administration, by design.** Admin roles, the moderation queue and its decisions, the audit log, outbox operations and country configuration are used through the server-rendered web experience (`/admin`, behind admin roles and a recent re-sign-in) and console commands only. They will not be added to this API; an operation for them would widen the attack surface for no client that needs it.
- Reporting a creator, a Creator Action or a message. `api-symfony` accepts reports from its web pages only (Section 15, item 87).
- Support tickets.
- Health endpoints.

## 15. Decisions needed

The list of points that the architecture and the TIS do not settle is kept in the private repository and is not published: it records questions that are not yet settled. Item numbers mentioned elsewhere in this file refer to that list. Where this file states a rule, the rule stands; where it says a value is configuration or not yet decided, it is.
