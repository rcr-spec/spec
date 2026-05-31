# RCR/1 Specification

Version: 1.0.0-rc1
Status: IETF Internet-Draft (`draft-snow-rcr-receipt-00`)
License: CC0 1.0 Universal

This document is the human-readable specification for the Receipt Commitment Record, version 1 (RCR/1). The normative artifact is the JSON Schema at `schema/rcr-receipt-v1.json` and the IETF Internet-Draft at `IETF/draft-snow-rcr-receipt-00.txt`.

## 1. Overview and Motivation

AI agents act across many systems. Each system emits evidence of those actions in a different shape: an email provider returns DKIM signatures and SMTP delivery status; a payment processor returns webhook signatures and charge IDs; a code host returns commit SHAs and CI status. None of these formats is comparable to the others, none is canonical, and none binds the receipt to the upstream agent event that caused it.

RCR/1 turns vendor-native receipts into a single canonical record that:

1. Preserves source identity (which vendor, which API, which version).
2. Classifies the action (mail send, payment charge, code commit, document signature).
3. Records the effect status (delivered, accepted, failed, refunded).
4. Carries cryptographic evidence (DKIM, vendor signatures, RFC 3161 timestamps, transparency log indexes).
5. Chains receipts via SHA-256 over RFC 8785 canonical JSON, so any party can detect tampering or gaps.
6. States the credibility of each claim using the strength ladder defined in section 6, so verifiers never overclaim.

RCR/1 is the on-the-wire receipt format. RANKIGI's closure bundle (section 7) is the container that ships RCR/1 receipts together with the chain of agent events that produced them.

## 2. Terminology

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" in this document are to be interpreted as described in RFC 2119 / RFC 8174.

- *Agent event*: an action taken by an AI agent, captured at the moment of execution.
- *World receipt* (or simply *receipt*): evidence emitted by an external system that the action produced a real effect.
- *Canonical event*: an agent event serialized using `rcr-jcs-1` canonicalization.
- *Canonical receipt* (RCR/1 record): a world receipt translated into the canonical format defined in this document.
- *Chain*: an ordered sequence of canonical events linked by `prev_hash`, where each event's hash is computed over its canonical bytes including the previous event's hash.
- *Closure bundle*: a RANKIGI-specific container that ships a closed chain of canonical events plus any associated RCR/1 receipts and external anchors (Rekor entry, RFC 3161 token).
- *Claim strength*: one of the seven levels defined in section 6, describing how independently a verifier can confirm a receipt.

## 3. The Canonical Event Format

A canonical event is the on-the-wire form of an agent action. The hash input is canonical JSON (rcr-jcs-1) over the following object.

```json
{
  "action":               "<string>",
  "agent_id":             "<UUID>",
  "canon_id":             "<string|null>",
  "canon_version":        "<string|null>",
  "chain_id":             "<UUID|null>",
  "occurred_at":          "<ISO 8601 UTC, millisecond precision, e.g. 2026-05-30T17:42:01.123Z>",
  "org_id":               "<UUID>",
  "passport_fingerprint": "<hex|null>",
  "payload":              { ... },
  "prev_hash":            "<hex>",
  "severity":             "<string|null>",
  "signature_hash":       "<hex|null>",
  "tool":                 "<string|null>"
}
```

The `hash_version` field on the carrier object (not part of the hash input) signals which canonical shape was used. Defined versions:

- `1` (legacy): pipe-concatenated input. See the reference verifier for the exact ordering. New producers SHOULD NOT emit hash_version 1.
- `2`: the object above without `canon_id`, `canon_version`, `passport_fingerprint`, `signature_hash`.
- `3`: adds `canon_id`, `canon_version`.
- `4` (current): adds `passport_fingerprint`, `signature_hash`. All five fields are emitted as JSON `null` when absent so the canonical shape is stable across signed and unsigned events.

Verifiers MUST dispatch on `hash_version`. When `hash_version` is missing or null, verifiers MUST default to version 3.

## 4. The World Receipt Format

A canonical RCR/1 receipt is a JSON object with the following top-level structure.

