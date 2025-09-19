# CPRAS Partner API (PrintPLANR)

## Purpose

This document details the **lightweight integration** between CPRAS and PrintPLANR, enabling contextual handoff (supplier/buyer) and order/status webhook exchanges.  
**Goal:** Quick, minimal-ambiguity implementation for both teams. CPRAS stays a simple gateway — PrintPLANR runs catalog/order UI and logic.

---

## Example Flow (MVP)

1. **CPRAS authenticates** with PrintPLANR.
2. **CPRAS initiates Supplier or Buyer Handoff** by passing minimal context.
3. **PrintPLANR manages user session, catalog, and ordering UI.**
4. **PrintPLANR sends webhook events** for supplier provisioning or order status.
5. **CPRAS updates "Print Orders" dashboard/status** only, never hosting catalog/order UI.

---

## Auth

- **OAuth2 Client Credentials** for PrintPLANR → CPRAS calls  
  - `POST /oauth/token` → `{ access_token, token_type, expires_in }`
- **CPRAS → PrintPLANR calls** (when needed) use PrintPLANR’s authentication (see PrintPLANR API docs).
- **Webhooks** from PrintPLANR are signed with HMAC: header `X-PrintPlanr-Signature: sha256=…`

---

## Feature & Endpoint Summary

| Endpoint                                  | Required (MVP) | Description                                 |
| ------------------------------------------ | :------------: | ------------------------------------------- |
| `POST /oauth/token`                        |      Yes       | Issue token for PrintPLANR → CPRAS webhook  |
| `POST /v1/webhooks/printplanr`             |      Yes       | Accept webhook events from PrintPLANR       |
| `GET /v1/partners/printplanr/context/suppliers/{supplier_id}` | Optional | Query supplier context from CPRAS           |
| `POST /v1/partners/printplanr/sessions/consume`             | Optional | SSO session handoff from PrintPLANR         |

---

## Endpoints (CPRAS)

### 1) Issue Token (for PrintPLANR → CPRAS)
`POST /oauth/token`  
Form/JSON: `{ client_id, client_secret, scope }`  
Response:  
```
{ "access_token": "...", "token_type": "Bearer", "expires_in": 600 }
```

### 2) Webhook Receiver (PrintPLANR → CPRAS)
`POST /v1/webhooks/printplanr`  
Headers:  
- `Authorization: Bearer <token>`
- `X-PrintPlanr-Signature: sha256=...`

Body examples:  
```
{
  "type": "supplier.provisioned",
  "id": "evt_001",
  "occurred_at": "2025-09-17T09:10:00Z",
  "data": {
    "supplier_external_id": "ppl_sup_456",
    "supplier_id": "sup_123",
    "status": "active"
  }
}
```
```
{
  "type": "order.created",
  "id": "evt_002",
  "occurred_at": "2025-09-17T09:12:00Z",
  "data": {
    "order_id": "pp_order_555",
    "buyer_id": "buyer_101",
    "project_id": "proj_wdc_2025",
    "lot_id": "lot_1",
    "supplier_id": "sup_123",
    "total": 1234.56,
    "currency": "GBP",
    "status": "pending"
  }
}
```
Response: `200 OK` (idempotent by `id`)

### 3) Context Lookup (Optional)
`GET /v1/partners/printplanr/context/suppliers/{supplier_id}`  
Example Response:
```
{
  "supplier_id": "sup_123",
  "company_name": "Acme Print Ltd",
  "primary_email": "[ops@acmeprint.co.uk](mailto:ops@acmeprint.co.uk)",
  "framework": { "framework_id": "fw_wdc_print", "contract_id": "ctr_789", "project_id": "proj_wdc_2025" },
  "contracted_lots": [
    { "lot_id": "lot_1", "lot_code": "LOT1", "title": "General Print" },
    { "lot_id": "lot_2", "lot_code": "LOT2", "title": "Large Format" }
  ],
  "pricing_policy": { "display": "as_is" }  // or "with_margin"
}
```

### 4) SSO Return Consume (Optional)
`POST /v1/partners/printplanr/sessions/consume`  
Request:
```
{ "code": "one-time-code-from-printplanr" }
```
Response:
```
{ "ok": true, "project_id": "proj_wdc_2025", "lot_id": "lot_1", "next": "/projects/proj_wdc_2025/orders/pp_order_555" }
```

---

## What CPRAS Sends **to** PrintPLANR (Data Contract)

