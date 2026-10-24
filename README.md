# Contracts

The shared API and event contracts (OpenAPI, event schemas) and the language-neutral conformance tests that every backend must pass.

Repository: `Faka-iAction/contracts-public` (FAKA iAction platform, Contracts component).

## Status

Published. The API contract is at version `1.0.0-alpha.4` and the Symfony API implements it. Version 1.0.0 is not yet tagged, and the conformance fixtures that every backend will run are still being written.

## Contents

| File | What it covers |
|---|---|
| [CONTRACT_FIRST.md](CONTRACT_FIRST.md) | What contract-first means in this project, why four backends share one contract, and how conformance tests will work. |
| [openapi/faka-v1.yaml](openapi/faka-v1.yaml) | The API specification (OpenAPI), version `1.0.0-alpha.4`. |
| [openapi/CONVENTIONS.md](openapi/CONVENTIONS.md) | The conventions every operation follows: errors, idempotency, pagination, money. |
| [domain/domain-model.md](domain/domain-model.md) | The entities and how they relate. |
| [domain/state-machines.md](domain/state-machines.md) | The states of each Creator Action and Action, and of payments and settlement. |
| [events/](events/README.md) | The event envelope and how events are described. |

Related repositories:

- [docs-public](https://github.com/Faka-iAction/docs-public) — plain-language introduction, scope of the first release, [roadmap](https://github.com/Faka-iAction/docs-public/blob/main/ROADMAP.md) and glossary.
- [architecture-public](https://github.com/Faka-iAction/architecture-public) — the architecture overview and design decisions.

## Visibility

This repository is **public**. Everything in it is visible to anyone. It holds only sanitised content: no secrets, credentials, production infrastructure details, risk thresholds or commercial terms.

## Architecture

This component follows the FAKA iAction master architecture. FAKA iAction is the platform; FAKA iRequest is one vertical on it.

## Licence

All rights reserved, with a limited permission for educational use. See [LICENSE.md](LICENSE.md). The licence is a draft pending legal review.
