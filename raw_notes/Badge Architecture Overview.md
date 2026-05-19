Chitt Protocol Architecture
Core Primitive: The Chitt
A Chitt is a cryptographic keypair plus a metadata document, identified by a mutable pointer. The mutable pointer — not the content hash — is the Chitt's stable identity. It is what gets passed around, embedded in messages, and used as an account identifier. The pointer resolves to the Chitt's current append-only log, which records the full history of updates, annotations, and revocations.

The metadata document lives on IPFS and contains:

The Chitt's public key
The mutable pointer back to this Chitt (self-reference, for log resolution)
The issuer's Chitt pointer and signature (chain link)
The Chitt's scope and attenuation rules
The Nym gateway address for inbound messages
The list of active sub-Chitt public keys with their re-encryption keys
A schema reference if the Chitt was issued by a template
An optional image CID pointing to a visual representation of the Chitt stored on IPFS, intended for display in a hexagonal frame

The metadata document is itself signed by the issuer, creating a verifiable chain from any Chitt back to a root of trust.

Component 1: Chitt Registry (IPFS + On-Chain Pointer Registry)
What it is: Two layers working together. IPFS provides content-addressed storage for Chitt metadata documents (version CIDs). A Solana program maps each Chitt's mutable pointer to the current version CID and maintains the append-only update log. Each Chitt is a Program Derived Account (PDA) under a single deployed program — one contract manages all Chitts, with PDA seeds separating them.
Core functions:

Store Chitt metadata documents on IPFS, returning a version CID
Resolve a mutable pointer to the current version CID (~400ms slot time, trivially cacheable)
Maintain the append-only log of all versions, with each log root anchored on-chain for rollback resistance and trusted timestamps
Store third-party annotation documents on IPFS, indexed via EAS

Key properties: The mutable pointer is stable across all updates. The version CID is what signatures commit to — it changes with every update, which is fine, because historical signatures reference the version CID they were made against. Updates cost ~$0.00025 per operation on Solana. One deployed program handles all Chitts; account rent (~0.002 SOL) is paid once per Chitt at creation.

Component 2: Keyring and Device Keys
What it is: Two-tier key management. The holder's master Chitt private key lives in an encrypted keyring stored on IPFS. Sub-Chitt keys for day-to-day operations live in secure device storage.
Core functions:
Keyring (encrypted IPFS blob):

Hold the master Chitt private key, encrypted with a key derived from passkey + service secret
Remain decryptable via a YubiKey-wrapped decryption key for recovery purposes
Be append-only: new keys are added; old keys are not deleted from the blob, preserving recoverability

Local device:

Hold sub-Chitt private keys in secure device storage (Secure Enclave, TPM)
Sign statements and messages day-to-day without decrypting the keyring
Authenticate to the message server to pull queued messages
Decrypt incoming messages using the device sub-Chitt private key

Key property: The master private key is accessed only when creating new sub-Chitts or performing high-stakes operations. All routine operations use sub-Chitt keys. Recovery does not require the original service — only the keyring blob (on IPFS) and the YubiKey.

Component 3: Inbound Message Transport (Nym)
What it is: A mixnet providing sender anonymity on the inbound leg. Senders route messages through Nym so that the message server receives messages without being able to observe who sent them or when.
Core functions:

Provide metadata-private routing from sender to the Chitt's Nym gateway
Obscure sender identity, timing, and traffic patterns through mix batching and shuffling
Deliver encrypted payloads to the Chitt's Nym gateway endpoint

Key property: Nym does one job — hiding sender metadata on the inbound leg. It is not used for storage, for device delivery, or for any other purpose. The payload it carries is already encrypted to the master Chitt public key; Nym cannot read it.
What it doesn't do: Durable storage, multi-device delivery, device-to-server communication. Those are handled by the message server.

Component 4: Message Server (Your Infrastructure)
What it is: A persistent server that acts as the Chitt's Nym gateway endpoint, proxy re-encryption service, and per-device message queue. This is the always-online component that bridges inbound Nym messages to offline devices.
Core functions:
Nym gateway:

Maintain a persistent Nym client connection to receive inbound messages
Accept encrypted payloads arriving from the Nym mixnet

Proxy re-encryption:

Hold re-encryption keys for each active sub-Chitt (generated at sub-Chitt creation, stored server-side)
Transform incoming ciphertexts encrypted to the master Chitt key into separate ciphertexts encrypted to each active sub-Chitt key using UMBRAL proxy re-encryption
Never sees plaintext — transformation is purely cryptographic

Per-device queue:

Store re-encrypted ciphertexts in a per-sub-Chitt queue
Apply configurable retention policies (delete after fetch, delete after N days, keep indefinitely)
Authenticate device connections via sub-Chitt signature challenge
Deliver queued ciphertexts to authenticated devices on request

