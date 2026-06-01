# Changelog

All notable changes to the ERC-8004 Validation Network Interface (VNI) specification
are documented in this file.

Format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This spec uses Draft `v0.x` versioning until a frozen `v1`; while in draft, the
`interfaceId` is intentionally left unpinned (see SPEC.md → Backwards Compatibility).

## [Unreleased]

### Added
- **ERC-165 conformance is now normative.** `IValidationNetwork` inherits `IERC165`;
  a conforming network MUST implement `supportsInterface` and MUST return true for
  `type(IValidationNetwork).interfaceId`. This is the canonical way a client detects
  that an ERC-8004 `validatorAddress` is a VNI network, and it removes any need for a
  separate validation-network registry. The literal `interfaceId` is left unpinned
  until the interface surface is frozen at v1 (it shifts with any signature change).

### Removed
- **EIP-7702 open question.** Sponsored validation is a separate trust boundary and
  should not affect validator selection or verdict logic; deferred to a possible
  follow-on extension once the base interface stabilizes.

## [0.2] - 2026-05-29

### Added
- Initial public draft of the `IValidationNetwork` interface: `submit`, `quote`,
  `policySchema`, `supportsPolicy`, `status`, errors, and events.
- Canonical `SelectionPolicy` struct and `schema/policy-v1.schema.json` (JSON Schema,
  Draft 2020-12).
- EIP-712 attestation envelope, aggregated-response file format (JCS-hashed),
  normative response-tag vocabulary, assurance tiers, and challenge-kind registry.
- Rationale entry clarifying the extension decentralizes **selection, not aggregation**,
  and that `minOperators` is bounded by each network's published operator-identification
  methodology.
- `discussions-to` frontmatter and a README link to the Ethereum Magicians thread
  (added 2026-06-01).

### Changed
- Renamed `supports()` → `supportsPolicy()` prior to first publish: `supports` is a
  Solidity reserved word (compiles today, would break and shift the interface ID if
  promoted to a keyword), and the new name disambiguates from ERC-165's
  `supportsInterface` and aligns with the existing `PolicyNotSupported` error.
- Repointed `policySchema()` from a dangling `eips.ethereum.org` anchor to the
  self-hosted raw schema URL, marked provisional pending migration to a canonical
  EIP anchor.