### Supplier Handoff
```
{
  "supplier": { "id": "sup_123", "company_name": "Acme Print Ltd", "primary_email": "[ops@acme.co.uk](mailto:ops@acme.co.uk)", "user_name": "acme.ops" },
  "framework": { "framework_id": "fw_wdc_print", "contract_id": "ctr_789", "project_id": "proj_wdc_2025" },
  "contracted_lots": [
    { "lot_id": "lot_1", "lot_code": "LOT1", "title": "General Print", "effective_from": "2025-09-01", "effective_to": "2027-08-31" }
  ],
  "redirect": { "return_url": "https://ukprocure.cpras.co.uk/return/printplanr" }
}
```

### Buyer Handoff
```
{
  "buyer": { "id": "buyer_101", "org_name": "Warwick District Council", "email": "[user@wdc.gov.uk](mailto:user@wdc.gov.uk)" },
  "context": { "project_id": "proj_wdc_2025", "lot_id": "lot_1", "supplier_ids": ["sup_123"], "mode": "order" },
  "pricing_policy": { "display": "as_is" },
  "redirect": { "return_url": "https://ukprocure.cpras.co.uk/return/printplanr" }
}
```

---

## Security & Ops

- Short-lived tokens (≤10 min).
- Least-privilege OAuth scopes.
- All inbound webhooks MUST have HMAC signatures, verified server-side.
- Webhook retries (with exponential backoff) until 200 OK returned.
- Idempotency by event `id`.
- PII minimization throughout.

---

## MVP UX Recommendation

**Redirect model (fastest to go-live):**  
- CPRAS = small “Print Orders” area → calls Buyer Handoff → redirects to PrintPLANR session URL.  
- PrintPLANR owns catalog/ordering UI.  
- CPRAS needs only to store and display basic status updates, informed by PrintPLANR webhooks.

---

# OpenAPI 3.0 (minimal/for dev)

```
openapi: 3.0.3
info:
  title: CPRAS Partner API (PrintPLANR)
  version: 0.1.0
servers:
  - url: https://api.ukprocure.cpras.co.uk/
paths:
  /oauth/token:
    post:
      summary: Issue OAuth2 token (Client Credentials)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                client_id: { type: string }
                client_secret: { type: string }
                scope: { type: string }
              required: [client_id, client_secret]
      responses:
        '200':
          description: Token issued
          content:
            application/json:
              schema:
                type: object
                properties:
                  access_token: { type: string }
                  token_type: { type: string, example: Bearer }
                  expires_in: { type: integer, example: 600 }
  /v1/webhooks/printplanr:
    post:
      summary: Receive PrintPLANR webhook events
      security:
        - bearerAuth: []
      parameters:
        - in: header
          name: X-PrintPlanr-Signature
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              oneOf:
                - $ref: '#/components/schemas/SupplierProvisionedEvent'
                - $ref: '#/components/schemas/OrderEvent'
      responses:
        '200':
          description: Acknowledged
  /v1/partners/printplanr/context/suppliers/{supplier_id}:
    get:
      summary: (Optional) Read supplier context from CPRAS
      security:
        - bearerAuth: []
      parameters:
        - in: path
          name: supplier_id
          required: true
          schema: { type: string }
      responses:
        '200':
          description: Context returned
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SupplierContext'
  /v1/partners/printplanr/sessions/consume:
    post:
      summary: (Optional) Consume a one-time session code from PrintPLANR
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [code]
              properties:
                code: { type: string }
      responses:
        '200':
          description: Session consumed
          content:
            application/json:
              schema:
                type: object
                properties:
                  ok: { type: boolean, example: true }
                  project_id: { type: string }
                  lot_id: { type: string }
                  next: { type: string }
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
  schemas:
    SupplierContext:
      type: object
      properties:
        supplier_id: { type: string }
        company_name: { type: string }
        primary_email: { type: string, format: email }
        framework:
          type: object
          properties:
            framework_id: { type: string }
            contract_id: { type: string }
            project_id: { type: string }
        contracted_lots:
          type: array
          items:
            type: object
            properties:
              lot_id: { type: string }
              lot_code: { type: string }
              title: { type: string }
        pricing_policy:
          type: object
          properties:
            display:
              type: string
              enum: [as_is, with_margin]
    SupplierProvisionedEvent:
      type: object
      properties:
        type: { type: string, enum: [supplier.provisioned] }
        id: { type: string }
        occurred_at: { type: string, format: date-time }
        data:
          type: object
          properties:
            supplier_external_id: { type: string }
            supplier_id: { type: string }
            status: { type: string }
    OrderEvent:
      type: object
      properties:
        type:
          type: string
          enum: [order.created, order.updated, order.completed]
        id: { type: string }
        occurred_at: { type: string, format: date-time }
        data:
          type: object
          properties:
            order_id: { type: string }
            buyer_id: { type: string }
            project_id: { type: string }
            lot_id: { type: string }
            supplier_id: { type: string }
            total: { type: number }
            currency: { type: string }
            status: { type: string }
```

