# HTTP DTO Contract: ProblemDetail (RFC 7807)

The canonical HTTP error response. Hand-written Java DTO (Jackson-annotated) in
`shared-contracts/src/main/java/com/ecommerce/contracts/error/ProblemDetail.java`.

Media type: `application/problem+json`

```json
{
  "type": "https://errors.ecommerce.example/product-not-found",
  "title": "Product not found",
  "status": 404,
  "detail": "No product exists with id 5f1c...",
  "instance": "/products/5f1c...",
  "sku": "ABC-123"
}
```

| Member | JSON type | Required | Notes |
|--------|-----------|----------|-------|
| type | string (URI) | no (defaults to "about:blank") | problem type identifier |
| title | string | yes | short human-readable summary |
| status | number | yes | HTTP status code |
| detail | string | no | occurrence-specific explanation |
| instance | string (URI) | no | identifies the specific occurrence |
| *(extensions)* | any | no | additional problem-specific members (e.g., `sku`) |

All HTTP error responses across the platform MUST conform to this shape (constitution
Principle II). HTTP request/response DTOs for product and order resources are documented in
`shared-contracts/api-contract.md`.
