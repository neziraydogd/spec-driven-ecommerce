# Contracts (design sketches)

These files are **design-time sketches** that show the intended shape of the published
contracts. The authoritative implementations will live in `shared-contracts/src/main/proto/`
(proto3) and `shared-contracts/src/main/java/.../http/` (HTTP DTOs) once implemented.

Scope reminder (clarification Q1=A): **define-and-publish only**. The proto `service`
definitions below declare gRPC interfaces as *contracts*; no servers or clients are
implemented in this feature. Event messages are proto3 payloads for Kafka — no
producers/consumers and no runtime Schema Registry are created here.

| File | Purpose | Channel |
|------|---------|---------|
| `common/problem_details.proto` | RFC 7807 error surface (proto mirror) | gRPC / events |
| `product/product.proto` | Product message + gRPC service | gRPC |
| `product/product_events.proto` | Stock movement + product event payloads | Kafka |
| `order/order.proto` | Order/OrderLine messages + gRPC service | gRPC |
| `order/order_events.proto` | Order state-changed event payload | Kafka |
| `http/problem-detail.md` | RFC 7807 HTTP DTO contract | HTTP |

All `.proto` files are checked by `buf lint` and `buf breaking` in CI. The HTTP DTO surface
and every gRPC/event contract are documented in `shared-contracts/api-contract.md`.
