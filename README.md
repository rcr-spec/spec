# RCR/1 Receipt Standard

RCR/1 (Receipt Commitment Record, version 1) is an open standard for binding world receipts to AI agent actions.

Every AI agent action that produces a real-world artifact (an email delivery confirmation, a Stripe charge ID, a GitHub commit SHA, a DocuSign envelope completion event) generates a receipt. RCR/1 defines the canonical format for binding that receipt to the agent event that caused it, scoring its credibility, and making both independently verifiable by any third party.

## Status

IETF Internet-Draft: `draft-snow-rcr-receipt-00`
Submission pending at datatracker.ietf.org.

License: CC0 1.0 Universal (Public Domain Dedication).

## The Standard

See [SPEC.md](SPEC.md) for the full specification.

The normative JSON Schema lives at [schema/rcr-receipt-v1.json](schema/rcr-receipt-v1.json).
The RANKIGI closure bundle schema (the container in which RCR/1 receipts are typically shipped) lives at [schema/closure-bundle-v1.5.json](schema/closure-bundle-v1.5.json).

Example bundles and receipts are in [examples/](examples/).

## Reference Implementation

RANKIGI Inc. is the reference implementation of RCR/1. See rankigi.com for the production deployment.

The offline verifier `verify.py` (Python, stdlib only plus optional `cryptography`) is published at rankigi.com/verify.py and mirrored in the [rcr-verify-python](https://github.com/rcr-spec/rcr-verify-python) repository.

A JavaScript/TypeScript verifier is published at [rcr-verify-js](https://github.com/rcr-spec/rcr-verify-js).

## Claim Strength Ladder

RCR/1 defines seven levels of receipt credibility. Verifiers report the strongest claim that an external party can confirm for each receipt.

1. `public_ledger`: the receipt content hash is anchored on a public cryptographic transparency log (Sigstore Rekor, Certificate Transparency, a public blockchain). Highest credibility, third-party witnessed.
2. `regulator_attested`: the receipt is signed or counter-signed by a recognized regulator or qualified attestation authority.
3. `qualified_seal`: the receipt carries a qualified electronic seal under eIDAS or an equivalent jurisdictional regime.
4. `notarized`: the receipt carries an RFC 3161 timestamp from a recognized Time Stamping Authority and a verifiable signer certificate chain.
5. `vendor_signed`: the receipt is signed by the issuing vendor's private key (DKIM signature on an email, Stripe webhook signature, a vendor-issued JWS).
6. `vendor_lookup_token`: the receipt carries a vendor-issued identifier (a Stripe charge ID, a Twilio MessageSid, a GitHub commit SHA) that any party can look up against the vendor's public API.
7. `customer_asserted`: the receipt is self-reported by the agent or its operator. Lowest credibility. Still creates an accountability record.

See [docs/claim-taxonomy.md](docs/claim-taxonomy.md) in the original `rcr-standard` repository for the canonical definitions.

## Canonicalization

RCR/1 uses the `rcr-jcs-1` canonical JSON profile, based on RFC 8785 (JSON Canonicalization Scheme). The receipt hash is SHA-256 over the canonical JSON of the entire receipt, with the `integrity.receipt_hash` field excluded from the hash input.

## Contributing

This standard is governed by the `rcr-spec` community. RANKIGI Inc. is a contributor and the reference implementor; it is not the owner.

All contributions welcome via pull request. Contributions should include:

- A schema issue or pull request describing the change.
- A test vector when behavior changes (see `examples/`).
- A clear statement of what evidence the receipt claims and what it does not prove.
- No raw personal data, protected health information, privileged material, secrets, or credentials in test vectors.

## License

CC0 1.0 Universal. The schema, documentation, and test vectors are dedicated to the public domain.
