Chitt Authentication: Detailed Overview

> **⚠ SUPERSEDED — Historical reference only.**
> This document was written when the registry substrate was Solana. The canonical decisions are in `specs/ARCHITECTURE.md` and `specs/chitt_protocol_spec.md` (v0.3, 2026-05-19). Key changes: "Solana address" → **Arbitrum One registry address**; signatures use **ML-DSA-44** (not Ed25519).

Concept
Chitt authentication lets a service gate access by requiring proof that a user holds a Chitt matching a specified policy. The user's keyring presents the Chitt, signs a challenge using a sub-Chitt key, and the service verifies both the signature and the Chitt's chain. The Chitt's mutable pointer becomes the durable account identifier; the Chitt's Nym gateway becomes the channel for server-to-user communication.

This is "Sign in with [community credential]" — but where the credential is verifiable, revocable, attenuable, and metadata-private, rather than an OAuth token from a centralized identity provider.

The Authentication Request
When a user lands on a gated page, the service generates an authentication request. The request is a signed object containing:

json{
  "request": {
    "requester_chitt": "<mutable pointer of Bob's Minecraft Server Chitt>",
    "policy": "cid:middle-school-student-policy",
    "challenge": "<32-byte random nonce>",
    "purpose": "Ongoing access to Bob's Minecraft Server",
    "session_id": "<opaque session identifier>",
    "callback": "https://minecraft.bobsserver.com/auth/callback",
    "expires": "2026-05-15T18:42:00Z"
  },
  "signature": "<requester's signature over the canonical request>"
}

The fields each do specific work:
requester_chitt identifies who is asking, by mutable pointer. The keyring app shows this to the user — "Bob's Minecraft Server is requesting a credential" — and the user can inspect Bob's Chitt chain, annotations, and reputation before deciding whether to authenticate. A phishing site can't impersonate Bob without compromising Bob's Chitt keys.
policy specifies what credential is acceptable, by reference to a published policy document. The policy describes which template chain the Chitt must derive from, what scope constraints apply, and any freshness requirements. This is richer than naming a single template Chitt pointer, because it survives template redeployments — when the school upgrades to a new press deployment, the policy ID stays stable and existing servers keep working without reconfiguration.
challenge is a fresh random nonce. The user's signature will bind to this nonce, so a captured authentication response can't be replayed against the same server in a later session.
purpose is a human-readable description of what the credential will be used for. The keyring app displays this so the user understands what they're authorizing, not just whom.
session_id lets the service correlate the eventual authentication response back to the specific browser session that initiated the request.
callback is where the keyring sends the response.
expires bounds how long the request is valid.
signature is the requester's signature over the canonical serialization of the request fields. The keyring verifies this before doing anything else: if the signature doesn't match the requester Chitt's keys, the request is rejected and the user never sees it. This is the defense against forged authentication prompts from phishing sites.

The whole request is encoded into a URL like:
chitt://request?r=<base64(signed_request)>
Or, equivalently, into a QR code containing the same payload for desktop-to-mobile handoff.

Keyring Evaluation
The user clicks the link (or scans the QR code), and the keyring app activates. It performs these steps before showing anything to the user:

Decode and verify the request. Parse the signed request blob. Resolve the requester Chitt's mutable pointer to get current metadata. Verify the request signature against the requester Chitt's keys. If verification fails, refuse to proceed and show an error.
Walk the requester's chain. Resolve the requester Chitt's mutable pointer and walk the issuance chain link by link, verify each signature, check the append-only log for revocations, and resolve third-party annotations. This populates the "who is asking" panel the user will see.
Fetch the policy. Resolve the policy CID from IPFS. The policy document specifies the match predicate: which template chain matching Chitts must derive from, what scope they must satisfy, any required additional claims, and freshness requirements.
Search the local keyring. Walk the user's held Chitts and identify those that satisfy the policy predicate. For each candidate, verify the chain to confirm it's still valid (signatures intact, revocations checked within the policy's freshness window). Discard candidates that don't currently verify.
Surface the results to the user. Depending on the number of matches:

No matches: "Bob's Minecraft Server is requesting a Middle School Student Chitt. You don't currently hold one." Optionally offer paths to obtain one if the policy publishes guidance on issuance.
One match: Show the requester, the purpose, the matched Chitt, and an authorize button.
Multiple matches: Present the matches with enough context for the user to choose, and let the user select which one to present. Offer "always use this Chitt for Bob's Minecraft Server" as a sticky preference.

The keyring UI should make three things visible and unmissable: who is asking, what they want the credential for, and which specific Chitt will be revealed if the user proceeds.

User Authorization and Signing
The user reviews the prompt and authorizes. The keyring then:

Generates the response payload. Constructs a JSON object containing the challenge, the chosen Chitt's mutable pointer, the session ID, the requester Chitt pointer, and a timestamp. Including the challenge ties the signature to this specific authentication attempt; including the requester Chitt pointer ties the signature to this specific service.

json   {
     "auth_response": {
       "challenge": "<echoed nonce>",
       "session_id": "<echoed session_id>",
       "requester_chitt": "<mutable pointer of Bob's Minecraft Server>",
       "presented_chitt": "<mutable pointer of student's master Chitt>",
       "timestamp": "2026-05-15T18:39:12Z"
     }
   }

Signs with the device sub-Chitt. The keyring signs the canonical serialization of the response payload using the current device's sub-Chitt private key. The master Chitt key stays in the encrypted keyring. The signature uses a sub-Chitt of the presented master Chitt, not the master key itself.
Assembles the full response envelope. The signed payload, the signature, and the signer sub-Chitt pointer are bundled together:

json   {
     "payload": { ... auth_response ... },
     "signatures": [
       {
         "signer_chitt": "<Arbitrum One registry address of student's active sub-Chitt>",
         "public_key": "<signer's ML-DSA-44 public key>",
         "signature": "<sig>"
       }
     ]
   }

Delivers the response. Posts the response envelope to the callback URL specified in the request.

Server Verification
The server receives the response at its callback endpoint and runs the following verification stages:

Challenge freshness. Look up the session by session_id. Confirm the challenge in the response matches the challenge the server issued for that session, that the request hasn't expired, and that this challenge hasn't already been consumed. Replay protection lives here.
Signature validity. Take the public key from the signature entry and verify the signature against the canonical serialization of the response payload. This check requires no network call. Resolve the signer sub-Chitt's Arbitrum One registry address only if freshness of the key needs to be confirmed (e.g., checking that the sub-Chitt hasn't been rotated since the key was embedded).
Sub-Chitt to master link. Resolve the presented_chitt's mutable pointer to get the master Chitt's current metadata. Confirm the signing sub-Chitt's pointer appears in the master Chitt's active sub-Chitt list, and verify the master Chitt's signature on that sub-Chitt registration. A sub-Chitt that has been deregistered (lost device, key rotation) cannot authenticate even if its private key is intact.
Master Chitt chain walk. Walk the master Chitt's issuance chain link by link via mutable pointers. At each link: fetch the metadata, verify the issuer's signature, check that scope at this link doesn't exceed the issuer's scope, check the append-only log for revocations, and continue upward. Walk continues until reaching a Chitt named by the policy as a trusted root, or reject.
Policy match. Evaluate the policy's match predicate against the presented chain. Does this Chitt derive from the required template? Does its scope satisfy any constraints? If any predicate fails, reject.
Annotation lookup. Optionally, query EAS/IPFS for third-party annotations on Chitts in the chain, filtered by trusted annotator Chitts. Annotations might surface safety flags or endorsements the server can act on beyond bare chain validity.

The verifier returns a structured result with the facts of each stage, and the server's policy layer decides how to act on them.

Account Binding
If verification passes, the server binds the session to an account.

Account ID = master Chitt mutable pointer. Not the sub-Chitt. Not a version CID. The mutable pointer is the durable identifier and remains stable across all annotations, updates, and sub-Chitt rotations.

This matters because sub-Chitts change (new devices, key rotations) and the Chitt's log accumulates new entries over time — all while the mutable pointer stays constant. Using the pointer as the account key means none of these normal lifecycle events disrupt account continuity.

The account record on the server side stores:

The master Chitt mutable pointer (primary key)
The Nym gateway address from the Chitt's current metadata, for server-to-user messaging
Any service-specific account data (Minecraft username, in-game preferences, friend lists)
A cached snapshot of the chain at last successful auth, for change-detection purposes
The timestamp of the last successful auth

The server does not store the sub-Chitt pointer as part of the account identity — that's authentication state, not account state.

Re-Authentication
On a subsequent login, the same flow runs. The user receives a new challenge, presents a Chitt matching the same policy, signs with whatever sub-Chitt they currently have. The server verifies the chain afresh — including a fresh revocation check via the append-only log — and on success, recognizes the mutable pointer as the existing account.

