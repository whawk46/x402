# Extension: `execution-evidence`

## Summary

The `execution-evidence` extension defines a canonical mechanism for cryptographically binding an on-chain x402 payment settlement to verified proof of digital resource delivery, compute execution, or AI model inference.

While the base x402 specification authenticates and settles value transfer between a client and a resource server, it does not normatively bind the settled transaction to the fulfilled response payload. In autonomous machine-to-machine interactions where no human is in the loop to initiate traditional payment disputes, this gap introduces two systemic challenges:

1. **Delivery Non-Repudiation**: A client cannot mathematically prove whether a paid resource server failed to deliver the promised payload or returned degraded/tampered data.
2. **Economic Wash-Trading Defense**: Circular on-chain settlements between colluding wallets can generate artificial transaction volume on facilitator ledgers. Requiring verifiable execution evidence allows downstream reputation and auditing systems to differentiate authentic computational work from empty volume.

This extension introduces canonical evidence reference identifiers (`x402ev/1`), canonical payload hashing (RFC 8785 JSON Canonicalization Scheme), counterparty-signed delivery receipts, and optional transparency log anchors (IETF SCITT / RFC 9162).

---

## `PaymentRequired`

A resource server advertises support for execution evidence in the `extensions` object of the **402 Payment Required** response.

The extension follows the standard v2 pattern:
- **`info`**: Declares evidence generation capabilities, supported hashing algorithms, and optional transparency log providers.
- **`schema`**: JSON Schema validating the structure of `info`.

### Example

```json
{
  "x402Version": 2,
  "error": "Payment required",
  "resource": {
    "url": "https://api.example.com/v1/inference",
    "description": "Confidential AI inference endpoint"
  },
  "accepts": [ ... ],
  "extensions": {
    "execution-evidence": {
      "info": {
        "supported": true,
        "required": false,
        "schemes": ["x402ev/1", "vaara.receipt/v1"],
        "digestAlgorithms": ["sha256"],
        "transparencyLog": "https://scitt.example.org"
      },
      "schema": {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "type": "object",
        "properties": {
          "supported": { "type": "boolean" },
          "required": { "type": "boolean" },
          "schemes": {
            "type": "array",
            "items": { "type": "string" }
          },
          "digestAlgorithms": {
            "type": "array",
            "items": { "type": "string", "enum": ["sha256", "sha384", "sha512"] }
          },
          "transparencyLog": { "type": "string", "format": "uri" }
        },
        "required": ["supported", "schemes", "digestAlgorithms"]
      }
    }
  }
}
```

---

## `PaymentPayload`

When a client requests that execution evidence be emitted for the transaction, it includes the `execution-evidence` parameter in the `extensions` map of its `PaymentPayload`.

### Example

```json
{
  "x402Version": 2,
  "resource": {
    "url": "https://api.example.com/v1/inference"
  },
  "accepted": {
    "scheme": "exact",
    "network": "eip155:8453"
  },
  "payload": { ... },
  "extensions": {
    "execution-evidence": {
      "info": {
        "requested": true,
        "scheme": "x402ev/1",
        "clientNonce": "d9f8c4e2a1b073e5"
      },
      "schema": {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "type": "object",
        "properties": {
          "requested": { "type": "boolean" },
          "scheme": { "type": "string" },
          "clientNonce": { "type": "string", "minLength": 8, "maxLength": 64 }
        },
        "required": ["requested", "scheme"]
      }
    }
  }
}
```

---

## `PaymentResponse` / Fulfillment Header

Upon successful settlement and resource generation, the resource server delivers the resource payload (HTTP `200 OK`) and emits the `x402-Execution-Evidence` header or returns the evidence object within `EXTENSION-RESPONSES`.

### Header Syntax

```http
x402-Execution-Evidence: uri="x402ev/1:sha256:4a8b7f2d...#scitt=entry_94821"; digest="sha256:4a8b7f2d..."; sig="base64url:..."
```

### Response Extension Object Shape

```json
{
  "extensions": {
    "execution-evidence": {
      "info": {
        "evidenceRef": "x402ev/1:sha256:4a8b7f2d1e9c8b3a7f0e2d4c6b8a1e3f5d7c9b1a3e5f7d9c1b3a5f7d9c1b3a5f",
        "scheme": "x402ev/1",
        "canonicalization": "RFC8785",
        "payment": {
          "network": "eip155:8453",
          "txHash": "0x7a2c1b9d4e8f...",
          "payer": "0x1234...abcd",
          "payee": "0x5678...ef01",
          "amount": "1000000",
          "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
        },
        "request": {
          "method": "POST",
          "urlHash": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
          "payloadHash": "sha256:b5a2c6d8e0f1...",
          "clientNonce": "d9f8c4e2a1b073e5",
          "timestamp": 1791024000
        },
        "delivery": {
          "statusCode": 200,
          "contentDigest": "sha256:4a8b7f2d1e9c8b3a7f0e2d4c6b8a1e3f5d7c9b1a3e5f7d9c1b3a5f7d9c1b3a5f",
          "contentType": "application/json",
          "latencyMs": 42,
          "timestamp": 1791024001
        },
        "signer": {
          "keyType": "Ed25519",
          "publicKey": "0xabcd...1234",
          "signature": "base64url:MEQCIG..."
        },
        "transparency": {
          "logId": "https://scitt.example.org",
          "entryNumber": 94821,
          "inclusionProof": "base64url:..."
        }
      },
      "schema": {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "type": "object",
        "properties": {
          "evidenceRef": { "type": "string" },
          "scheme": { "type": "string" },
          "canonicalization": { "type": "string", "enum": ["RFC8785"] },
          "payment": { "type": "object" },
          "request": { "type": "object" },
          "delivery": { "type": "object" },
          "signer": { "type": "object" },
          "transparency": { "type": "object" }
        },
        "required": ["evidenceRef", "scheme", "canonicalization", "payment", "request", "delivery", "signer"]
      }
    }
  }
}
```

---

## Verification Rules

A client, auditor, or downstream reputation system MUST verify the execution evidence using the following four-stage validation sequence:

1. **Canonical Schema & Signature Verification**:
   - Canonicalize the receipt body using RFC 8785 (JSON Canonicalization Scheme).
   - Verify the `signer.signature` against the resource server's authorized public key.
   - Assert that `payer` and `payee` are distinct non-null entities.

2. **Delivery Content Integrity**:
   - Recompute the SHA-256 digest of the received response body bytes.
   - Assert that the recomputed digest matches `delivery.contentDigest` character for character.

3. **Settlement Binding Verification**:
   - Verify against the settlement ledger that `payment.txHash` confirmed with consensus finality.
   - Confirm that the on-chain transfer transferred the exact `amount` of `asset` from `payer` to `payee`.
   - Ensure `request.timestamp` and `payment.txHash` fall within allowable clock skew windows (default: $\pm 300$ seconds).

4. **Transparency Log Inclusion (Optional / High-Assurance)**:
   - When `transparency` is populated, recompute the leaf hash and verify Merkle inclusion proof against the log's published signed tree head (STH).

---

## Security Considerations

- **Privacy Preservation**: Raw request parameters and response bodies are never published in public ledgers or shared logs. Only SHA-256 digests are anchored, preserving data confidentiality for proprietary enterprise payloads and HIPAA/GDPR-sensitive prompts.
- **Fail-Open Operational Model**: When requested as optional telemetry, evidence generation failure MUST NOT block delivery of the fulfilled digital resource.
- **Sybil Resistance**: By requiring cryptographic binding between confirmed on-chain transaction hashes and distinct counterparty keys, this extension prevents zero-cost receipt manufacturing.
