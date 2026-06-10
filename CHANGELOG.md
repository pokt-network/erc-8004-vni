# Changelog

All notable changes to the ERC-8004 Validation Network Interface (VNI) specification
are documented in this file.

Format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This spec uses Draft `v0.x` versioning until a frozen `v1`; while in draft, the
`interfaceId` is intentionally left unpinned (see SPEC.md → Backwards Compatibility).

## [Unreleased]

### Added
- **ERC-165 conformance is now normative.** `IValidationNetwork` inherits `IERC165`;
  a conforming network MUST implement `supportsInterface`, MUST return true for both
  `0x01ffc9a7` (`type(IERC165).interfaceId`) and `type(IValidationNetwork).interfaceId`,
  MUST return false for `0xffffffff`, and SHOULD answer in at most 30,000 gas. This is
  the canonical way a client detects that an ERC-8004 `validatorAddress` is a VNI
  network, and it removes any need for a separate validation-network registry. The
  literal `interfaceId` is left unpinned until the interface surface is frozen at v1.
- **Identity definitions.** Distinguish validation network, validator, and operator so
  `minOperators` is not misread as a validator count.
- **Validation lifecycle.** Document the Unknown → Accepted → Validators-selected →
  Responded / Failed-terminal progression, and the rule that a written Validation
  Registry response is canonical (later attestations MUST NOT change it).
- **Eligibility methodology.** Networks SHOULD publish a versioned, content-addressed
  operator-identification / eligibility methodology and SHOULD anchor its hash on-chain.
- **`responseURI` semantics.** Treat `responseURI` as an opaque, network-defined locator;
  the portable invariant is that the resolved file hashes to the recorded `responseHash`.
- **Payment lifecycle.** Document finite vs `OUT_OF_BAND_PRICE` pricing and that generic
  clients MUST NOT infer refund behavior from non-`vni:ok` tags.
- **`verificationProfile()` introspection.** A stable `bytes32` identifier for a network's
  aggregation / verification model, complementing `supportsPolicy()` without a registry.
- **`wyriwe-input-provenance-v1` challenge kind** and a non-normative ERC-8263 composition
  note (`agentId` stays `uint256`; REGISTRY scheme `0x01` anchors as `bytes32(uint256(agentId))`).

### Changed
- **Submitted to `ethereum/ERCs` ([PR #1808](https://github.com/ethereum/ERCs/pull/1808)); assigned ERC-8294.**
  ERC title set to "Validation Network for ERC-8004" (dropped "Interface" per editor review);
  `eip: 8294` and updated `discussions-to` slug.
- **Aggregated-response `version` → `schema`**, with canonical value
  `erc-8004-vni/aggregated-response/v1`.
- **`evidenceHash`** is computed over the canonical, unframed evidence payload, independent
  of any transport envelope.
- **`submit()` sequencing.** Callers SHOULD ensure `submit()` observes the intended
  `validatorAddress` (e.g. bundle with `validationRequest()`) before work begins.
- **`attestationsRoot`** single-attestation case defined as the lone EIP-712 struct hash,
  with no Merkle wrapping.
- **EIP-712 attestation layout is normative**; contract-specific submission/storage
  function shapes are implementation-defined.
- Recorded current-lean positions for the TEE, slashing, cost-discovery, and
  multi-network open questions; expanded AVS on first use.

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
