---
title: "WhatsApp Deep Dive — Multi-Device Sync"
date: 2026-04-27
updated: 2026-04-27
tags: [system-design, case-study, whatsapp, deep-dive, multi-device, sync]
---

# WhatsApp Deep Dive — Multi-Device Sync

**Date:** 2026-04-27 | **Updated:** 2026-04-27
**Tags:** `system-design` `case-study` `whatsapp` `deep-dive` `multi-device` `sync`

## Table of Contents

- [Summary](#summary)
- [Overview](#overview)
- [Historical Constraint — One Device Per Account](#historical-constraint--one-device-per-account)
- [Companion-Device Model — WhatsApp's Choice](#companion-device-model--whatsapps-choice)
- [Peer-Device Model — iMessage and Signal](#peer-device-model--imessage-and-signal)
- [Device List Management — Sesame Protocol](#device-list-management--sesame-protocol)
- [Per-Device Keys vs Shared Key](#per-device-keys-vs-shared-key)
- [State Sync Protocol — Read State Without Plaintext](#state-sync-protocol--read-state-without-plaintext)
- [Offline Edits and Conflicts](#offline-edits-and-conflicts)
- [Login on a New Device — QR Pairing](#login-on-a-new-device--qr-pairing)
- [Key Transparency](#key-transparency)
- [Recovery — iCloud Keychain and Encrypted Backups](#recovery--icloud-keychain-and-encrypted-backups)
- [Bandwidth Cost — N-Device Fanout](#bandwidth-cost--n-device-fanout)
- [iOS vs Android Multi-Device Constraints](#ios-vs-android-multi-device-constraints)
- [Anti-Patterns](#anti-patterns)
- [Related](#related)
- [References](#references)

## Summary

Multi-device sync is the hardest problem in a strict end-to-end encrypted messenger because the server — the one component that has a global view of every user's traffic — is also the one component that must never see plaintext. WhatsApp launched its multi-device feature in 2021 after eight years of phone-only operation, and the architecture they shipped is shaped by that legacy: a **companion-device model** where the phone is the trust anchor and web/desktop are subordinate companions, each with its own Signal Protocol session, capped at four linked devices. Senders encrypt every message N times — once per recipient device — and the server fans out opaque ciphertext. Cross-device state (read markers, archive, mute) is synchronized via encrypted control envelopes that piggyback the same per-device session machinery. The Signal Protocol's [Sesame specification](https://signal.org/docs/specifications/sesame/) is the underlying device-list management layer for both WhatsApp and Signal; iMessage uses a structurally similar but separate design. Every choice in this space is a trade between sender CPU/bandwidth, server statelessness, recovery ergonomics, and the cryptographic invariant that the server learns nothing about content.

## Overview

A multi-device messenger needs to answer four questions for every conversation:

1. **Who are the recipient's currently-trusted devices?** (device list)
2. **What key do I encrypt to for each one?** (per-device session)
3. **How does state — read markers, edits, deletions — propagate across the user's own devices without leaking?** (state sync)
4. **What happens when devices are added, removed, lost, or restored?** (lifecycle)

The naive answer to (1) — "the server tells me" — is also the dangerous one. If the server can lie about which devices belong to a user, it can silently add a wiretap device to the recipient's list and the sender's client will dutifully encrypt to it. This is the **device-list trust** problem, and it is the multi-device equivalent of the public-key distribution problem that vanilla E2EE already had to solve. Sesame, key transparency logs, and out-of-band safety numbers are the layered defenses.

Above that crypto layer sits a **state sync** layer that has its own engineering nightmares: ordering guarantees across devices that come and go, conflict resolution when two devices act offline, and back-pressure when one device is on cellular while another is on WiFi. WhatsApp's design pushes most of this complexity onto the sending client and leaves the server as a routing layer that sees only opaque envelopes plus their per-device addressing.

## Historical Constraint — One Device Per Account

WhatsApp was launched in 2009 as a phone-only application. The account identity was the **phone number**, the cryptographic identity was a key tied to that phone, and the chat database was a single SQLite file on the device. WhatsApp Web, introduced in 2015, was not actually a second device — it was a remote screen for the phone. The desktop browser opened a session that proxied messages through the phone, and if the phone was off or out of battery, the web client stopped working entirely.

Why this lasted so long:

- **One identity key, one device, no fanout problem.** The Signal Protocol session was 1:1 with the recipient's phone, and ciphertext was a single payload to a single endpoint.
- **Plaintext store on a single device** simplified search, sync, and the chat database. iCloud/Drive backups were either unencrypted (default) or encrypted with a user-set password the company itself didn't hold.
- **No state-sync problem.** "Read on phone" and "read on this account" were tautologies.

The cost was user-hostile: a dead phone meant no messaging, web required tethering to the phone's data plan in a sense (and to its battery), and no true desktop client could exist. iMessage (Apple, 2011) and Signal (originally TextSecure, 2010 — multi-device added later) had different starting positions but eventually all converged on multi-device — which is why understanding the historical constraint matters: WhatsApp's architecture is *retrofitted* around a phone-as-anchor assumption, not designed for symmetric peers.

In **2021**, WhatsApp announced [multi-device support](https://engineering.fb.com/2021/07/14/security/whatsapp-multi-device/) with a beta that let users link up to four companion devices (web, desktop, additional phones via secondary install) without requiring the phone to stay online. The redesign required rebuilding sync, message history, group encryption, and device authentication.

## Companion-Device Model — WhatsApp's Choice

WhatsApp picked an asymmetric architecture:

- The **primary device** is the phone tied to the registered phone number. It owns the long-term identity and is the only device that can authorize new device links.
- **Companion devices** (WhatsApp Web, WhatsApp Desktop, additional linked devices) are subordinate. They each have their own Signal Protocol identity key, their own session keys, but they are *authorized into the account* by the primary phone.

This design has explicit consequences:

| Property | Companion-device model |
|---|---|
| Trust anchor | Single primary device (phone) |
| New device authorization | Primary must scan QR / approve |
| Primary loss | Account recovery effectively destroys companions; they must re-link |
| Identity key per device | Yes (each device generates its own) |
| Phone always online required | **No** (post-2021); companions sync via server-relayed encrypted envelopes |
| Phone can revoke companion | Yes, instantly |

The model maps cleanly onto a real human's intuition — "this phone is *me*, those other things are *places I happen to be logged in*" — and lets the company keep account recovery anchored to the phone-number SMS verification flow that already existed. It also caps blast radius: if a companion is compromised, the attacker can read messages going forward but cannot impersonate the user to add additional devices.

The trade-off is asymmetric availability. You cannot use WhatsApp from a desktop without ever having owned a phone. The phone is the single point of bootstrapping.

## Peer-Device Model — iMessage and Signal

iMessage and modern Signal use a more symmetric design where every device an account has — iPhone, iPad, Mac, Apple Watch — is a **peer** with its own identity key, all linked at the account level (Apple ID for iMessage, Signal account for Signal).

Differences from WhatsApp's model:

- **No single trust anchor.** Any provisioned device can theoretically participate in adding new devices (subject to additional security gates like password or biometric).
- **Loss of any one device does not orphan the others.** You can remove a stolen iPhone from `iCloud.com` while still holding the iPad.
- **More attack surface for device-list manipulation.** Without a single primary, the server's claim of "these are the user's devices" carries more weight, which is why Apple's [Contact Key Verification](https://support.apple.com/en-us/HT213465) and Signal's safety numbers exist.

Signal's [multi-device design](https://signal.org/blog/sealed-sender/) actually started closer to WhatsApp's primary/companion model (a phone-anchored linked-device design) and gradually moved toward greater peer symmetry, particularly with the Signal Desktop and standalone iPad clients. iMessage was always peer-symmetric because Apple controls the Apple ID account layer and the device attestation chain.

The two models converge on the same hard cryptographic problem — **how do you authenticate a device list?** — and diverge on **who is allowed to mutate it**.

## Device List Management — Sesame Protocol

The [Sesame specification](https://signal.org/docs/specifications/sesame/) is Signal's name for the per-recipient, per-device session management layer. It defines:

- **Recipient addressing** — each user has an identifier (e.g., phone number, Signal UUID, WhatsApp account ID) and each user has a set of **device IDs**.
- **Per-(user, device) session state** — the sender keeps a Double Ratchet session per recipient device, indexed by `(recipient_id, device_id)`.
- **Session establishment per device** — when a new device appears in the recipient's list, the sender fetches that device's prekey bundle from the server and runs X3DH against it.
- **Session teardown per device** — when a device leaves, sessions are evicted.

Sesame is *not* a key-distribution authority. It assumes the device list is delivered correctly (out of scope for the spec). In practice the server is the device-list source, and clients defend against a malicious server with:

1. **Out-of-band verification** of the *primary* identity (safety numbers / security codes).
2. **Notification on device-list change** so the user sees "Bob added a new device" in the chat.
3. **Optional refusal to send** until the user re-confirms safety numbers after a change.
4. **Key transparency logs** (newer; see below).

```python
# Pseudocode: device-list update handling on the sender side
def on_recipient_device_list_change(recipient_id, new_list, old_list):
    added   = set(new_list) - set(old_list)
    removed = set(old_list) - set(new_list)

    for device_id in removed:
        sessions.evict((recipient_id, device_id))

    for device_id in added:
        bundle = server.fetch_prekey_bundle(recipient_id, device_id)
        # X3DH against the new device's prekeys
        session = x3dh_initiate(bundle)
        sessions.put((recipient_id, device_id), session)

        ui.notify_device_added(recipient_id, device_id)  # security UX

    # Surface a "safety number changed" affordance so the user can re-verify
    if added or removed:
        ui.flag_safety_number_changed(recipient_id)
```

The cryptographic envelope changes; the Double Ratchet sessions get rebuilt; the user gets a UI signal. Sesame names and structures all of this, but the security-critical decisions live in the application layer and the user's willingness to actually inspect that "safety number changed" prompt.

## Per-Device Keys vs Shared Key

There are two conceivable designs for multi-device E2EE:

### Option A: Shared Account Key (rejected by WhatsApp/Signal/iMessage)

Every device holds the same private key. Sender encrypts once, every device decrypts.

- **Pro:** Sender CPU and bandwidth are O(1) regardless of recipient device count.
- **Con:** Provisioning a new device requires moving the private key to it. This either requires:
  - The phone to upload the key to the server (no — server learns it).
  - Direct device-to-device transfer for every new device (operationally hostile).
  - A passphrase-derived KEK with cloud upload (relies on user passphrase strength; bad track record).
- **Con:** A compromised device leaks the account key for all past and future messages on all devices.
- **Con:** Removing a device requires rotating the entire account key and re-provisioning all remaining devices.

### Option B: Per-Device Keys (the chosen design)

Each device generates its own identity key and prekey bundles locally. The private keys never leave the device.

- **Pro:** Compromising one device compromises only its own future traffic; past sessions on other devices remain forward-secret.
- **Pro:** Removing a device is a list-mutation operation; remaining devices are unaffected.
- **Pro:** New devices can be provisioned without ever copying long-term private material.
- **Con:** Sender encrypts N times — one ciphertext per recipient device. Bandwidth and CPU scale linearly with device count.
- **Con:** Server stores per-device prekey bundles and per-device queues.
- **Con:** Group conversations get expensive — a 10-person group with 4 devices each is 40 ciphertexts per message (hence Sender Keys; see [group-chat-fanout.md](group-chat-fanout.md)).

Per-device keys won because their failure mode (CPU/bandwidth cost) is engineering-tractable, while shared-key's failure mode (key extraction across devices) is cryptographically catastrophic. The 4-device cap on WhatsApp companions is a direct consequence: it bounds the worst-case fanout.

```python
# Pseudocode: per-device fanout on send
def send_to_user(plaintext, recipient_id):
    device_ids = device_list_for(recipient_id)  # from server, optionally TOFU-cached
    envelopes = []

    for device_id in device_ids:
        session = sessions.get_or_init((recipient_id, device_id))
        ciphertext = double_ratchet_encrypt(session, plaintext)
        envelopes.append({
            "to_user":   recipient_id,
            "to_device": device_id,
            "ciphertext": ciphertext,  # opaque to server
        })

    # Server fans out to per-device sockets / queues.
    server.deliver_batch(envelopes)
```

Note that the *plaintext is encrypted N times*, not encrypted once and wrapped with N keys. The Double Ratchet session is per-device, and ratchet state advances independently per session. This is what gives you forward secrecy and post-compromise security per device but also what makes the bandwidth cost real.

## State Sync Protocol — Read State Without Plaintext

When Alice has phone + desktop and reads a message on the desktop, her phone needs to mark it as read so the unread badge clears. The server cannot see plaintext, so the read marker cannot be a server-mediated state machine over message content. The pattern WhatsApp and Signal use:

1. **Read state is itself a message.** When Alice's desktop marks message `m` as read, it sends an encrypted control envelope to her own other devices: `{ type: READ, message_id: m, timestamp: t }`.
2. **The envelope is encrypted to each of Alice's own companion devices.** The phone receives the same kind of opaque ciphertext it would receive from a contact, decrypts it, and updates its local read state.
3. **The server routes by device address** — the same multi-device fanout machinery — and never sees the read marker semantics.

This generalizes: archive, mute, pin, delete-for-me, message-star, and edit-state all flow through the same encrypted-control-envelope channel. The server is a stateless courier.

```python
# Pseudocode: cross-device read receipt fanout (Alice's own devices only)
def mark_read_locally(message_id):
    state.set_read(message_id)
    own_other_devices = device_list_for(self.user_id) - {self.device_id}

    payload = {
        "type": "READ",
        "message_id": message_id,
        "read_at": now(),
        "device": self.device_id,
    }

    for device_id in own_other_devices:
        session = sessions.get_or_init((self.user_id, device_id))
        ciphertext = double_ratchet_encrypt(session, json_encode(payload))
        server.deliver({
            "to_user": self.user_id,
            "to_device": device_id,
            "ciphertext": ciphertext,
            "envelope_type": "device_sync",   # routing hint, not content
        })
```

The server sees `envelope_type=device_sync`, which lets it route appropriately (and lets it apply different push-notification policy — you don't notify the user about their own sync envelopes). It does not see what the envelope says.

For richer state — a full chat-level read marker that says "everything up to and including message X is read" — the same channel carries the watermark. Receiving devices apply the highest watermark seen.

> See also [read-receipts-and-typing.md](read-receipts-and-typing.md) for the per-conversation read-receipt path between *different* users (the contract that the chat ticks turn blue).

## Offline Edits and Conflicts

When two of Alice's devices are both offline and both make changes — phone deletes a message, desktop pins it, both archive different chats — the server cannot adjudicate because it can't read the changes. Conflict resolution must be deterministic at the receiving device.

Common approaches:

### Last-Writer-Wins (LWW)

Each operation carries a timestamp from the originating device. When devices reconcile, the highest timestamp wins. Cheap, simple, susceptible to clock skew across devices and to lost intent (a "delete" with an older timestamp loses to a stale "pin").

WhatsApp uses LWW for most chat metadata where the user-visible cost of a wrong merge is low (a pin that briefly comes back, an archive that bounces). It does not use LWW for message content (messages are immutable post-send except for explicit edit/delete operations, which themselves are LWW-ordered with a special tombstone).

### Vector Clocks

Each device maintains a per-device counter. Operations carry the vector. Causal ordering is detectable; concurrent ops surface to the application as conflicts. More expressive than LWW; more metadata per op. Used in academic CRDTs and some Riak deployments — mostly avoided in messaging because the user does not want a "merge conflict" UI.

### CRDTs (Conflict-Free Replicated Data Types)

Data types designed to merge automatically and commutatively. The classic CRDT for read state is an **OR-Set** (observed-removed set) or a **2P-Set** (two-phase set) — for unread message IDs, you want monotone "this id is read" with no resurrection on offline merges, which is essentially a **G-Set** (grow-only set):

```python
# Pseudocode: G-Set CRDT for read message IDs
class ReadSet:
    def __init__(self):
        self.read_ids = set()

    def mark_read(self, message_id):
        self.read_ids.add(message_id)

    def merge(self, other):
        # Commutative, associative, idempotent — no conflict possible
        self.read_ids = self.read_ids | other.read_ids

    def is_read(self, message_id):
        return message_id in self.read_ids
```

Once a message is marked read on any device, the union semantics guarantee it stays read everywhere after merge, regardless of order. This is exactly the semantics users expect — once you've seen a message, it shouldn't reappear as unread because the phone synced last.

For richer state with retraction (mute on/off, pin/unpin), an **LWW-Element-Set** with per-element timestamps gets you commutativity at the cost of accepting LWW's clock-skew risk. WhatsApp and Signal use a mix: G-Sets where retraction isn't allowed, LWW-Element-Sets where it is, and explicit conflict UI for the rare cases where a deterministic merge would be wrong (e.g., different edits to the same message — last edit wins, but the edit history is preserved).

Deeper treatment of merge semantics in [../../../data-consistency/quorum-and-tunable-consistency.md](../../../data-consistency/quorum-and-tunable-consistency.md), specifically the LWW vs vector-clock discussion.

## Login on a New Device — QR Pairing

The QR-code pairing flow is the most security-sensitive UX in the entire system because it is the moment a new device is granted account-level trust. The flow:

1. The new device (web, desktop) generates a fresh keypair and displays a QR code containing its public key plus a session nonce.
2. The primary phone's WhatsApp app, in "Linked Devices → Link a Device", opens the camera and scans the QR.
3. The phone now has the new device's public key. It validates the nonce, and:
   - Authenticates the new device via the user (biometric or PIN on the phone).
   - Sends the new device's public key + device descriptor to the server, signed by the phone's identity key, as an authorization to add this device to the account.
   - Sends initial state to the new device — recent chat history, contact list, group membership — encrypted to the new device's key.
4. The server adds the device to the account's device list and starts publishing prekey bundles for it.
5. Other contacts pulling the device list see "Alice added a new device" notifications and (depending on settings) a safety-number-changed prompt.

```python
# Pseudocode: QR pairing on the new device
def pair_with_primary():
    keypair = generate_identity_keypair()
    nonce   = secure_random(32)
    payload = {
        "device_pub":  keypair.public,
        "nonce":       nonce,
        "device_type": "desktop",
        "expires_at":  now() + 60,  # short-lived
    }
    qr_data = base64(serialize(payload))
    display_qr(qr_data)

    # Wait for primary to authorize via the server
    auth = server.wait_for_authorization(nonce, timeout=120)
    if not auth:
        raise PairingTimeout()

    # auth contains: signed device-list update, initial state ciphertext
    verify_signature(auth, primary_identity_pub=server.fetch_primary_pub(self.account_id))
    install_initial_state(auth.initial_state, decrypt_with=keypair.private)
    sessions.bootstrap_from(auth.initial_state.sessions)
    return Ok
```

Two security properties matter:

- **The QR contains only the new device's public material** — never the private key, never the account, never anything sensitive. A compromised QR (someone photographs your screen) lets the attacker know *what device is trying to pair*, not impersonate it.
- **The phone is the only entity that can authorize.** The server can refuse to add a device but cannot add one on its own (assuming key transparency or out-of-band verification is in place — see below).

The 60-second QR expiry is a phishing mitigation: an attacker can't pre-generate a QR and trick the user into scanning it later. The session nonce binds the authorization to *this* pairing attempt.

## Key Transparency

The deepest weakness in any E2EE system is the **public-key distribution channel**. If the server can lie to Alice about Bob's public key, it can man-in-the-middle every message. Multi-device makes this worse: the server now distributes a *list* of keys per user, and adding a single rogue key to that list is enough to silently wiretap every conversation.

Defenses:

### Out-of-Band Safety Numbers (the original answer)

Each pair of users has a deterministic fingerprint derived from their identity public keys. They can compare it in person, on a phone call, or via QR. If the fingerprint matches what both clients show, the keys are confirmed.

- **Strength**: cryptographically sound; no trust in third parties.
- **Weakness**: nobody actually does this. Real-world adoption is < 1%.

### Auditable Logs (the modern answer)

The server publishes a log of every (user, device, key) tuple in an append-only structure (typically a Merkle tree). Every client periodically audits its own entries to make sure the server isn't showing different keys to different users.

Two production systems implement this:

- **Apple's [Contact Key Verification](https://support.apple.com/en-us/HT213465)** — uses a key transparency service that publishes signed proofs for every iMessage account's device-key set. Clients fetch inclusion proofs and verify the log is consistent across queries.
- **Signal's [contact-discovery and key-transparency work](https://signal.org/blog/building-faster-oram/)** — earlier prototypes; production rollout is still partial as of 2025.

Key transparency does not prevent a one-time MITM — the server can show Bob a malicious key for one query — but it makes *persistent, undetected* MITM impossible because the audit log will contain the malicious key, and Bob's own clients will eventually see it and alarm.

```python
# Pseudocode: client-side key transparency audit
def audit_my_own_devices():
    expected_devices = local_device_list()
    log_entry        = key_transparency.fetch_inclusion_proof(self.user_id)
    log_devices      = log_entry.devices

    if set(log_devices) != set(expected_devices):
        # The server is publishing a different device list than I authorized
        alarm("KEY_TRANSPARENCY_MISMATCH",
              expected=expected_devices,
              published=log_devices)
        refuse_to_communicate_until_resolved()

    # Also verify the log itself is consistent with the previous audit
    key_transparency.verify_append_only_consistency()
```

WhatsApp does not, as of public documentation, ship full key transparency. It relies on safety numbers, on the user-visible "this device was added" prompts, and on the assumption that adding a wiretap device is a noisy operation that the legitimate user will notice. This is a real weakness compared to iMessage's Contact Key Verification or Signal's stated direction.

## Recovery — iCloud Keychain and Encrypted Backups

When the primary phone is lost, stolen, or destroyed, the recovery story has to thread three needles:

1. The user must regain access to their account.
2. The server must not gain plaintext access during the process.
3. Existing companion devices must be revoked or re-authenticated, because the new phone is a new trust anchor.

The standard approaches:

### Phone-Number Re-Verification

WhatsApp's primary recovery mechanism. The user installs WhatsApp on a new phone, verifies the same phone number via SMS, and the new device becomes the primary. Existing companions are **automatically logged out** because their device-list authorization, signed by the old primary's identity key, is no longer valid against the new identity key.

This is brutally simple and brutally effective: it ties account recovery to control of the phone number, which has its own SIM-swap attack surface but doesn't introduce a new one.

### iCloud Keychain / Google Account Device-Bound Keys

For peer-symmetric systems (iMessage, partial in newer WhatsApp), device identity material can be bound to the user's iCloud Keychain or Google account. When the user signs in on a new device, the keychain syncs the identity material into it — but only after a [Secure Enclave] / [Titan-M] backed authentication that the cloud provider itself cannot bypass.

- **Pro:** Recovery without losing message history; the new device can re-establish sessions seamlessly.
- **Con:** The cloud account is now part of the trust base. iCloud account compromise → device material compromise.
- **Mitigation:** Apple's [Advanced Data Protection](https://support.apple.com/en-us/HT212520) extends end-to-end encryption to keychain itself, removing Apple from the trust path at the cost of removing recovery-via-Apple-Support.

### Encrypted Backups (WhatsApp's chat backup)

The chat history itself can be backed up, encrypted with a user-controlled key (a 64-digit passphrase or, since 2021, an end-to-end encrypted backup with a [Hardware Security Module](https://engineering.fb.com/2021/09/10/security/whatsapp-end-to-end-encrypted-backups/) holding the key behind a user PIN with brute-force resistance). A new device can restore from backup before re-establishing sessions with contacts.

- **Pro:** Preserves chat history across device migration.
- **Con:** Backup is *not* the same as session state — sessions still need to be re-established, which means *new* messages will go through new ratchets, but old messages can be decrypted from the backup.
- **Con:** A weak backup PIN (4 digits) is a real attack surface; the HSM rate-limits but can't fix the core entropy problem.

The recovery story shows the structural difference between the companion-device model (recovery = phone-number re-verification, companions get re-linked) and the peer-device model (recovery = sign in on any peer, all peers stay valid). WhatsApp accepted simpler recovery semantics in exchange for slightly worse UX on phone loss.

## Bandwidth Cost — N-Device Fanout

The bandwidth math for per-device fanout:

```
sender_upload_bytes_per_message ≈ N_recipient_devices × ciphertext_size + envelope_overhead
```

For a 1:1 chat between Alice (4 devices: phone + 2 desktops + 1 web) and Bob (also 4 devices), every message Alice sends becomes:

- 4 ciphertexts to Bob's devices.
- 3 ciphertexts to her *own other* devices (own-device sync).
- = **7 ciphertexts** uploaded for one logical message.

For a 100-person group with 4 devices each, the naive fanout is 400 ciphertexts. This is the failure mode that motivated **Sender Keys** ([group-chat-fanout.md](group-chat-fanout.md)) — distribute a symmetric chain key once via pairwise sessions, then encrypt subsequent messages just once with the symmetric key. The first message in a session is still O(N_devices), but steady-state is O(1).

Mitigations the WhatsApp client uses:

| Mitigation | Effect |
|---|---|
| **Hard cap of 4 companion devices** | Bounds N at small constant |
| **Sender Keys for groups** | First-message cost only |
| **Lazy session establishment** | Don't pre-build sessions for inactive devices |
| **Compressed media references** | Send the encrypted blob to CDN once, send only a key + URL per device |
| **Ratchet pre-keys batched** | Reduce round trips on session init |
| **Server-side device-set deduplication** | Skip devices the server knows are zombie/uninstalled |

The media trick is important: a 5 MB image is *not* sent N times. The image is encrypted once with a random symmetric key, uploaded to a CDN, and only the (key, URL, hash) tuple is sent per device — kilobytes, not megabytes. The big-payload fanout is media-aware in a way the per-message control flow cannot be.

## iOS vs Android Multi-Device Constraints

Mobile background-execution policy shapes what "sync" means in practice.

### iOS

- **No long-lived TCP sockets in background.** When the phone is locked or backgrounded, the WhatsApp process is suspended. Pushes go through APNs.
- **Push payload is small (4 KB).** Useful for a "wake up, fetch messages" signal but not for full sync.
- **Background fetch is coarse-grained and OS-controlled.** The OS decides when WhatsApp can run.
- **Consequence:** the phone, when it is the primary trust anchor, lags behind on cross-device sync envelopes when locked. Companion devices may receive and display messages before the phone catches up.

### Android

- **More flexible background services**, though increasingly restricted in modern Android (Doze, App Standby).
- **FCM (Firebase Cloud Messaging) push** with broadly similar payload-size constraints.
- **Foreground service for active calls / chats** can hold sockets longer.
- **Consequence:** Android phones tend to stay more current with sync envelopes, but at the cost of battery — which Android users notice and which the app must balance.

### Implications for sync design

- **No assumption of phone availability.** Post-2021 WhatsApp explicitly does not require the phone to be online for companions to work.
- **Sync envelopes queue server-side.** The server holds an encrypted queue per (user, device) and delivers when the device reconnects. Same mechanism as offline message delivery (see [offline-delivery-and-push.md](offline-delivery-and-push.md)).
- **Read-receipt eventual consistency.** A read on the desktop may take seconds to minutes to reflect on the phone if the phone is locked and on cellular. The G-Set merge semantics make this safe — once read, always read after merge.
- **Edits and deletes** propagate via the same channel and inherit the same latency. The "edited" indicator may briefly disagree across devices.

The mobile platform constraints mean the *server's* job is fundamentally about queueing per-device payloads and waking devices up via push, not about pushing real-time sync. The "real-time" experience users feel comes from devices that are foreground-active; everything else is best-effort eventual consistency over an encrypted transport.

## Anti-Patterns

**Trusting the server's device list without UI signal.** A multi-device E2EE system that silently encrypts to whatever device list the server returns is a wiretap waiting to happen. There must be a user-visible event for device-list changes, ideally with a friction step (re-verify safety numbers) before sending continues.

**Conflating "device sync" with "server-mediated state".** Once the design lets the server hold authoritative read state, archive state, or any other user data, the server can lie about it, and forward secrecy on message content stops mattering for everything except message content. State sync must use the same encrypted-envelope channel as messages.

**Using LWW for monotone state.** Read state should never become "unread" because of a clock-skewed merge. Use a G-Set or equivalently a one-way mark. LWW belongs to state with legitimate retraction (mute on/off, pin/unpin, archive/unarchive) and even there it's a known clock-skew hazard.

**Synchronizing entire chat history on every link.** Sending all chat history to a newly-linked desktop, encrypted to the desktop's identity key, is gigabytes of upload from the phone. Better designs send the most-recent N messages on link, lazy-load older history on demand, or rely on direct device-to-device transfer over local network when available. WhatsApp's history-on-link sends a recent window; full history requires a chat backup restore.

**Assuming companion devices revoke cleanly.** When the primary phone is replaced, companions show "logged out" but the *server-side prekey bundles for the old companion devices may persist* until garbage-collected. Senders with stale device lists can encrypt messages to dead devices — those messages are lost. Mitigation: short prekey bundle TTLs, server-side liveness gating, and aggressive client-side device-list refresh on send.

**Treating "QR scanned" as full authentication.** The QR transports the device's public key, but the *user* must still authenticate on the primary (biometric/PIN) before the device-list mutation is signed. Apps that auto-approve QR scans turn a stolen unlocked phone into an account takeover.

**Ignoring rate limits on session re-establishment.** A flapping device that keeps re-linking generates a stream of session-establishment events, each of which forces remote senders to re-do X3DH and cycle ratchets. Servers should rate-limit per-device prekey publication; senders should debounce session re-init on flap.

**Letting media fanout dominate bandwidth.** Re-encrypting and re-uploading a 50 MB video N times is operationally fatal on cellular. The CDN-with-key-fanout pattern is mandatory for any large payload; a multi-device design that doesn't bake this in from day one will be rewritten.

## Related

- [WhatsApp HLD — design-whatsapp.md](../design-whatsapp.md) — parent case study with end-to-end transport, fanout, and edge-gateway design
- [End-to-End Encryption deep dive](end-to-end-encryption.md) _(planned sibling)_ — Signal Protocol, X3DH, Double Ratchet, Sender Keys, Sealed Sender
- [Group Chat Fanout deep dive](group-chat-fanout.md) — Sender Keys and the steady-state group cost that multi-device makes worse
- [Offline Delivery and Push](offline-delivery-and-push.md) — the per-device queue and APNs/FCM wakeup mechanism that powers cross-device sync
- [Read Receipts and Typing](read-receipts-and-typing.md) — the user-to-user read-receipt path that runs on the same envelope channel as own-device sync
- [Quorum Reads/Writes and Tunable Consistency](../../../data-consistency/quorum-and-tunable-consistency.md) — LWW vs vector-clock vs CRDT discussion that informs offline-edit conflict resolution
- [Per-Conversation Ordering](per-conversation-ordering.md) — how messages within a chat get ordered when devices may receive them in different orders

## References

- [WhatsApp Multi-Device Architecture (Meta Engineering, 2021)](https://engineering.fb.com/2021/07/14/security/whatsapp-multi-device/) — official write-up of the companion-device model, per-device session design, and rollout
- [WhatsApp End-to-End Encrypted Backups (Meta Engineering, 2021)](https://engineering.fb.com/2021/09/10/security/whatsapp-end-to-end-encrypted-backups/) — HSM-backed backup-key recovery design
- [Signal Sesame Specification](https://signal.org/docs/specifications/sesame/) — the per-recipient-per-device session management protocol underlying both Signal and WhatsApp multi-device
- [Signal X3DH Specification](https://signal.org/docs/specifications/x3dh/) — the asynchronous key agreement run for each new device
- [Signal Double Ratchet Specification](https://signal.org/docs/specifications/doubleratchet/) — the per-session forward-secret ratchet
- [Apple iMessage Contact Key Verification](https://support.apple.com/en-us/HT213465) — Apple's key transparency rollout for iMessage device lists
- [Apple Platform Security — Messages and FaceTime](https://support.apple.com/guide/security/messages-and-facetime-secd9a0ce71a/web) — official documentation on iMessage's per-device key model and account device list
- [MLS — Messaging Layer Security, RFC 9420 (IETF, 2023)](https://www.rfc-editor.org/rfc/rfc9420.html) — the IETF's standardized successor to Sender Keys, designed for groups with frequent membership changes; relevant for multi-device-as-implicit-group models
- [Sealed Sender (Signal blog, 2018)](https://signal.org/blog/sealed-sender/) — sender-anonymity envelope used in conjunction with multi-device addressing
- [Designing Data-Intensive Applications, Kleppmann — Chapter 5: Replication](https://dataintensive.net/) — CRDT and conflict resolution background applicable to offline-edit merge semantics