```json
{
  "rcr_version":    "1.0.0-rc1",
  "receipt_id":     "<URI or vendor identifier>",
  "issued_at":      "<ISO 8601 UTC>",
  "source": {
    "vendor":       "<string, e.g. 'stripe' or 'sendgrid'>",
    "product":      "<string, optional>",
    "api_version":  "<string, optional>"
  },
  "action": {
    "class":        "<string, e.g. 'mail.send', 'payment.charge', 'document.signed'>",
    "subject":      "<string or object identifying the artifact>",
    "effect":       "<string: 'attempted'|'accepted'|'delivered'|'failed'|'reversed'>"
  },
  "evidence": {
    "vendor_payload_hash": "<hex sha256 of the raw vendor payload>",
    "vendor_signature":    "<base64 signature, optional>",
    "dkim":                "<DKIM signature header value, optional>",
    "tsa_token":           "<base64 RFC 3161 TimeStampToken, optional>",
    "rekor_log_index":     "<integer, optional>"
  },
  "chain": {
    "previous_receipt_hash": "<hex|null>",
    "agent_event_hash":      "<hex>",
    "binding_proof":         "<hex|null>"
  },
  "claim": {
    "strength":     "<one of the seven levels in section 6>",
    "what_proven":  "<short human-readable statement>",
    "what_not_proven": "<short human-readable statement>"
  },
  "integrity": {
    "canonicalization": "rcr-jcs-1",
    "hash_alg":         "sha256",
    "receipt_hash":     "<hex, see section 8>"
  }
}
```

`integrity.receipt_hash` is computed over the canonical JSON of the entire receipt with the `integrity.receipt_hash` field excluded from the hash input.

## 5. The Binding Mechanism

A canonical receipt binds to an agent event via the `chain.agent_event_hash` field, which MUST equal the hash of a canonical event as defined in section 3.

When the receipt is one of several receipts produced by the same agent event (for example, a single mail send that produces both an SMTP accepted receipt and a delivered receipt), `chain.previous_receipt_hash` MUST point to the prior receipt in receipt-issue order. This forms a per-event receipt chain anchored at the agent event hash.

The optional `chain.binding_proof` field carries an additional cryptographic binding (an HMAC over `agent_event_hash || vendor_payload_hash` using a per-org binding key, or a signed JWS) when the issuing producer wants to attest to the binding itself, not just to the agent event and the vendor payload independently.

## 6. The Claim Strength Ladder

Verifiers MUST report the strongest claim level that an external party can independently confirm. The seven levels, from strongest to weakest:

| Level | Name | What "verified" means at this level |
|------:|------|-------------------------------------|
| 7 | `public_ledger` | The receipt's canonical hash is anchored on a public transparency log entry that any party can fetch and inspect. |
| 6 | `regulator_attested` | A recognized regulator or qualified attestation authority has counter-signed the receipt. |
| 5 | `qualified_seal` | The receipt carries a qualified electronic seal under eIDAS or equivalent. |
| 4 | `notarized` | The receipt carries an RFC 3161 timestamp whose certificate chain validates back to a recognized TSA root. |
| 3 | `vendor_signed` | The receipt is signed by the issuing vendor's private key (DKIM, JWS, X.509). |
| 2 | `vendor_lookup_token` | The receipt carries a vendor identifier that any party can look up against the vendor's public API. |
| 1 | `customer_asserted` | The receipt is self-reported by the agent or operator. No third party witness. |

A verifier MUST NOT report a strength higher than what it has actually confirmed. A verifier MAY report a lower strength than the receipt claims when the higher claim cannot be confirmed offline or with available keys.

## 7. The Closure Bundle Format

A closure bundle (RANKIGI bundle schema version 1.5) is a JSON container that ships a chain of canonical events plus the RCR/1 receipts and external anchors associated with that chain. Its top-level shape:

```json
{
  "rankigi_proof_bundle_schema": "1.5",
  "exported_at_utc":             "<ISO 8601>",
  "receipt6":                    "<6 hex chars>",
  "envelope": {
    "passport_key_fingerprint":  "<hex>",
    "passport_public_key_pem":   "<PEM-wrapped Ed25519 SPKI>"
  },
  "events": [
    {
      "action":        "<string>",
      "agent_id":      "<UUID>",
      "chain_id":      "<UUID|null>",
      "nonce":         "<string|null>",
      "occurred_at":   "<ISO 8601 UTC>",
      "payload":       { ... },
      "tool":          "<string|null>",
      "signature":     "<base64 Ed25519 signature>",
      "hash":          "<hex>",
      "prev_hash":     "<hex>",
      "chain_index":   "<integer|null>",
      "hash_version":  4
    }
  ],
  "rcr_receipts": [
    { "<RCR/1 receipt object per section 4>" }
  ],
  "rekor_anchor": {
    "log_index":             "<integer>",
    "anchor_payload_hash":   "<hex>"
  },
  "rfc3161_timestamp": {
    "tsa_url":               "<string>",
    "tsr_token_base64":      "<base64>",
    "tsa_cert_chain_pem":    "<PEM chain>",
    "message_imprint_hash":  "<hex>"
  },
  "tsa_tokens": [
    {
      "event_id":              "<hex>",
      "tsa_url":               "<string>",
      "verified_at":           "<ISO 8601>",
      "token_present":         true,
      "tsr_token_base64":      "<base64>",
      "tsa_cert_chain_pem":    "<PEM chain>",
      "message_imprint_hash":  "<hex>"
    }
  ],
  "verify_with": {
    "script":           "https://rankigi.com/verify.py",
    "usage":            "<string>",
    "pubkey_endpoint":  "<URL>"
  }
}
```

