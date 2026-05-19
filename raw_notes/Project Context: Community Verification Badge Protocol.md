Project Context: Community Verification Chitt Protocol

Core Concept
A verification protocol where community members hold cryptographic Chitts proving membership. Some Chitts carry issuance privileges, allowing holders to authorize a Chitt Press to produce further Chitts for others (with attenuated scope). Chitts can sign statements, providing verifiable trust chains. Storage is on IPFS; the mutable pointer registry and log anchoring live on an L2 blockchain (Base or Optimism). Verification walks the chain confirming each signature against the public key from the prior link, checking the append-only log for revocations at each step.

Key Design Elements
Mutual signing during issuance: the enclave (as issuer) signs the proposed Chitt JSON; the recipient adds their public key and countersignature to complete the Chitt.
Policy-gated issuance: all Chitts are produced by a Chitt Press running attested code under an approved policy signed by an authorizing Chitt.
Invite-only ecosystem: first-time Chitt receipt happens via an invitation link containing the signed Chitt JSON; no prior Chitt is required to receive an invitation.
Two-tier key architecture: master Chitt keys in an encrypted keyring; sub-Chitt keys on devices for routine signing.
YubiKey recovery: keyring recovery via a YubiKey-wrapped decryption key, independent of the primary service.
Mutable pointer identity: Chitts are identified by stable mutable pointers, not content hashes. The pointer resolves to an append-only log that records all updates and revocations.
Two distinct annotation systems: issuer annotations (entries in the Chitt's own log, governed by annotation policy) and third-party annotations (EAS/IPFS, filtered by trusted annotator Chitts).
AI-generated summaries of verification chain context (planned feature).
Integration with messaging via Nym gateway addresses embedded in each Chitt.

Resolved Design Decisions
The following questions were open in early design and have since been resolved:

Naming: The canonical term is "Chitt." A device-level key is a "sub-Chitt." The issuance function is a "Chitt Press."
Chitt identity: Mutable pointer, not CID. The pointer is stable across all updates; it resolves to the append-only log.
Revocation substrate: The append-only log associated with the mutable pointer is the canonical revocation mechanism. Log roots are anchored on-chain for rollback resistance.
Pointer registry: On-chain registry contract on Base or Optimism. Chosen over IPNS for resolution latency (~50–200ms vs. 7–11s) and cost ($0.01–$0.05 per update on L2).
Issuance model: Gated enclave (Chitt Press) with approved policy. Enclaves are operated as a service and accept arbitrary approved policies. Not an open permissionless issuance model.
First-Chitt flow: Invitation link. The ecosystem is invite-only. The link contains the signed Chitt JSON and directs the recipient to create a new keyring.
Keypair generation: The enclave sends the proposed Chitt JSON without the recipient's public key. The recipient generates a keypair, adds the public key, and countersigns. The completed Chitt is posted to IPFS without further enclave involvement.
Annotation systems: Issuer annotations and third-party annotations are distinct. Annotation policies govern who may issue annotations of which types to a given Chitt.
Wallet/recovery: No smart-contract wallet in the default model. The keyring is an append-only encrypted blob on IPFS; recovery uses a YubiKey-wrapped decryption key via a backup service with a 72-hour notification window.
Authentication identity: Services bind accounts to the mutable pointer, not any version CID.
Matrix/room signing: Sub-Chitt keys, consistent with all other routine signing. Master key never used for routine operations.

Messaging Integration
Goal: meet users in familiar messaging apps to avoid the "nobody installs PGP" failure mode.
Recommended platform: Matrix — real bot API, federated, proper E2E, aligns with decentralization ethos. Prototype on Matrix, port to consumer messengers once flows are proven.
Platform notes: WhatsApp Business API has 24-hour session windows and template restrictions; Signal has no official bot API; Telegram is permissive but has weak E2E.
Trust bootstrap: Bot identity is a Chitt. One-time fingerprint verification at first contact, with human-readable confirmation hashes checked out of band.

Similar / Related Protocols
Direct ancestors and conceptual cousins:

PGP Web of Trust — original peer-signed identity attestation; instructive failure modes around key management UX, revocation, and chain evaluation
W3C Verifiable Credentials (VCs) — issuer/holder/verifier model, mature ecosystem (Veramo, Trinsic, Spruce)
Decentralized Identifiers (DIDs) — typically paired with VCs; provides updatable identity documents useful for revocation pointers
UCAN (User Controlled Authorization Networks) — signed capability tokens with attenuating delegation chains, content-addressed storage; closest existing match to the proposed design

Badge-specific protocols:

Mozilla Open Badges — long-running educational credentials standard
Nostr NIP-58 — badge protocol over signed-event network
Soulbound Tokens — Vitalik's blockchain framing of non-transferable credentials

Transparency and revocation infrastructure:

Certificate Transparency — append-only log model for credential issuance
Sigstore / Rekor — recent well-engineered take on signed artifacts plus transparency log plus revocation; design decisions translate directly

Notary and attestation systems:

Reclaim Protocol — closest productized match for HTTPS-verified attestations as Chitt Press inputs
TLSNotary, DECO — zkTLS family; useful for wrapping external services as attestation sources

Open Questions to Carry Forward

Attenuation predicate structure: Exact format for scope constraints at each chain link (Macaroon caveats vs. UCAN-style capability narrowing vs. custom).
Harm taxonomy: Standardized category codes for third-party safety annotations, modeled on existing T&S frameworks (TSPA, GIFCT).
Annotation policy defaults: What the default annotation policy is for Chitts that don't specify one, and how annotation policy inheritance works across chain links.
Post-quantum migration: Timeline and tooling for adopting ML-DSA / SLH-DSA for long-lived authorizer Chitts.
Notification semantics for cascading revocation: How downstream Chitt holders are informed when an upstream Chitt in their chain is revoked.
Multi-Chitt account support: Whether and how services should support multiple Chitts (different types) under one account, and what the UX looks like.
