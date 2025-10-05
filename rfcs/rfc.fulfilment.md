# RFC: ACP-Fulfilment — Order Status & Shipment Events (Optional Profile)

**Status**: Draft
**Authors**: Mitchell Bryson
**Target version**: `acp.fulfilment.v1`
**Requires**: ACP Core (Checkout)
**Non-breaking**: Yes – optional capability

## 1. Summary

This RFC proposes an optional ACP profile that defines how merchants can surface post-checkout order status and shipment events to agents. It introduces a minimal event model and two transports (pull & push) while leaving ACP Core unchanged. Merchants advertise support via `capabilities.fulfilment` in existing core responses.

## 2. Motivation

- ACP v1 defines checkout session semantics but not post purchase fulfilment.
- Agents need a standard way to know if an order is in preparation, shipped, delivered, cancelled or refunded.
- Keeping fulfilment as a separate profile avoids bloating Core and reuses existing auth and signing conventions.

## 3. Goals

- A small set of standard events with predictable names and payloads.
- Support both pull (`/orders/{id}/events`) and push (webhook) models.
- Provide per line item status and shipment metadata.

## 4. Event types

Events are namespaced under `acp.*` and include (but are not limited to):

- `acp.order.accepted`
- `acp.order.rejected`
- `acp.order.in_preparation`
- `acp.shipment.handed_to_carrier`
- `acp.shipment.in_transit`
- `acp.shipment.out_for_delivery`
- `acp.shipment.delivered`
- `acp.order.partially_delivered`
- `acp.order.cancelled`
- `acp.return.requested` (optional)
- `acp.return.received` (optional)
- `acp.order.refunded`
- `acp.order.chargeback`

## 5. Transports

**Pull**: Agents call `GET /orders/{order_id}/events?since=` and receive a paginated list of events sorted by creation time.

**Push**: After checkout an agent may register a callback; the merchant posts events to this URL with ACP signatures and idempotency keys. Retries use exponential backoff and are at‑least‑once.

## 6. Schema & OpenAPI

Detailed JSON Schema definitions for `fulfilment.event.v1` and an OpenAPI path specification for `/orders/{order_id}/events` should be added under `spec/json-schema/` and `spec/openapi/`.

Each event includes a unique `id`, `type`, `created` timestamp, `order` summary, optional `shipment` and `problems` objects, and an `signature` string. Line items are represented by `fulfilment_item` objects with per‟line status.

## 7. Versioning & Rollout

- Profile identifier: `acp.fulfilment.v1`.
- Backwards compatible: new events may be added, unknown types must be ignored.
- Add examples and conformance notes under `examples/`.
- Once merged, update `changelog/unreleased.md`.