Required: `rankigi_proof_bundle_schema`, `events`, `envelope`.
Optional: `rcr_receipts`, `rekor_anchor`, `rfc3161_timestamp`, `tsa_tokens`, `verify_with`.

## 8. Verification Algorithm

A conforming verifier MUST, in order:

1. Parse the bundle JSON. Reject malformed input.
2. For each event:
   a. Recompute the canonical hash per section 3, dispatching on `hash_version`.
   b. Compare the recomputed hash to `event.hash`. Mismatch is a hash chain break.
   c. Confirm `event.prev_hash` matches the prior event's `event.hash` (per chain_id, or per agent_id when chain_id is null).
   d. When `envelope.passport_public_key_pem` is present and `event.signature` is present, verify the Ed25519 signature against the canonical event bytes. Mismatch is a signature break.
3. When `rekor_anchor` is present and the verifier was invoked with network access, fetch the Rekor entry at `log_index` and confirm that its body hash equals `anchor_payload_hash`. Network failure MUST be reported as SKIPPED, not as VERIFIED.
4. When `rfc3161_timestamp` or `tsa_tokens` are present, parse each TSR token, recompute the message imprint, confirm the signer certificate chain validates to a pinned TSA root, and verify the TSA's signature over the signed attributes. Report each leg independently (`MessageImprint`, `TSA signature`, `Cert chain`).
5. For each receipt in `rcr_receipts`:
   a. Recompute `integrity.receipt_hash` per section 8.
   b. Confirm `chain.agent_event_hash` matches an event's `hash` in the bundle.
   c. Validate evidence according to the claimed strength level. Report the strongest level the verifier can confirm.

A verifier MUST emit one of five overall verdicts:

- `VERIFIED`: all events hashed correctly, all signatures valid, all anchors confirmed.
- `SELF_ATTESTED`: structurally valid but no third-party witness available; all receipts are at `customer_asserted` strength.
- `PARTIAL`: some legs verified, some skipped due to missing keys or offline anchors. The output MUST identify which legs.
- `UNVERIFIED`: hash chain or signature broken. Report which event and which leg.
- `COMPROMISED`: a signed receipt's signature failed, or a Rekor anchor mismatched. This is stronger than UNVERIFIED: an active tampering signal.

Receipt hash computation (`integrity.receipt_hash`):

```
canon_input = canonical_json(receipt with integrity.receipt_hash field removed)
receipt_hash = sha256_hex(canon_input)
```

## 9. Security Considerations

- *Trust anchor*: the only out-of-band trust input to RCR/1 verification is the issuer's signing public key. Verifiers SHOULD obtain this key via a published well-known URL with HTTPS pinning, or out of band from a trusted directory.
- *Replay*: the `nonce` field on canonical events and the `prev_hash` chain make replay detectable. Verifiers MUST treat duplicate `(agent_id, chain_id, chain_index)` tuples as a chain break.
- *Encrypted payloads*: an event MAY carry `payload_canonical_encrypted` (AES-256-GCM over the canonical payload bytes, with HKDF-SHA256 over a per-org master key). Verifiers without the master key MUST report such events as PARTIAL with reason "encrypted payload, no key", not as VERIFIED.
- *No raw sensitive data*: producers MUST NOT place credentials, secrets, PHI, or privileged material in the canonical payload. The receipt mechanism is hash-based and bound to evidence; the raw content remains at the source system.
- *Vendor signature forgery*: a `vendor_signed` claim is only as strong as the vendor's published key distribution. Verifiers SHOULD pin vendor keys explicitly when available.
- *Time stamping*: an RFC 3161 token proves existence at or before the TSA's clock, not at or after the agent action. Producers SHOULD timestamp shortly after the action.
- *Public ledger anchoring*: a Rekor entry proves the hash existed at a Rekor inclusion time. It does not prove the receipt is true; it proves the receipt content has not been changed since anchoring.

## 10. IANA Considerations

This document requests no immediate IANA actions. A future revision is expected to register:

- A media type `application/rcr+json` for canonical RCR/1 receipts.
- A `+rcr` structured suffix.
- The well-known URI suffix `/.well-known/rcr-issuer-key`.

A registry for canonicalization profiles (`rcr-jcs-1`, future versions) and for evidence claim types is under discussion in the working draft.

## Appendix A: Reference Implementation

The reference verifier is `verify.py`, published at:

- rankigi.com/verify.py (canonical)
- github.com/rcr-spec/rcr-verify-python (mirrored, with Python package wrapper)

The reference receipt builder is at `src/lib/rcr/receipt.ts` in the rankigi-core repository. RANKIGI Inc. operates the reference implementation; the standard itself is owned by the rcr-spec community.