Key property: The server sees that messages arrived and approximately when, but cannot read content (ciphertexts only) and cannot identify senders (Nym hides this on the inbound leg). Chitt holders who don't trust even this metadata visibility can run their own message server — the Nym gateway address is just a field in the Chitt metadata.
Upgrade path: If device check-in metadata becomes a concern, devices can connect via Nym rather than plain HTTP. The queue becomes a Nym-addressed endpoint rather than an HTTP endpoint. Architecture otherwise unchanged.

Component 5: Chitt Press (Gated Enclave Issuance)
What it is: A gated enclave service — running attested, audited code in a hardware-isolated environment — that produces new Chitts according to an approved policy. Chitt Presses are operated as a service and can accept arbitrary approved policies.
Core functions:

Accept a signed policy approved by an authorizing Chitt
Verify the authorizing chain on every issuance request
Accept additional inputs (co-signed statements, HTTPS attestations via Reclaim, manual approval, etc.)
Produce a new Chitt JSON — without the recipient's public key, which the recipient adds — signed by the enclave's keypair
Log each issuance in an append-only issuance log, encrypted with the policy authorizer's audit key

Privacy properties of the press:

The press never holds plaintext CIDs. The client encrypts the CID before handoff; the press posts ciphertext and never has the decryption key.
The press never knows the Chitt's address derivation secret. The client derives the PDA address locally and tells the press where to write. The press signs and submits the transaction without knowing why that address was chosen.
The press does record the CID in its issuance log — it necessarily knows the CID since it performed the IPFS upload and chain write. This record is encrypted with the policy authorizer's audit key, making the log readable only to the authorizer. This provides a recovery path: if a recipient loses their capability bundle, the authorizer can retrieve the CID from the press log and reissue the bundle.

Examples:

School enrollment press: accepts administrator Chitt + enrollment record → issues student Chitt
Reclaim press: accepts verified HTTPS response pattern → issues attestation Chitt
Expertise press: accepts journalism organization Chitt → issues journalist Chitt
Survey aggregator press: accepts encrypted survey inputs + ZK proof of aggregation → issues community health Chitt

Key property: Trust in an issued Chitt derives from trust in the policy, the authorizing Chitt, and the attested enclave code — not from trusting the enclave operator's intentions. The enclave operator cannot forge Chitts; they can only run or refuse to run the attested code.

Component 6: Chain Verification
What it is: The logic that walks a Chitt's trust lineage from a given Chitt back to a root of trust, checking every link. This is the core cryptographic primitive that makes the whole system meaningful.
Core functions:

Resolve the mutable pointer to the current version CID
Fetch the Chitt metadata document from IPFS
Verify the issuer's signature on the metadata document
Check the Chitt's append-only log for revocation entries
Walk up to the issuer's Chitt (the template Chitt, then the authorizer's Chitt) and repeat
Validate scope attenuation at each link (derived Chitt cannot exceed issuer Chitt's scope)
Return: valid chain to trusted root / revoked / invalid signature / scope violation

Key property: Stateless and verifiable by anyone. No trusted oracle needed — a verifier resolves the pointer, fetches from IPFS, and does the cryptography locally. Chain walks can be parallelized using the cached chain array embedded in each Chitt's metadata.

Component 7: Annotation Layer (IPFS + EAS)
What it is: A public, signed, append-only stream of third-party statements about Chitts. This is distinct from issuer annotations (which are entries in the Chitt's own append-only log). Third-party annotations are published by parties outside the issuance chain and are governed by annotation policies.
Core functions:

Publish a signed third-party annotation referencing a Chitt's mutable pointer
Resolve all third-party annotations for a given Chitt
Filter annotations by the signer's Chitt chain (show me only annotations from Chitts I trust)
Surface annotation context alongside Chitt verification results
Enforce annotation policies where present (e.g. only certain Chitt types may append metadata; only corroborated annotations surface as warnings)

Substrate: Ethereum Attestation Service (EAS) as the on-chain registry for annotation references, with annotation content stored on IPFS.
Key property: The annotation layer is the reputation accumulation surface. A Chitt is not just binary valid/revoked — it has a history of what others have said about it, weighted by the trust you place in the signers of those annotations.

Component 8: Matrix Room Integration (Separate Feature)
What it is: A distinct feature from private messaging. Matrix rooms are shared, persistent, multi-party spaces. Chitt integration here means using Chitts as access credentials to enter rooms and as signing identities within them.
Core functions:

Gate room entry: challenge entrants to prove they hold a Chitt satisfying specified requirements (chain, scope, claims)
Sign in-room messages with a sub-Chitt key so other members can verify the sender's Chitt chain (via the sub-Chitt-to-master link)
Surface Chitt metadata and annotation context alongside messages in the room UI
Support room-level policies: "only Chitts from this issuance chain can post," "only Chitts with fewer than N statements today can post" (spam control)

Key property: All signing in Matrix rooms uses sub-Chitt keys, consistent with the rest of the protocol. The master Chitt key is never used for routine operations. The room bot verifies the sub-Chitt-to-master link as part of entry verification.

Privacy Model

Two append-only logs exist in the system with different privacy requirements:

1. Chitt logs — the sequence of CID updates representing a Chitt's history. The owner chooses whether this is public or private.
2. Press logs — each press maintains a log of Chitts pressed under a given policy. Private by default, encrypted with the policy authorizer's audit key.

Chitt addresses and content are private by default. Privacy is a client-side choice; the contract is neutral. A Chitt can be made public simply by using a pubkey-derived address and storing the CID in plaintext — discoverable by anyone who knows the owner's public key. The privacy spectrum is:

Public: pubkey-derived PDA address, plaintext CID on-chain. Discoverable and readable by anyone.
Selectively shared: secret-derived PDA address, encrypted CID on-chain. Owner hands capability bundles to specific recipients.
Fully private: secret-derived address, encrypted CID, encrypted IPFS content. Content unreadable even to someone who obtains the CID.

Secret-derived addresses: rather than using a public key as the PDA seed, the client derives the seed from hash(sign(private_key, "chitt-log-v1")). The resulting account address is opaque — not linkable to any identity without the private key. The Ed25519 signature is deterministic, so the address is always recoverable from the same key.

Two keys per private Chitt:
- Address secret — derives the PDA. Controls who can find the account. Never shared.
- Decryption key — decrypts the on-chain CID. Grants read access. Can be shared independently.

Capability bundle: to share a private Chitt, the owner provides the recipient with an (address, decryption_key) pair. The key can be encrypted via ECDH to the recipient's public key, tying it to their identity and preventing trivial forwarding.

What an observer always sees: that transactions are happening to the program, when, and the fee payer (the press wallet). They cannot correlate transactions to identities, content, or each other without the address secret.

Key separation for policy authorizers: the policy control key and the audit log encryption key should be separate keypairs. A compromised audit key must not grant policy control, and vice versa.

How the components connect
Chitt created via invitation link
  → recipient receives signed Chitt JSON (without public key)
  → recipient creates keyring, generates keypair
  → recipient adds public key and countersignature to Chitt JSON
  → completed Chitt posted to IPFS
  → mutable pointer registered on-chain
  → sub-Chitt created: re-encryption key generated,
    stored on message server, sub-Chitt key stored on device

Someone sends a private message
  → resolves recipient's mutable pointer to current metadata
  → encrypts to master public key
  → routes through Nym mixnet
  → arrives at message server (sender hidden)
  → proxy re-encryption transforms to sub-Chitt ciphertexts
  → stored in per-device queue

Device comes online
  → authenticates to message server with sub-Chitt signature
  → downloads ciphertext queue
  → decrypts with local device key
  → resolves sender's mutable pointer, fetches metadata from IPFS
  → verifies sender chain and signature

Someone verifies a Chitt
  → resolves mutable pointer to current version CID
  → fetches metadata from IPFS
  → walks append-only log for revocation entries
  → walks chain (policy → template Chitt → authorizer Chitt)
  → fetches third-party annotations from EAS/IPFS
  → filters annotations by trusted annotator Chitts
  → returns: chain validity + revocation status + annotation context

Someone enters a Matrix room
  → room bot issues challenge nonce
  → entrant signs with sub-Chitt key
  → bot verifies sub-Chitt-to-master link
  → bot resolves master Chitt's mutable pointer, walks chain
  → access granted or denied

What the npm package exports
javascript// Chitt lifecycle
ChittProtocol.createChitt(options)
ChittProtocol.resolveChitt(mutablePointer)
ChittProtocol.verifyChitt(mutablePointer, trustedRoots)
ChittProtocol.revokeChitt(mutablePointer, masterKey)

// Sub-Chitt / device management
ChittProtocol.createSubChitt(masterChittPointer, devicePublicKey)
ChittProtocol.revokeSubChitt(subChittPointer)

// Chitt Press / issuance
ChittProtocol.deployPress(policy, enclaveEndpoint)
ChittProtocol.issueViaPress(pressId, requesterChitt, additionalInputs)

// Private messaging (Nym inbound, server queue, device pull)
ChittProtocol.sendMessage(recipientPointer, content, senderChitt?)
ChittProtocol.fetchMessages(subChittPointer, deviceKey)

// Annotations (third-party)
ChittProtocol.annotate(targetPointer, content, signingChitt)
ChittProtocol.getAnnotations(pointer, trustedAnnotatorRoots?)

// Matrix integration
ChittProtocol.createGatedRoom(requirements)
ChittProtocol.verifyRoomEntry(challenge, subChittProof)
ChittProtocol.signRoomMessage(content, subChittPointer, deviceKey)

What you're not building
Worth being explicit. The npm package does not include:

The Nym network itself (existing infrastructure)
IPFS (existing infrastructure)
EAS contracts (existing infrastructure)
The Matrix homeserver (existing infrastructure, operator brings their own)
The UMBRAL re-encryption library (existing library, you wrap it)
The on-chain pointer registry contract (deployed once, shared infrastructure)

What you're building is the opinionated glue layer — the Chitt format, the chain verification logic, the Chitt Press interface, the message server, and the integrations that make these components work together as a coherent trust primitive. That's a meaningful and tractable scope.
