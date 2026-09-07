# KCB Instant Payment Notification (IPN) — Receiving Endpoint

**Prepared for:** KCB Buni Integration Team (`buni@kcbgroup.com`)
**Subject:** Third-party IPN notification endpoint specification
**Status:** Ready for UAT

This document describes the endpoint we expose to receive Instant Payment Notifications, as defined in the *KCB Instant Payment Notification (IPN) API Specification Document*. We accept KCB's payload verbatim — no changes to the request contract are requested.

---

## 1. Endpoint

| | |
|---|---|
| **Transport protocol** | HTTPS |
| **Interaction type** | Synchronous |
| **Method** | `POST` |
| **Message format** | JSON (`Content-Type: application/json`) |
| **Character encoding** | UTF-8 |

### URL

```
POST https://api.nwrealite.app/api/v1/integration/nw-realite-ltd/banks/kcb/ipn
```

| Segment | Meaning |
|---|---|
| `api.nwrealite.app` | Fixed. Our API host. |
| `nw-realite-ltd` | The account holder's organisation identifier on our platform. This value is fixed for NW Realite Ltd and never changes. |
| `banks/kcb` | Fixed. Identifies KCB as the notifying institution. |

**Environment URLs**

| Environment | URL |
|---|---|
| UAT | `https://api.nwrealite.app/api/v1/integration/nw-realite-ltd/banks/kcb/ipn` |
| Production | `https://api.nwrealite.app/api/v1/integration/nw-realite-ltd/banks/kcb/ipn` |

The same endpoint serves UAT and production traffic. UAT notifications are distinguished by their content (test references and amounts), not by a separate URL.

---

## 2. Authentication

**No bearer token or API key is required.** The endpoint authenticates the caller by the `Signature` header described in the KCB specification.

| Header | Required | Notes |
|---|---|---|
| `Signature` | Yes | Base64-encoded RSA signature, as per the sample in KCB's specification. |
| `Content-Type` | Yes | `application/json` |

**Verification model:** we verify the signature as `RSA-SHA256` over the **raw, unmodified request body** using KCB's public key. During the initial UAT window the signature is **recorded but not enforced**, so that KCB can begin sending traffic before key exchange completes. Once the public key is installed, an invalid signature is rejected (see `statusCode` `1` below).

> **Action required from KCB:** please supply (a) the public key in PEM format, and (b) confirmation of exactly what is signed — the raw JSON body as transmitted, or a canonicalised/concatenated form of specific fields. If it is the latter, we need the field order and separator.

---

## 3. Request body

We accept all fields exactly as specified by KCB.

| Parameter | Type | Requirement |
|---|---|---|
| `transactionReference` | String | mandatory |
| `requestId` | String | mandatory |
| `channelCode` | String | mandatory |
| `timestamp` | timeStamp | mandatory |
| `transactionAmount` | String | mandatory |
| `currency` | String | mandatory |
| `customerReference` | String | mandatory |
| `customerName` | String | mandatory |
| `customerMobileNumber` | String | mandatory |
| `balance` | String | mandatory *(empty string accepted)* |
| `narration` | String | mandatory |
| `creditAccountIdentifier` | String | mandatory |
| `organizationShortCode` | String | mandatory |
| `tillNumber` | String | mandatory |

Notes:

- `timestamp` is accepted as an opaque string; no particular date format is imposed.
- `balance` may be sent as an empty string, as in KCB's own sample payload.
- Unrecognised additional fields are accepted and recorded rather than rejected.

### Sample request

```http
POST /api/v1/integration/nw-realite-ltd/banks/kcb/ipn HTTP/1.1
Host: api.nwrealite.app
Content-Type: application/json
Signature: 69EJ7+KmfkYHCu7+2mtAks5aFXyQUcEvuZjlpRMEbNszApUymF9eFt25QDb/...

{
  "transactionReference": "FT00026252",
  "requestId": "c7d702cb-6b5f-4fa6-8b57-436d0f789017",
  "channelCode": "202",
  "timestamp": "2021111103005",
  "transactionAmount": "100.00",
  "currency": "KES",
  "customerReference": "INV-0001",
  "customerName": "John Doe",
  "customerMobileNumber": "25471111111",
  "balance": "",
  "narration": "Payment for goods",
  "creditAccountIdentifier": "JD001",
  "organizationShortCode": "777777",
  "tillNumber": "150150"
}
```

---

## 4. Response body

Returned at the **top level** of the JSON document, unwrapped, exactly as specified by KCB.

| Parameter | Type | Requirement |
|---|---|---|
| `transactionID` | String | mandatory |
| `statusCode` | String | mandatory |
| `statusMessage` | String | mandatory |

