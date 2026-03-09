# Payments API Reference

> **Base URL (Sandbox):** `https://api-m.sandbox.example.com`  
> **Base URL (Production):** `https://api-m.example.com`  
> **Version:** v2  
> **Related resources:** [Common Resources](./common-resources.md) · [Integration Guide](./integration-guide.md)

---

## Authentication

All requests require a Bearer token in the `Authorization` header. Refer to [Authentication](./common-resources.md#authentication) in Common Resources for the full token acquisition flow.

```
Authorization: Bearer {access_token}
```

---

## Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/v2/checkout/orders` | Create an order |
| `GET` | `/v2/checkout/orders/{order_id}` | Get order details |
| `POST` | `/v2/checkout/orders/{order_id}/authorize` | Authorize payment for an order |
| `POST` | `/v2/checkout/orders/{order_id}/capture` | Capture payment for an order |
| `POST` | `/v2/payments/authorizations/{auth_id}/capture` | Capture an authorization |
| `POST` | `/v2/payments/authorizations/{auth_id}/void` | Void an authorization |
| `POST` | `/v2/payments/captures/{capture_id}/refund` | Refund a captured payment |
| `GET` | `/v2/payments/captures/{capture_id}` | Get capture details |

---

## POST /v2/checkout/orders

Creates a new payment order.

### Request headers

| Header | Type | Required | Description |
|---|---|---|---|
| `Authorization` | string | Yes | Bearer access token. |
| `Content-Type` | string | Yes | Must be `application/json`. |
| `Example-Request-Id` | string | Recommended | Idempotency key (UUID v4). See [Idempotency](./common-resources.md#idempotency). |

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `intent` | string (enum) | Yes | Payment intent. `CAPTURE` collects funds immediately; `AUTHORIZE` places a hold. |
| `purchase_units` | array | Yes | Array of purchase unit objects. Minimum: 1. Maximum: 10. |
| `purchase_units[].reference_id` | string | No | Merchant-defined reference. Max 256 characters. |
| `purchase_units[].amount` | object | Yes | Amount object. See [Amount object](#amount-object). |
| `purchase_units[].description` | string | No | Order description shown to buyer. Max 127 characters. |
| `application_context` | object | No | Redirect URLs and buyer experience settings. |
| `application_context.return_url` | string | Yes (if redirect) | URL to redirect the buyer after approval. |
| `application_context.cancel_url` | string | Yes (if redirect) | URL to redirect the buyer on cancellation. |
| `application_context.brand_name` | string | No | Company name displayed during checkout. |
| `application_context.user_action` | string (enum) | No | `PAY_NOW` or `CONTINUE`. Default: `CONTINUE`. |

#### Amount object

| Field | Type | Required | Description |
|---|---|---|---|
| `currency_code` | string | Yes | ISO 4217 three-letter currency code (for example, `USD`, `EUR`, `GBP`). |
| `value` | string | Yes | Amount as a decimal string (for example, `"100.00"`). |
| `breakdown` | object | No | Itemised breakdown of the total amount. |
| `breakdown.item_total` | object | No | Subtotal of all items. |
| `breakdown.shipping` | object | No | Shipping cost. |
| `breakdown.tax_total` | object | No | Tax amount. |
| `breakdown.discount` | object | No | Discount applied. Must be negative or zero. |

### Response

**201 Created**

```json
{
  "id": "5O190127TN364715T",
  "status": "CREATED",
  "links": [
    {
      "href": "https://api-m.sandbox.example.com/v2/checkout/orders/5O190127TN364715T",
      "rel": "self",
      "method": "GET"
    },
    {
      "href": "https://www.sandbox.example.com/checkoutnow?token=5O190127TN364715T",
      "rel": "approve",
      "method": "GET"
    },
    {
      "href": "https://api-m.sandbox.example.com/v2/checkout/orders/5O190127TN364715T/authorize",
      "rel": "authorize",
      "method": "POST"
    }
  ]
}
```

**Order status values**

| Status | Description |
|---|---|
| `CREATED` | Order created; awaiting buyer approval. |
| `SAVED` | Order saved for later; not yet approved. |
| `APPROVED` | Buyer has approved; ready to authorize or capture. |
| `VOIDED` | Order has been voided and cannot be transacted. |
| `COMPLETED` | Order is complete; funds captured or authorized. |
| `PAYER_ACTION_REQUIRED` | Buyer must complete an action (for example, 3DS challenge). |

---

## GET /v2/checkout/orders/{order_id}

Retrieves the details of an existing order.

### Path parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `order_id` | string | Yes | The order ID returned in the `POST /v2/checkout/orders` response. |

### Response

**200 OK** — Returns the full order object. Same schema as the create response, with an additional `purchase_units[].payments` object if authorization or capture has occurred.

---

## POST /v2/checkout/orders/{order_id}/authorize

Authorizes payment for an approved order. Places a hold on the buyer's funding source.

### Path parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `order_id` | string | Yes | The order ID to authorize. |

### Response

**201 Created**

```json
{
  "id": "5O190127TN364715T",
  "status": "COMPLETED",
  "purchase_units": [
    {
      "reference_id": "PU-001",
      "payments": {
        "authorizations": [
          {
            "id": "3C679366HH908993T",
            "status": "CREATED",
            "amount": { "currency_code": "USD", "value": "100.00" },
            "expiration_time": "2025-04-12T10:30:00Z",
            "links": [
              {
                "href": "https://api-m.sandbox.example.com/v2/payments/authorizations/3C679366HH908993T",
                "rel": "self",
                "method": "GET"
              },
              {
                "href": "https://api-m.sandbox.example.com/v2/payments/authorizations/3C679366HH908993T/capture",
                "rel": "capture",
                "method": "POST"
              },
              {
                "href": "https://api-m.sandbox.example.com/v2/payments/authorizations/3C679366HH908993T/void",
                "rel": "void",
                "method": "POST"
              }
            ]
          }
        ]
      }
    }
  ]
}
```

**Authorization status values**

| Status | Description |
|---|---|
| `CREATED` | Authorization active; funds on hold. |
| `CAPTURED` | Authorization fully captured. |
| `PARTIALLY_CAPTURED` | Authorization partially captured; remaining hold is active. |
| `VOIDED` | Authorization voided; hold released. |
| `EXPIRED` | Authorization expired after 29 days without capture. |
| `PENDING` | Authorization pending additional buyer action. |

---

## POST /v2/payments/authorizations/{auth_id}/capture

Captures an existing authorization, collecting the reserved funds.

### Path parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `auth_id` | string | Yes | The authorization ID to capture. |

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `amount` | object | No | Amount to capture. Omit to capture the full authorized amount. |
| `amount.currency_code` | string | Yes (if amount) | ISO 4217 currency code. Must match the original authorization. |
| `amount.value` | string | Yes (if amount) | Amount to capture as a decimal string. Cannot exceed authorized amount. |
| `final_capture` | boolean | No | If `true`, releases the remaining hold after this capture. Default: `false`. |
| `note_to_payer` | string | No | Note to the buyer. Appears on their transaction record. Max 255 characters. |
| `invoice_id` | string | No | Merchant invoice reference. Max 127 characters. |

### Response

**201 Created**

```json
{
  "id": "2GG279541U471931P",
  "status": "COMPLETED",
  "amount": { "currency_code": "USD", "value": "100.00" },
  "final_capture": true,
  "invoice_id": "INV-2025-0312",
  "seller_protection": {
    "status": "ELIGIBLE",
    "dispute_categories": ["ITEM_NOT_RECEIVED", "UNAUTHORIZED_TRANSACTION"]
  },
  "create_time": "2025-03-06T10:45:00Z",
  "update_time": "2025-03-06T10:45:03Z"
}
```

**Capture status values**

| Status | Description |
|---|---|
| `COMPLETED` | Funds captured successfully. |
| `DECLINED` | Capture declined by the issuer. |
| `PARTIALLY_REFUNDED` | Capture has been partially refunded. |
| `REFUNDED` | Capture has been fully refunded. |
| `PENDING` | Capture pending; awaiting settlement. |
| `FAILED` | Capture failed. |

---

## POST /v2/payments/authorizations/{auth_id}/void

Voids an active authorization, releasing the hold on buyer funds.

### Path parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `auth_id` | string | Yes | The authorization ID to void. |

### Response

**204 No Content** — No response body. A `204` confirms the authorization was voided.

> **Note:** You cannot void an authorization that has already been captured, partially captured, or expired.

---

## POST /v2/payments/captures/{capture_id}/refund

Refunds a completed capture, either fully or partially.

### Path parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `capture_id` | string | Yes | The capture ID to refund. |

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `amount` | object | No | Amount to refund. Omit to refund the full capture amount. |
| `amount.currency_code` | string | Yes (if amount) | ISO 4217 currency code. Must match the original capture. |
| `amount.value` | string | Yes (if amount) | Amount to refund as a decimal string. |
| `note_to_payer` | string | No | Reason for refund shown to the buyer. Max 255 characters. |
| `invoice_id` | string | No | Merchant reference for this refund. Max 127 characters. |

### Response

**201 Created**

```json
{
  "id": "1JU08902781691411",
  "status": "COMPLETED",
  "amount": { "currency_code": "USD", "value": "100.00" },
  "create_time": "2025-03-06T12:00:00Z",
  "update_time": "2025-03-06T12:00:04Z"
}
```

---

## HTTP status codes

| Code | Name | Description |
|---|---|---|
| `200` | OK | Request succeeded. |
| `201` | Created | Resource created successfully. |
| `204` | No Content | Request succeeded; no response body. |
| `400` | Bad Request | Malformed request syntax or invalid parameters. |
| `401` | Unauthorized | Invalid or expired access token. |
| `403` | Forbidden | Valid token, but insufficient scope. |
| `404` | Not Found | Resource does not exist. |
| `409` | Conflict | Request conflicts with the current state of the resource. |
| `422` | Unprocessable Entity | Semantically invalid request (for example, amount mismatch). |
| `429` | Too Many Requests | Rate limit exceeded. See [Rate Limiting](./common-resources.md#rate-limiting). |
| `500` | Internal Server Error | Unexpected server-side error. Retry with exponential backoff. |

---

## Error response schema

All error responses follow this structure:

```json
{
  "name": "UNPROCESSABLE_ENTITY",
  "details": [
    {
      "field": "/purchase_units/@reference_id=='PU-001'/amount/value",
      "issue": "AMOUNT_MISMATCH",
      "description": "Capture amount exceeds the authorized amount."
    }
  ],
  "message": "The requested action could not be performed, semantically incorrect, or failed business validation.",
  "debug_id": "c9d3f3b8d921c",
  "links": [
    {
      "href": "https://developer.example.com/api/rest/reference/orders/v2/errors/#AMOUNT_MISMATCH",
      "rel": "information_link",
      "method": "GET"
    }
  ]
}
```

| Field | Description |
|---|---|
| `name` | High-level error category. |
| `details` | Array of specific validation errors. |
| `details[].field` | JSON Pointer to the offending field. |
| `details[].issue` | Machine-readable error code. |
| `details[].description` | Human-readable explanation. |
| `debug_id` | Unique identifier for this request. Include in support requests. |
| `links` | Links to developer documentation for this error. |

---

## Common error codes

| Error code | HTTP status | Description | Resolution |
|---|---|---|---|
| `INVALID_TOKEN` | `401` | Access token is invalid or expired. | Re-authenticate using client credentials. |
| `INSUFFICIENT_SCOPE` | `403` | Token lacks required OAuth scope. | Request appropriate scope. See [OAuth Scopes](./common-resources.md#oauth-scopes). |
| `RESOURCE_NOT_FOUND` | `404` | Order, authorization, or capture not found. | Verify the ID and retry. |
| `ORDER_ALREADY_CAPTURED` | `422` | A capture already exists for this order. | Do not retry; retrieve order status. |
| `AUTHORIZATION_EXPIRED` | `422` | Authorization is older than 29 days. | Create a new order and restart the flow. |
| `AMOUNT_MISMATCH` | `422` | Capture amount exceeds authorization. | Reduce capture amount or create a new order. |
| `COMPLIANCE_VIOLATION` | `422` | Transaction declined for compliance reasons. | Contact support with the `debug_id`. |
| `RATE_LIMIT_REACHED` | `429` | Too many requests in a short period. | Apply exponential backoff. See [Rate Limiting](./common-resources.md#rate-limiting). |
| `INTERNAL_SERVER_ERROR` | `500` | Unexpected error on Example Payments' servers. | Retry with exponential backoff. |
