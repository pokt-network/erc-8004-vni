---
title: "ERC-8004 Validation Network Interface"
description: A standard interface for permissionless, operator-diverse validator networks plugging into ERC-8004's Validation Registry
status: Draft v0.2
type: Extension to ERC-8004
discussions-to: https://ethereum-magicians.org/t/erc-8004-validation-network-interface-extension-for-multi-validator-networks/28669
authors:
  - Chris "Jinx" Jenkins (Pocket Network Foundation) @TheFeloniousMonk
  - Luis Correa de León (Synaptika) @luyzdeleon
  - Bryan White @bryanchriswhite
  - "[additional co-authors TBD]"
created: 2026-04-13
revised: 2026-06-01
extends: ERC-8004 (https://eips.ethereum.org/EIPS/eip-8004)
license: CC0
---

# ERC-8004 Validation Network Interface

## Abstract

This proposal extends ERC-8004's Validation Registry by defining a standard contract interface, `IValidationNetwork`, that allows a `validatorAddress` to be a network of independent validators rather than a single party. A conforming network selects validators according to a caller-supplied policy, collects signed attestations from the selected set, and submits a single aggregated response back through the existing Validation Registry. The extension introduces operator-diversity as a first-class policy parameter and standardizes the attestation envelope so that responses from any conforming network can be verified by any client using the same code.

The proposal is strictly additive. The Validation Registry contract is not modified, single-address validators continue to work unchanged, and any sufficiently decentralized network — permissionless RPC networks, restaking-based AVSs, TEE consortia, decentralized oracle networks — can implement the interface.

## Motivation

ERC-8004's Validation Registry is intentionally unopinionated about who validates. Its `validationRequest(validatorAddress, agentId, requestURI, requestHash)` accepts any address; the spec leaves "incentives and slashing related to validation … outside the scope of this registry."

This is a deliberate design choice and a correct one. It leaves room for many validator implementations to compete on the merits.

In practice, three patterns have emerged:

- **Single-validator addresses** — operationally simple, but reintroduce the centralized-trust assumption that ERC-8004 was designed to relax.
- **TEE-based attesters** — strong cryptographic guarantees, but with a hardware-trust dependency and a limited set of qualifying operators.
- **Bespoke threshold schemes** — each implementer rolls a custom multi-validator setup with its own selection rules, attestation format, and verification path. Clients integrating with multiple networks must write per-network verification code.

The third pattern is where the gap lives. There is no shared interface that lets a client say "I want a validation, run by N independent validators, drawn from at least D distinct operators, with a deadline of T seconds, and give me back a verifiable bundle of their signed attestations." Every team that wants this property is building it from scratch.

This proposal defines that interface. It does not prescribe a selection algorithm, an incentive model, a slashing model, or a network topology. It standardizes the contract surface that lets clients request multi-validator validation in a portable way and verify the result without per-network glue code.

## Specification

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, NOT RECOMMENDED, MAY, and OPTIONAL in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Overview

A validation network is a smart contract deployed at an address that conforms to `IValidationNetwork`. From the perspective of ERC-8004's Validation Registry, this address is the `validatorAddress` passed to `validationRequest`. Internally, the contract performs validator selection, accepts off-chain attestations from selected validators, and writes back to the Validation Registry via `validationResponse` with an aggregated result.

The extension adds three things on top of ERC-8004:

- A contract interface (`IValidationNetwork`) that networks implement.
- A policy schema (`SelectionPolicy`) that callers pass to express assurance requirements.
- An attestation envelope (`Attestation`) signed by individual validators and aggregated into the response.

### Contract Interface

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.20;

/// @notice Minimal view of ERC-8004's Validation Registry needed by IValidationNetwork.
interface IValidationRegistryView {
    function getValidationStatus(bytes32 requestHash) external view returns (
        address validatorAddress,
        uint256 agentId,
        uint8 response,
        bytes32 responseHash,
        string memory tag,
        uint256 lastUpdate
    );
}

/// @notice ERC-165 standard interface detection.
interface IERC165 {
    function supportsInterface(bytes4 interfaceId) external view returns (bool);
}

/// @dev A conforming network MUST implement ERC-165. supportsInterface(interfaceId)
///      MUST return true for both type(IERC165).interfaceId (0x01ffc9a7) and
///      type(IValidationNetwork).interfaceId. This is the canonical way for a client
///      to detect that a given ERC-8004 validatorAddress is a VNI network before
///      submitting a request. See Backwards Compatibility for the pinned interface ID.
interface IValidationNetwork is IERC165 {
    /// @notice Sentinel returned by quote() when pricing is settled out-of-band (e.g. via x402).
    /// @dev A return of (OUT_OF_BAND_PRICE, etaSeconds) instructs the caller to obtain pricing
    ///      via the network's documented out-of-band channel. (0, 0) means free.
    ///      Any other finite value is a literal price in wei.
    function OUT_OF_BAND_PRICE() external view returns (uint256); // = type(uint256).max

    /// @notice The Validation Registry this network reads from and writes responses to.
    /// @dev MUST return the address of the ERC-8004 Validation Registry contract that
    ///      this network is bound to. Set at construction; immutable for the lifetime of the contract.
    function validationRegistry() external view returns (address);

    /// @notice Submit a validation request to this network.
    /// @dev MUST be called after the corresponding validationRequest() has been submitted to
    ///      the ERC-8004 Validation Registry. Implementations MUST, before doing any work,
    ///      call IValidationRegistryView(validationRegistry()).getValidationStatus(requestHash)
    ///      and verify that the returned validatorAddress equals address(this). If it does
    ///      not, implementations MUST revert with NotAddressee(requestHash). This check
    ///      prevents griefing where a third party burns a network's resources on requests
    ///      addressed to a different validator.
    ///
    ///      Implementations SHOULD also reject duplicate submissions for the same requestHash
    ///      (revert with AlreadySubmitted) and policies they cannot satisfy (revert with
    ///      PolicyNotSupported).
    /// @param requestHash The keccak256 commitment recorded in ERC-8004's Validation Registry.
    /// @param policy ABI-encoded SelectionPolicy. See policySchema().
    /// @param payload Optional opaque payload (e.g. a challenge nonce). Network-defined.
    function submit(bytes32 requestHash, bytes calldata policy, bytes calldata payload) external payable;

    /// @notice Quote the price (in wei) and expected ETA (in seconds) for a given policy.
    /// @dev Returns (0, 0) for free requests. Returns (OUT_OF_BAND_PRICE, etaSeconds) when
    ///      pricing is settled out-of-band (e.g. via x402). Any other finite priceWei is a
    ///      literal on-chain price. A generic client MUST treat OUT_OF_BAND_PRICE as a
    ///      signal to obtain pricing through the network's documented out-of-band channel
    ///      and not as a literal value.
    function quote(bytes calldata policy) external view returns (uint256 priceWei, uint32 etaSeconds);

    /// @notice URI of the policy schema this network accepts.
    /// @dev SHOULD return a stable URI resolving to a JSON schema document.
    ///      Networks implementing the canonical SelectionPolicy struct (this spec)
    ///      SHOULD return the current schema URI:
    ///      "https://raw.githubusercontent.com/pokt-network/erc-8004-vni/main/schema/policy-v1.schema.json".
    ///      This URI is provisional for Draft v0.2 and will migrate to the canonical
    ///      eips.ethereum.org anchor if this extension is accepted as an ERC.
    function policySchema() external view returns (string memory);

    /// @notice Whether this network can serve a given policy.
    /// @dev Networks MUST return false for policies whose minOperators exceeds the
    ///      network's distinct-operator capacity. Networks SHOULD return false for any
    ///      policy they would otherwise revert on at submit() time; supportsPolicy() is a
    ///      pure pre-flight check and SHOULD NOT be used for addressee verification.
    function supportsPolicy(bytes calldata policy) external view returns (bool);

    /// @notice Status of an in-flight or completed request.
    /// @return state 0=unknown, 1=pending, 2=responded, 3=failed
    /// @return validatorCount Number of validators selected.
    /// @return responseAggregated Whether validationResponse() has been written to the Validation Registry.
    function status(bytes32 requestHash) external view returns (
        uint8 state,
        uint16 validatorCount,
        bool responseAggregated
    );

    /// @notice Reverted by submit() when getValidationStatus(requestHash).validatorAddress != address(this).
    error NotAddressee(bytes32 requestHash);

    /// @notice Reverted by submit() when called twice for the same requestHash.
    error AlreadySubmitted(bytes32 requestHash);

    /// @notice Reverted by submit() when supportsPolicy(policy) would have returned false.
    error PolicyNotSupported(bytes policy);

    /// @notice Emitted when a request is accepted into the network.
    event RequestAccepted(
        bytes32 indexed requestHash,
        bytes policy,
        uint256 priceWei
    );

    /// @notice Emitted when validators have been selected for a request.
    event ValidatorsSelected(
        bytes32 indexed requestHash,
        address[] validators,
        bytes32 selectionEntropy
    );

    /// @notice Emitted when individual attestations are aggregated and submitted to the Validation Registry.
    event AttestationsAggregated(
        bytes32 indexed requestHash,
        address[] validators,
        bytes32 attestationsRoot,
        uint8 aggregatedVerdict
    );
}
```

### Selection Policy

The canonical policy is ABI-encoded as the following struct. Networks MAY accept extended policies as long as their decode begins with these fields.

```solidity
struct SelectionPolicy {
    uint8 version;          // MUST be 1 for this revision
    uint8 selectionSize;    // N: number of validators to select; MUST be >= 1
    uint8 minOperators;     // D: minimum distinct operators across the selected set; MUST be <= N
    uint8 minResponses;     // M: minimum attestations required for a valid aggregated response; MUST be <= N
    uint64 deadlineSeconds; // Wall-clock budget from submit() to aggregated response
    uint8 verdictMode;      // 0=any-pass, 1=majority, 2=unanimous
    bytes32 challengeKind;  // Network-defined kind tag (e.g. keccak256("rpc-equivalence"))
    bytes extensions;       // Network-specific opaque field; MUST decode to empty for canonical policies
}
```

Networks MUST treat any policy whose version they do not recognize as unsupported and `supportsPolicy()` MUST return false.

### Operator Diversity

`minOperators` is the central decentralization knob and MUST be enforced at selection time, not at response time. A network whose effective operator count is O and which receives a policy with `minOperators` > O MUST refuse the request via `supportsPolicy()` returning false rather than silently degrading.

How a network defines "operator" is network-specific (staking address, key cluster, self-declared operator ID, etc.) and SHOULD be documented in a public operator-identification methodology alongside the deployed contract. The methodology document is the artifact a third-party auditor uses to verify the network's diversity claims. Pseudo-anonymous Sybil at the operator level is a known risk; networks SHOULD describe their mitigations in the same document.

### Attestation Envelope

Validators sign attestations off-chain. The signed payload is an EIP-712 typed-data structure:

```solidity
EIP712Domain {
    string name;        // "ERC8004ValidationNetwork"
    string version;     // "1"
    uint256 chainId;
    address verifyingContract;  // the IValidationNetwork contract
}

Attestation {
    bytes32 requestHash;
    uint256 agentId;
    address validator;
    uint8 verdict;          // 0..100, same scale as ERC-8004 validationResponse
    bytes32 evidenceHash;   // keccak256 of off-chain evidence file
    uint64 issuedAt;        // unix seconds, validator's view
    bytes32 challengeKind;  // copied from policy
    bytes32 nonceHash;      // keccak256 of the network-issued nonce, if any
}
```

The off-chain aggregated response file referenced from the Validation Registry's `responseURI` MUST contain:

```json
{
  "version": "vni-v1",
  "requestHash": "0x...",
  "policy": "0x...",
  "validators": ["0x...", "0x..."],
  "attestations": [
    {
      "validator": "0x...",
      "verdict": 100,
      "evidenceHash": "0x...",
      "issuedAt": 1745870000,
      "challengeKind": "0x...",
      "nonceHash": "0x...",
      "signature": "0x..."
    }
  ],
  "attestationsRoot": "0x...",
  "aggregatedVerdict": 100,
  "verdictMode": "majority"
}
```

The `attestationsRoot` is a Merkle root over the per-validator attestation hashes, included so a holder of a single attestation can prove inclusion without the full file.

The aggregated `responseHash` written to the Validation Registry is the keccak256 of the canonical JSON serialization of this file (RFC 8785 / JCS).

### Aggregated Verdict

The aggregated verdict written to ERC-8004's `validationResponse()` is binary in this revision and computed per `verdictMode`:

- **any-pass**: 100 if any received attestation reports verdict ≥ 50, else 0.
- **majority**: 100 if more than half of received attestations report verdict ≥ 50, else 0.
- **unanimous**: 100 if all received attestations report verdict ≥ 50, else 0.

In all three modes, the aggregated verdict is exactly 0 or 100. Spectrum verdicts (e.g., a literal mean of received verdicts) are deferred; see Open Questions.

The verdict alone does not distinguish a successful failed-validation from a timeout or other operational failure. That distinction is communicated through the Validation Registry's `tag` field; see Response Tags.

### Response Tags

The `tag` argument to ERC-8004's `validationResponse()` carries the network's outcome status for the request. Networks conforming to this extension MUST set `tag` to exactly one of the following normative values:

| Tag | Meaning |
| --- | --- |
| `vni:ok` | Aggregated response produced from at least `minResponses` attestations. The verdict is meaningful. |
| `vni:timeout` | `deadlineSeconds` elapsed before `minResponses` attestations were collected. Verdict MUST be 0. |
| `vni:insufficient-operators` | Selection could not satisfy `minOperators` from the network's effective operator set after entry. Verdict MUST be 0. Networks SHOULD prefer rejecting at `supportsPolicy()` and reverting at `submit()` over reaching this state. |
| `vni:cancelled` | The request was cancelled (e.g., by the network operator under documented emergency procedures) before completion. Verdict MUST be 0. |

Networks MAY define additional tag values for network-specific outcomes; non-normative tags MUST be prefixed `vni:x-` to avoid collision with future normative additions.

A client interpreting an aggregated response MUST treat any tag other than `vni:ok` as a non-meaningful verdict regardless of the numeric value, and MUST NOT use the verdict to update reputation or downstream state.

### Assurance Tiers (Informative)

The following tiers are provided as recommended starting points. They are not normative; clients construct their own policies.

| Tier | selectionSize | minOperators | minResponses | Typical use |
| --- | --- | --- | --- | --- |
| 1 | 1 | 1 | 1 | Cheap signal, equivalent to single-validator |
| 2 | 3 | 2 | 2 | Default for low-stakes agent-to-agent |
| 3 | 5 | 3 | 4 | Default for payment-bearing flows |
| 4 | 7 | 5 | 6 | High-stakes, multi-operator floor |

Empirical analysis of any specific network's ability to satisfy each tier MUST be published alongside the deployed contract. See Security Considerations.

### Challenge Kinds (Informative)

The `challengeKind` field tags the type of validation being requested. Networks MAY define their own kinds; the following are suggested as starting points:

- `keccak256("identity-control-v1")` — verify the agent controls a claimed key by signing a network-issued nonce.
- `keccak256("rpc-equivalence-v1")` — verify the agent's RPC endpoint returns results consistent with a reference set.
- `keccak256("a2a-card-fetch-v1")` — verify the agent's A2A AgentCard is reachable and well-formed.
- `keccak256("tee-attestation-pass-through-v1")` — verify a presented TEE attestation; the network bridges, does not re-execute.

The full list and the verification semantics for each kind belong in a separate, evolving registry document.

## Rationale

**Why network-agnostic.** Standards bodies correctly reject extensions that look like infrastructure land-grabs. The interface is defined so that any network — a permissionless RPC network, a restaking AVS, a TEE consortium, a decentralized oracle network — can conform. The specific Pocket reference implementation is described in a separate document and is not part of this proposal.

**Why operator-diversity as a first-class field.** Multi-validator selection where all validators share an operator is collusion-by-default and provides no meaningful security improvement over single-validator. The policy must let callers express the diversity they want, and the network must enforce it at selection time. Burying this in a tier abstraction or a network-specific extension hides the most security-relevant parameter.

**Why "network of independent validators" does not mean end-to-end trustlessness.** Read the phrase as "diverse, auditable selection plus a single aggregation locus," not as a trustless pipeline. This extension distributes the *inputs* to a validation — which validators are chosen, and the signed attestations they each produce — but the aggregated verdict is still computed and written through a single `submit()` flow at one contract. The decentralization gain is in selection diversity and the auditable per-validator attestation set, not in the tally itself. The interface deliberately exposes the full validator set in the aggregated response (rather than collapsing it to one number) so clients can re-derive the verdict and build their own collusion detectors; it does not claim the aggregation step is trustless. Equally, the contract can *count* distinct operators but cannot *prove* they are independent — `minOperators` is enforced against a network's self-declared operator identities, so the strength of the "independent" claim reduces to the credibility of the network's published operator-identification methodology (see Security Considerations).

**Why aggregate on-chain via existing Validation Registry.** The Validation Registry already stores `validatorAddress`, `requestHash`, `response`, `responseHash`, `tag`, `lastUpdate`. This is sufficient for an aggregated response. Forking the registry would split tooling, indexers, and explorers (8004scan, agentscan, etc.) for no gain.

**Why EIP-712 typed-data attestations.** Wallets and standard libraries already verify EIP-712. A custom signing scheme would force every client integration to bring its own verifier.

**Why JCS canonical JSON.** The off-chain aggregated file must hash deterministically across implementations. JCS is the cheapest path to that property.

**Why the extensions field in SelectionPolicy.** Networks need room to evolve. Canonical policies decode `extensions` to empty and ignore the rest; network-aware clients can pack additional fields without breaking compatibility.

**Why mandatory addressee verification on submit().** Without an explicit check, a third party who watches the Validation Registry can call any network's `submit()` for a `requestHash` that was registered with a different `validatorAddress`. The network has no contract-level signal that it is not the legitimate addressee and may burn resources on a request it should never have accepted. Requiring `submit()` to read `getValidationStatus(requestHash)` and revert with `NotAddressee` if the recorded `validatorAddress` is not `address(this)` closes the griefing path at the interface layer rather than relying on per-network convention.

**Why binary aggregated verdicts.** A spectrum verdict (e.g., the mean of received attestations) collapses the timeout case into the success case: a timeout that produced no attestations is indistinguishable from "every validator returned 0," and a partial-response under spectrum mode is indistinguishable from a full-response with low scores. Binary verdicts plus an explicit tag vocabulary preserve that distinction without forcing every generic client to branch on `verdictMode` to interpret `aggregatedVerdict`. Spectrum verdicts may be revisited in a follow-on extension.

**Why a normative tag vocabulary.** ERC-8004's Validation Registry already stores a free-form `tag` field. Without a vocabulary, every implementation invents its own and indexers cannot distinguish a timeout from a failed validation from a successful negative verdict. A small normative set (`vni:ok`, `vni:timeout`, `vni:insufficient-operators`, `vni:cancelled`) covers the cases that matter for downstream state and leaves room for network-specific values under the `vni:x-` prefix.

**Why a sentinel for out-of-band pricing in quote().** Returning (0, 0) to mean "settled out-of-band" collides with returning (0, 0) to mean "free." A generic client cannot distinguish the two and cannot know whether to fall back to an x402-style discovery flow. A sentinel (`type(uint256).max`, exposed as `OUT_OF_BAND_PRICE()`) makes the distinction explicit and verifiable.

**Why `supportsPolicy`, not `supports`.** `supports` is on Solidity's reserved-words list (alongside `sealed`, `typedef`, `match`, `static`). It compiles under 0.8.x today, but were it ever promoted to a real keyword, every conforming implementation would break and the interface ID would shift. `supportsPolicy` also disambiguates from ERC-165's `supportsInterface`, which answers a different question, and lines up with the existing `PolicyNotSupported` error.

## Backwards Compatibility

This proposal is strictly additive. It defines a new interface contract type and a payload format for `requestURI` and `responseURI` files. ERC-8004's Validation Registry contract is not modified. Existing single-address validators continue to work unchanged.

Conformance is detectable via ERC-165. A conforming network MUST implement `IERC165` and MUST return true from `supportsInterface(interfaceId)` for `type(IValidationNetwork).interfaceId`. A client resolves whether a given ERC-8004 `validatorAddress` is a VNI network by calling `supportsInterface` on that address; an address that returns false (or does not implement ERC-165) is not a VNI network and is treated as an opaque single-address validator. This is the canonical discovery path and removes the need for any separate validation-network registry.

> **TODO (v1 freeze):** pin the literal `type(IValidationNetwork).interfaceId` (the XOR of all `IValidationNetwork` function selectors) here as a fixed `bytes4` constant. It is intentionally left unpinned for Draft v0.x because any change to a function signature shifts the value; it MUST be computed and frozen once the interface surface is final.

## Reference Implementation

A reference implementation against the Pocket Network supplier set will accompany v1, published in a companion repository. The reference demonstrates the interface backed by a permissionless network of approximately 5,000 supplier nodes operated across multiple independent operators, with pseudo-random selection drawn from the active session set.

The reference is one of several possible implementations. TEE consortia, restaking-based AVSs, and oracle networks are equally suitable substrates and the interface is designed to admit them on equal footing.

A naive single-validator implementation is also provided in the test repository as a baseline against which multi-validator implementations can be benchmarked.

## Security Considerations

**Operator concentration.** The strength of any conforming network's `minOperators` claim is bounded by its actual operator distribution. Networks MUST publish a current concentration analysis (operator count, top-K share, Herfindahl-Hirschman or equivalent, methodology used to cluster operators) alongside the deployed contract. Reviewers should treat unpublished or stale concentration analyses as cause to reject the network, not just the policy. A network with strong claims and no analysis is not credibly permissionless.

**Operator-level Sybil.** A single party splitting its stake across multiple apparent operators undermines `minOperators`. Networks SHOULD describe the methodology by which they cluster apparent-operator identities into actual-operator identities. The defense is methodology, not a contract field; the contract enforces the count, the methodology defines what counts as one.

**Validator collusion.** Pseudo-random selection plus operator-diversity floor reduces but does not eliminate collusion risk. The on-chain audit trail (per-validator addresses in the aggregated response) lets clients build their own collusion detectors over time. The interface intentionally exposes the validator set rather than aggregating it away.

**Replay.** Per-attestation `nonceHash` and `issuedAt` allow clients to enforce freshness windows. The network-issued nonce, when used, MUST be tied to `requestHash` to prevent cross-request replay.

**Stale attestations.** `issuedAt` is the validator's view, not the chain's. Clients SHOULD compare `issuedAt` against `lastUpdate` from the Validation Registry and reject responses with abnormal skew.

**Censorship by a single network.** A network can refuse a request or selectively serve. ERC-8004's Validation Registry already permits multiple `validatorAddress` values per agent over time, so clients with censorship concerns SHOULD diversify across networks rather than relying on a single one.

**Misrepresented network capabilities.** A network claiming to satisfy Tier 4 while operationally satisfying Tier 2 is the most likely misuse. The defense is the published concentration analysis plus third-party audits, not the contract.

**Liveness under partial validator failure.** `minResponses` < `selectionSize` admits liveness under partial failure but accepts the security trade-off that a smaller-than-expected attestation set is producing the verdict. Clients SHOULD set `minResponses` close to `selectionSize` for high-stakes applications.

**Cross-network composition.** The interface does not specify how to combine attestations from multiple networks. Clients composing across networks do so at the application layer; the spec does not currently propose a multi-network aggregator pattern.

## Open Questions

The following are unresolved and explicitly invited for co-author and community input:

- **Policy schema standardization.** This draft proposes a canonical `SelectionPolicy` struct. Should the schema be registry-defined (extensible per network) or fixed at this layer? Current lean: fixed canonical schema with an opaque `extensions` field for network-specific additions.
- **Validation network registry.** Should there be a contract registering known validation networks (analogous to the agent registry)? Current lean: no — clients pass `validatorAddress` directly to the Validation Registry, and discovery happens through agent endpoints. A registry may emerge organically at the indexer layer.
- **TEE attestation pass-through.** Should the spec define a normative `tee-attestation-pass-through-v1` challenge kind, or leave TEE bridging to TEE-implementer documentation?
- **Slashing surface.** Networks define their own slashing internally. Should the interface expose a normative `slash(validator, evidence)` hook so clients can trigger network-internal slashing in standard form, or is that strictly out of scope?
- **Cost discovery.** `quote()` returns a single price. Real networks may price differently per validator or per challenge kind. Is a single `uint256` expressive enough, or does this need to be a structured response?
- **Multi-network aggregation.** A natural follow-on is "request the same validation from K networks and combine." Is this a separate spec or an addendum here?
- **Spectrum verdicts as a follow-on.** v1 is binary-only by design (see Rationale). Should a follow-on extension define `verdictMode = mean` (or weighted-mean) plus the tag and timeout semantics needed to keep success and failure cases distinguishable? Current lean: yes, but only after at least one pilot demonstrates a use case where binary loses meaningful information.

## References

- ERC-8004: Trustless Agents — https://eips.ethereum.org/EIPS/eip-8004
- ERC-8004 discussion thread (Fellowship of Ethereum Magicians) — https://ethereum-magicians.org/t/erc-8004-trustless-agents/25098
- ERC-8004 awesome-list and reference implementations — https://github.com/sudeepb02/awesome-erc8004
- EIP-712: Typed structured data hashing and signing — https://eips.ethereum.org/EIPS/eip-712
- ERC-1271: Standard Signature Validation Method for Contracts — https://eips.ethereum.org/EIPS/eip-1271
- RFC 8785: JSON Canonicalization Scheme — https://www.rfc-editor.org/rfc/rfc8785
- A2A Protocol — https://github.com/google/A2A
- x402 — https://www.x402.org/

## Copyright

Copyright and related rights waived via [CC0](./LICENSE).