A few cases worth being explicit about:
Same Chitt, different sub-Chitt. Normal case. Phone got replaced, sub-Chitt rotated. Master Chitt pointer unchanged. Account found by pointer, login succeeds.
Master Chitt revoked since last login. The chain walk finds a revocation entry in the log. The server rejects the login. Policy choice on further action.
Master Chitt's chain has changed. The school issued a new student Chitt to the same person under a new master keypair. The mutable pointer is different (it's a new Chitt), so by default this looks like a different account. If the service wants to support continuity across reissuance, it needs an explicit account-linking flow where the user authenticates with both the old and new Chitts in sequence. This is a deliberate operation, not something that should happen silently.
Same person, different Chitt type. Authenticating with a tutor Chitt instead of a student Chitt. Different mutable pointer, different account. Multi-Chitt account support is a product decision the service makes explicitly.

Server-to-User Messaging
Once authenticated, the server holds the user's Chitt Nym gateway address. When the server wants to send a message — a ban notification, a server announcement, a notification that someone wants to friend them — it encrypts the message to the user's master public key and routes it through Nym to the gateway. The message arrives at the user's message server, is re-encrypted via UMBRAL to the user's active sub-Chitts, and is queued for device pickup.

The server never learned the user's email. Never learned their phone number. Never learned anything beyond "holds a Chitt deriving from the middle school's student template." But the server can still reliably reach them, and the user can verify those messages came from the server they registered with by checking the server's signature against the requester Chitt they consented to originally.

Sessions
After successful authentication, the server issues a conventional session token. The session token is short-lived; the Chitt-authentication flow is what creates and renews it.

Session expiration: Cadence depends on the service. A Minecraft server might keep sessions alive for weeks; a higher-stakes service might require re-auth every few hours.
Login-time revocation checks: On each authentication, the server re-walks the chain and re-checks the append-only log.
Mid-session revocation: For services where mid-session revocation matters, the server can re-check periodically during long sessions, or subscribe to revocation notifications via Nym.
Logout: Logout invalidates the session token. It does not revoke or modify the Chitt itself.

Unlinkability Considerations
The Chitt-as-account-identifier pattern is pseudonymous, not anonymous. A student who uses the same school Chitt on multiple services is identifiable as the same person across those services if operators collude — they all see the same mutable pointer.

For use cases where stronger unlinkability across services matters, the user should obtain separate Chitts per service from the same template. Open-recipient template policies with rate-limiting support this: the same student gets a fresh Chitt for each service, all chaining to the same school authority, but with independent mutable pointers that cannot be correlated without breaking the chain separately.

Failure Modes and Their Handling
User has no matching Chitt. Keyring reports no matches. User cannot proceed. If the policy publishes issuance guidance, surface it.
User declines to authorize. Keyring returns nothing. Standard auth-cancelled UX.
Request signature invalid. Keyring refuses to proceed and warns the user. Likely a phishing attempt.
Response signature invalid. Server rejects.
Chain doesn't reach the policy's trusted root. Server rejects.
Chain reaches the trusted root but the Chitt has been revoked. Server rejects (revocation entry found in log).
Sub-Chitt no longer registered. Server rejects. User should authenticate from a current device.
Challenge replay. Server rejects.
Stale revocation data. Policy choice. Conservative default: reject.

What the npm Package Exposes
javascript// Server side — generating and verifying auth requests
ChittAuth.createRequest({
  requesterChitt, policyCid, purpose, callback, sessionId
})
ChittAuth.verifyResponse(request, response, policy)

// Server side — session and account management
ChittAuth.bindSession(masterChittPointer, sessionData)
ChittAuth.lookupAccount(masterChittPointer)
ChittAuth.notifyUser(masterChittPointer, message, signingChitt)

// Client side — keyring integration
ChittAuth.parseRequest(deepLinkOrQrPayload)
ChittAuth.findMatchingChitts(request, localKeyring)
ChittAuth.signResponse(request, chosenChitt, subChittKey)
ChittAuth.deliverResponse(request, signedResponse)

Summary
Chitt authentication provides community-credentialed sign-in flows. The user proves they hold a Chitt matching a policy; the server verifies and binds the resulting session to the master Chitt's mutable pointer; subsequent communication flows through the Chitt's Nym address. The pattern composes with everything else in the protocol — the same chain-verification logic, the same revocation model, the same two-tier key architecture, the same metadata-private messaging substrate.
