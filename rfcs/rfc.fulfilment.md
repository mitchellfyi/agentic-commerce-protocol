# RFC: ACP-Fulfilment — Order Status & Shipment Events (Optional Profile)

**Status**: Draft
**Authors**: Mitchell Bryson (mitchellbryson.com)
**Target version**: `acp.fulfilment.v1`
**Requires**: ACP Core (Checkout)
**Non-breaking**: Yes – optional capability

## 1. Summary

This RFC proposes an **optional ACP profile** to expose post-checkout order status and shipment updates to agents. It adds a small, interoperable **event model** and two transport patterns (pull and push) while keeping ACP Core focused on checkout. Merchants advertise support via capability discovery; agents that don’t care about fulfilment can ignore it. ACP Core remains unchanged.

## 2. Motivation

* ACP today standardises **checkout session create/update/complete** and mentions **order events webhooks** to keep state consistent around checkout, but it does not define post-purchase fulfilment semantics. Agents need a predictable way to learn whether an order is **in preparation, shipped, delivered, cancelled, refunded**, etc.
* Keeping fulfilment as an **optional profile** prevents bloating Core, avoids fragmenting identity/auth/signature conventions, and aligns with ACP’s design and repo contribution model (specs + JSON Schema + examples).
* Adjacent efforts (AP2, x402) focus on **payments/mandates**, not logistics. A thin fulfilment layer complements them. 

## 3. Goals

* Define a minimal, boring set of **order/fulfilment events** every agent can parse.
* Support **pull** (`/orders/{id}/events`) and **push** (merchant → agent webhook) with ACP signatures and idempotency.
* Allow **per-line** item status (partial fulfilment/returns).
* Include **shipment and tracking** metadata without prescribing carriers.

## 4. Non-Goals

* Returns/RMA workflows beyond signalling that one was **requested/approved/received**.
* Dispute/chargeback flows (only a terminal `chargeback` status).
* Real-time streaming. A paged poll + webhook push is sufficient initially.
* Replacing a merchant’s existing PSP/settlement rails (unchanged from ACP Core).

## 5. Capability discovery

Merchants indicate support in any ACP Core response that already returns configuration (e.g. at checkout complete):

```json
{
  "capabilities": {
    "fulfilment": {
      "version": "acp.fulfilment.v1",
      "pull": {"events_url": "https://merchant.example.com/orders/{order_id}/events"},
      "push": {"enrol_url": "https://merchant.example.com/orders/{order_id}/callbacks"}
    }
  }
}
```

This follows ACP’s pattern of layering features without altering Core endpoints. 

## 6. Event model

### 6.1 Types

`acp.order.accepted`
`acp.order.rejected`
`acp.order.in_preparation`
`acp.shipment.handed_to_carrier`
`acp.shipment.in_transit`
`acp.shipment.out_for_delivery`
`acp.shipment.delivered`
`acp.order.partially_delivered`
`acp.order.cancelled`
`acp.return.requested` (optional)
`acp.return.received` (optional)
`acp.order.refunded`
`acp.order.chargeback`

These mirror common ecommerce webhook conventions to maximise familiarity and interoperability.

### 6.2 JSON Schema (excerpt)

```json
{
  "$id": "https://agenticcommerce.dev/spec/json-schema/fulfilment.event.v1.json",
  "type": "object",
  "required": ["id","type","created","order","signature"],
  "properties": {
    "id": {"type":"string", "description":"Event id, UUIDv4"},
    "type": {"type":"string", "enum":[
      "acp.order.accepted","acp.order.rejected","acp.order.in_preparation",
      "acp.shipment.handed_to_carrier","acp.shipment.in_transit","acp.shipment.out_for_delivery",
      "acp.shipment.delivered","acp.order.partially_delivered","acp.order.cancelled",
      "acp.return.requested","acp.return.received","acp.order.refunded","acp.order.chargeback"
    ]},
    "created": {"type":"string","format":"date-time"},
    "order": {
      "type":"object",
      "required":["id","status"],
      "properties":{
        "id":{"type":"string"},
        "status":{"type":"string","enum":[
          "accepted","in_preparation","partially_fulfilled","fulfilled",
          "cancelled","refunded","chargeback"
        ]},
        "currency":{"type":"string"},
        "totals":{"type":"object","properties":{
          "items_total":{"type":"integer"},
          "shipping_total":{"type":"integer"},
          "tax_total":{"type":"integer"},
          "grand_total":{"type":"integer"}
        }},
        "items":{"type":"array","items":{"$ref":"#/$defs/fulfilment_item"}}
      }
    },
    "shipment": {"$ref":"#/$defs/shipment"},
    "problems": {"type":"array","items":{"$ref":"#/$defs/problem"}},
    "signature": {"type":"string","description":"ACP HTTP-signature of event body"}
  },
  "$defs": {
    "fulfilment_item": {
      "type":"object",
      "required":["line_item_id","sku","qty","status"],
      "properties":{
        "line_item_id":{"type":"string"},
        "sku":{"type":"string"},
        "qty":{"type":"integer"},
        "status":{"type":"string","enum":[
          "pending","in_preparation","shipped","delivered","cancelled","returned","refunded"
        ]}
      }
    },
    "shipment": {
      "type":"object",
      "properties":{
        "id":{"type":"string"},
        "carrier":{"type":"string"},
        "service":{"type":"string"},
        "tracking_number":{"type":"string"},
        "tracking_url":{"type":"string","format":"uri"},
        "estimated_delivery":{"type":"string","format":"date-time"},
        "last_checkpoint":{"type":"string"}
      }
    },
    "problem": {
      "type":"object",
      "properties":{
        "code":{"type":"string"},
        "message":{"type":"string"},
        "path":{"type":"string","description":"JSONPath to affected object"}
      }
    }
  }
}
```

