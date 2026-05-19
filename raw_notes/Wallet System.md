Key Management Architecture

Overview
A Chitt holder's private keys live in a keyring — an append-only encrypted blob stored on IPFS. Day-to-day operations are handled by sub-Chitt keys stored in secure device storage. The master Chitt private key is accessed only for high-stakes operations (creating new sub-Chitts, performing key rotations). Recovery is handled via a YubiKey-wrapped decryption key, independently of any primary service.

This document covers the key management landscape, discusses design tradeoffs, and explains how post-quantum considerations affect long-lived Chitts.

The Default Architecture: Keyring + YubiKey
The default key management model does not rely on a smart-contract wallet. It is:

Keyring (encrypted IPFS blob): Holds master Chitt private keys and sub-Chitt private keys, encrypted with a key derived from a passkey + service secret. The service holding the encrypted blob never sees plaintext. The blob is append-only: new keys are added without destroying old entries, preserving recoverability.

Sub-Chitt keys on device: Held in secure device storage (Secure Enclave on Apple devices, TPM on others). Used for all routine signing operations. The master Chitt key is not accessed during routine operations.

YubiKey recovery: A YubiKey-wrapped decryption key for the keyring is held by a backup service. If the primary service goes down, the user can recover their full keyring via the YubiKey + 72-hour notification window without the primary service's involvement. See the Recovery Flow document for full details.

Landscape of Wallet Architectures (Research Context)
The following is background on the broader wallet design space, for context when evaluating future architecture decisions:

Seed-phrase EOAs (the legacy model): Responsible for most lost funds in practice. No social recovery; loss of the seed phrase means permanent loss.

Smart-contract wallets with account abstraction (Argent, Safe, Coinbase Smart Wallet): Support guardian-quorum recovery — M-of-N trusted parties can initiate a time-windowed key rotation. Well-audited. Gas costs on L2 are negligible (~$0.01–$0.05 per operation). The Safe model is noted here as a potential substrate for any on-chain components of the Chitt protocol (e.g. the on-chain pointer registry contract), but is not the default key management approach for Chitt holders.

MPC/threshold-signature wallets (ZenGo, Fireblocks, Web3Auth): Distribute key shares across multiple parties. No single point of failure. Higher implementation complexity.

Passkey-backed wallets: A sub-pattern of smart-contract wallets. Use WebAuthn (Face ID, Touch ID) for authorization. Strong UX; passkeys are tied to device ecosystems, so a 1-of-1 passkey-only configuration is inadvisable.

On-Chain Components and L2 Choice
The Chitt protocol uses on-chain infrastructure for the mutable pointer registry and append-only log anchoring, but not for key custody in the default model. For these components:

Base or Optimism is recommended over Ethereum mainnet. Gas costs become negligible (divide mainnet costs by ~100–200). The EIP-7212 precompile is present on all major L2s, making P-256 (passkey) signature verification cheap (~3.5k gas vs. ~330k on mainnet without the precompile).

Gas costs (as of May 2026): pointer registry updates cost $0.01–$0.05 on L2. Wallet deployment (if using a smart-contract wallet for any components) runs ~$0.35–$0.40. Paymasters can sponsor gas so users never need to hold ETH.

Post-Quantum Considerations
NIST finalized ML-KEM, ML-DSA, and SLH-DSA in 2024, with FN-DSA and HQC in 2025. No major production wallet has migrated yet. For long-lived, high-value Chitts, the recommended migration pattern is:

Algorithm-agile Chitt formats: Specify signature algorithm in the Chitt metadata so future versions can use different algorithms without breaking the format.
Hybrid signatures during transition: Sign with both a classical algorithm and a post-quantum algorithm; verifiers accept either until migration is complete.
SLH-DSA for high-value long-lived Chitts (e.g., root-of-trust Chitts, authorizer Chitts), ML-DSA for routine ones.