`transactionID` echoes back the `transactionReference` from the request, so both sides share a correlation handle.

### Status codes

| `statusCode` | `statusMessage` | Meaning |
|---|---|---|
| `0` | `Notification received` | Accepted and recorded. No retry required. |
| `1` | `Invalid signature` | Signature verification failed. The notification was not accepted. |

### Sample response

```json
{
  "transactionID": "FT00026252",
  "statusCode": "0",
  "statusMessage": "Notification received"
}
```

---

## 5. HTTP status codes

The endpoint always returns **HTTP 200** for a notification it has processed, whether accepted or rejected — the outcome is carried in `statusCode`, so KCB's parser never has to handle a non-200 body shape.

The following are the only non-200 responses, and all indicate a configuration or addressing problem rather than a payment problem:

| HTTP | Cause | Resolution |
|---|---|---|
| `404` | The organisation segment is not `nw-realite-ltd`, or the bank path is not `banks/kcb`. | Check the URL against Section 1. |
| `429` | Rate limit exceeded (see below). | Retry after the interval in the `Retry-After` header. |
| `5xx` | Unexpected error on our side. | Safe to retry; the notification was not recorded. |

---

## 6. Idempotency and retries

- Every delivery is recorded, **including retries**. We do not deduplicate on KCB's behalf.
- We treat `requestId` and `transactionReference` as the natural deduplication keys. Please keep these stable across retries of the same notification.
- A `statusCode` of `0` means the notification is safely recorded and **should not be retried**.

> **Action required from KCB:** please confirm your retry policy — number of attempts, backoff interval, and the request timeout after which a delivery is considered failed.

---

## 7. Rate limiting

The endpoint accepts **300 requests per minute per source IP address**. Requests beyond this receive HTTP `429` with a `Retry-After` header.

> **Action required from KCB:** please confirm your expected peak notification rate so we can raise this ceiling ahead of go-live if required.

---

## 8. Checklist — what we need from KCB

| # | Item | Why |
|---|---|---|
| 1 | Public key (PEM) used to sign the `Signature` header | To enable signature verification |
| 2 | Definition of the signed content (raw body vs. canonicalised fields) | To verify correctly on the first attempt |
| 3 | Source IP addresses / CIDR ranges for UAT and production | For network-level allowlisting |
| 4 | Retry policy and request timeout | To align idempotency handling |
| 5 | Expected peak notification rate | To size the rate limit |
| 6 | Mapping of `organizationShortCode` / `tillNumber` / `creditAccountIdentifier` to each onboarded account | To route each notification to the correct organisation |
| 7 | UAT test credentials and a scheduled test window | To run end-to-end certification |

---

## 9. Contact

Please direct integration queries and the items above to our integration team. We will confirm receipt of the public key and schedule a joint UAT session on delivery.

---

## Appendix — internal implementation notes

*Not part of the bank-facing specification.*

| Concern | Location |
|---|---|
| Route | [routes/Api/v1/integration.php](../../routes/Api/v1/integration.php) — `integration.banks.ipn` |
| Controller | [app/Http/Controllers/Api/V1/Integration/Bank/BankIpnController.php](../../app/Http/Controllers/Api/V1/Integration/Bank/BankIpnController.php) |
| Signature middleware | [app/Http/Middleware/VerifyBankIpnSignature.php](../../app/Http/Middleware/VerifyBankIpnSignature.php), alias `bank.ipn.signature` |
| Payload DTO | [app/Data/Integration/Bank/BankIpnNotificationData.php](../../app/Data/Integration/Bank/BankIpnNotificationData.php) |
| Provider registry | [config/banks.php](../../config/banks.php) |
| Log channel | `bank_ipn` → `storage/logs/bank-ipn/bank-ipn-YYYY-MM-DD.log`, retained 90 days |
| Rate limiter | `bank-ipn`, defined in [app/Providers/RouteServiceProvider.php](../../app/Providers/RouteServiceProvider.php) |
| Tests | [tests/Feature/Integration/BankIpnTest.php](../../tests/Feature/Integration/BankIpnTest.php) |

Environment variables:

```dotenv
BANK_KCB_IPN_ENABLED=true
BANK_KCB_IPN_PUBLIC_KEY=      # PEM contents or absolute path; empty => signature logged, not enforced
```

**Current phase:** notifications are logged only. Posting the credit to the cashbook (`bank_account_transactions`), matching `customerReference` to an invoice or lease, and resolving `creditAccountIdentifier`/`tillNumber` to a `BankAccount` are deliberately deferred until UAT traffic shows the real shape of the data.