## 7. Transports

### 7.1 Pull (required)

```
GET /orders/{order_id}/events?since={rfc3339}&page_token={token}
```

* **Auth**: same as ACP Core; include `Signature`, `Idempotency-Key`, `Timestamp` headers.
* **Response**: array of `fulfilment.event.v1` sorted by `created` asc; include `next_page_token` if more.
* **Idempotency**: agents de-duplicate by `event.id`. 

### 7.2 Push (optional)

**Agent enrols a callback** after checkout completion:

```
POST /orders/{order_id}/callbacks
{ "url": "https://agent.example/callbacks/acp",
  "events": ["acp.shipment.*","acp.order.*"],
  "secret": "hmac-shared-secret" }
```

Merchant thereafter `POST`s events to `url` with ACP signature headers. Retry with exponential backoff; deliver **at-least-once**. The push model mirrors ACP’s use of webhooks for authoritative order state during checkout. 

## 8. OpenAPI (excerpt)

```yaml
openapi: 3.1.0
info:
  title: ACP Fulfilment
  version: acp.fulfilment.v1
paths:
  /orders/{order_id}/events:
    get:
      summary: List order/fulfilment events
      parameters:
        - in: path
          name: order_id
          required: true
          schema: { type: string }
        - in: query
          name: since
          schema: { type: string, format: date-time }
        - in: query
          name: page_token
          schema: { type: string }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                required: [events]
                properties:
                  events:
                    type: array
                    items:
                      $ref: '#/components/schemas/FulfilmentEvent'
                  next_page_token:
                    type: string
components:
  schemas:
    FulfilmentEvent:
      $ref: 'https://agenticcommerce.dev/spec/json-schema/fulfilment.event.v1.json'
```

## 9. Security

* **Auth**: reuse ACP request authentication and signature verification; sign webhook bodies as in Core.
* **Replay**: verify `Timestamp` drift and `Idempotency-Key`; store last `event.id` per order.
* **PII**: events must not introduce new buyer PII beyond what Core returns.

## 10. Versioning & Compatibility

* Profile identifier: `acp.fulfilment.v1`.
* Advertised via `capabilities.fulfilment.version`.
* Backwards-compatible additions: new event types may be added; agents must ignore unknown `type`s.
* Conformance artefacts to be added under `spec/json-schema/` and `spec/openapi/`; examples under `examples/`; changelog entry via repo guidelines.

## 11. Examples

### 11.1 Shipment handed to carrier (push)

```http
POST /agent/callbacks/acp HTTP/1.1
Content-Type: application/json
Signature: sig=..., key-id=merchant-abc, ts=2025-10-05T20:45:12Z
Idempotency-Key: 1b3e6b3a-...

{
  "id": "evt_8a6e...",
  "type": "acp.shipment.handed_to_carrier",
  "created": "2025-10-05T20:45:12Z",
  "order": {
    "id": "ord_123",
    "status": "in_preparation"
  },
  "shipment": {
    "id": "shp_1",
    "carrier": "ups",
    "service": "ground",
    "tracking_number": "1Z999...",
    "tracking_url": "https://track.example/1Z999..."
  },
  "signature": "base64..."
}
```

### 11.2 Delivered (pull)

```json
{
  "events": [
    {
      "id":"evt_dlv",
      "type":"acp.shipment.delivered",
      "created":"2025-10-06T09:01:00Z",
      "order":{"id":"ord_123","status":"fulfilled"},
      "shipment":{"id":"shp_1","last_checkpoint":"Delivered at front door"},
      "signature":"base64..."
    }
  ]
}
```

## 12. Implementation guidance

* **Schema mapping**: merchants can map their native events (e.g. `fulfilled`, `partially_fulfilled`, `cancelled`) to the standard set here. Prior art from mainstream ecommerce webhook topics can guide mappings. 
* **Minimal viable set**: start with `accepted | in_preparation | handed_to_carrier | in_transit | delivered | cancelled | refunded`.
* **Tracking**: include `tracking_url` if available; omit rather than fake.
* **Partial fulfilment**: update `items[].status` and use `acp.order.partially_delivered` when any line reaches `delivered` but others remain pending.

[5]: https://shopify.dev/docs/api/webhooks?utm_source=chatgpt.com "Webhooks"
[6]: https://stripe.com/blog/developing-an-open-standard-for-agentic-commerce?utm_source=chatgpt.com "Developing an open standard for agentic commerce"
