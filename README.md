# ERC-8004 Validation Network Interface (VNI)

A standard interface for permissionless, operator-diverse validator networks plugging into [ERC-8004](https://eips.ethereum.org/EIPS/eip-8004)'s Validation Registry.

**Status:** Draft v0.2 — Extension to ERC-8004

The full specification lives in [**SPEC.md**](./SPEC.md).

## What this is

VNI defines a contract interface, `IValidationNetwork`, that lets an ERC-8004 `validatorAddress` be a *network of independent validators* rather than a single party. A conforming network selects validators per a caller-supplied policy, collects signed attestations, and submits one aggregated response through the existing Validation Registry — with operator-diversity as a first-class policy parameter.

The proposal is strictly additive: the Validation Registry is not modified, single-address validators keep working, and any sufficiently decentralized substrate (permissionless RPC networks, restaking AVSs, TEE consortia, oracle networks) can implement it.

## Authors

- Chris "Jinx" Jenkins (Pocket Network Foundation) — @TheFeloniousMonk
- Luis Correa de León (Synaptika) — @luyzdeleon
- Bryan White — @bryanchriswhite
- [additional co-authors TBD]

## License

[CC0](./LICENSE) — copyright and related rights waived.
