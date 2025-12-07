# API Patterns

## Quick Decision Matrix

| If you need... | Use | Avoid |
|----------------|-----|-------|
| Simple CRUD, broad client support | REST | gRPC |
| Flexible queries, mobile bandwidth optimization | GraphQL | REST with many endpoints |
| High performance microservices, streaming | gRPC | REST |
| Public API with third-party consumers | REST | gRPC |
| iOS app with complex nested data | GraphQL | Multiple REST calls |

---

## REST

| Aspect | Details |
|--------|---------|
| **Best for** | CRUD operations, public APIs, simple data models |
| **Pros** | Universal support, cacheable (HTTP caching), simple to understand, great tooling, stateless |
| **Cons** | Over-fetching/under-fetching, multiple round trips for nested data, no built-in schema |
| **When to use** | Public APIs, simple applications, when caching is critical, broad client support needed |
| **When to avoid** | Complex nested data requirements, bandwidth-sensitive mobile apps |
| **Works well with** | HTTP/2, JSON, OpenAPI/Swagger, any client |
| **iOS considerations** | URLSession native support, easy to implement, may need multiple calls for complex screens |

### REST Best Practices
- Use nouns for resources (`/users`, `/orders`)
- HTTP verbs for actions (GET, POST, PUT, PATCH, DELETE)
- Version in URL (`/v1/users`)
- Consistent error format with status codes
- HATEOAS for discoverability (optional)

---

## GraphQL

| Aspect | Details |
|--------|---------|
| **Best for** | Complex UIs, mobile apps, multiple client types with different data needs |
| **Pros** | Single endpoint, client specifies exact data needed, strongly typed schema, reduces over-fetching |
| **Cons** | Caching complexity, N+1 query risk on backend, learning curve, overkill for simple APIs |
| **When to use** | Mobile apps (bandwidth matters), complex nested data, multiple frontend clients |
| **When to avoid** | Simple CRUD, file uploads (needs workaround), real-time heavy (use subscriptions carefully) |
| **Works well with** | Apollo (client/server), Relay, DataLoader (N+1 prevention) |
| **iOS considerations** | Apollo iOS client, excellent for reducing network calls, single request for complex screens |

### GraphQL Best Practices
- Use DataLoader to batch database queries
- Implement query depth/complexity limits
- Persisted queries for iOS (smaller payloads, security)
- Separate queries and mutations clearly
- Use fragments for reusable field sets

---

## gRPC

| Aspect | Details |
|--------|---------|
| **Best for** | Microservices communication, high-performance internal APIs, streaming |
| **Pros** | Binary protocol (fast, small), HTTP/2 native, streaming support, strongly typed (protobuf) |
| **Cons** | No browser support (needs proxy), harder to debug, less human-readable |
| **When to use** | Service-to-service communication, real-time streaming, performance-critical paths |
| **When to avoid** | Public APIs, browser clients, simple applications |
| **Works well with** | Kubernetes, microservices, Protocol Buffers |
| **iOS considerations** | grpc-swift available, good for high-frequency updates, adds complexity vs REST |

### gRPC Best Practices
- Use for internal services, REST/GraphQL for client-facing
- Implement health checks
- Use deadlines/timeouts
- Consider gRPC-Web for browser support if needed

---

## API Versioning Strategies

| Strategy | Pros | Cons | Best For |
|----------|------|------|----------|
| URL path (`/v1/`) | Explicit, easy routing | URL clutter | Public APIs, major versions |
| Header (`Accept-Version: v1`) | Clean URLs | Less visible | Internal APIs |
| Query param (`?version=1`) | Easy to add | Can be missed | Quick iterations |

**Recommendation**: URL path for public/mobile APIs (explicit, cacheable).

---

## Mobile API Optimization

| Technique | Benefit | Implementation |
|-----------|---------|----------------|
| Pagination | Reduces payload size | Cursor-based preferred over offset |
| Field filtering | Client requests only needed fields | `?fields=id,name,email` |
| Compression | Reduces bandwidth | gzip/brotli on responses |
| ETags | Conditional requests | Return 304 if unchanged |
| Batch endpoints | Reduces round trips | `POST /batch` with multiple operations |

---

## iOS + Backend API Checklist

- [ ] API designed for offline-first (idempotent operations, conflict resolution)
- [ ] Pagination on all list endpoints
- [ ] Consistent error format with error codes
- [ ] Authentication via Bearer token in header
- [ ] Token refresh endpoint
- [ ] API versioning strategy decided
- [ ] Response compression enabled
- [ ] Request/response logging for debugging
