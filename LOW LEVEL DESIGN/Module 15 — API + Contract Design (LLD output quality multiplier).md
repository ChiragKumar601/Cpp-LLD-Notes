
## Intuition

Good LLD isn’t just classes; it’s **stable contracts**:
- clean request/response shapes    
- versioning strategy
- idempotency and safe retries
- pagination/filter/sort that won’t break later
- DTO vs domain separation

**Note** - A **DTO** (Data Transfer Object) is a simple object/struct used to **carry data between layers or across the network**.

In interviews, this is where you show “production thinking”.


# 15.1 Designing public APIs: minimal surface, stable contracts

## Intuition

A public API is a promise. Every extra field/method increases long-term maintenance.

## Concepts (rules)

- Prefer **few endpoints** with clear nouns/verbs
- Make contracts **explicit**: required vs optional, error codes, invariants
- Avoid leaking internal models (DB fields, internal IDs)
- Prefer **compatibility-friendly** evolution: add optional fields, don’t rename/remove
- Use consistent naming, consistent error structure

## Practical example

Bad:
- `POST /doEverything`  

Good:
- `POST /orders`
- `GET /orders/{id}`

## Real-world use cases

- internal service APIs
- public SDKs
- plugin contracts 

## Interview questions

**Q. How do you avoid breaking clients when evolving APIs?**
Keep changes backward-compatible: **add optional fields**, don’t rename/remove, keep error shapes stable, and use explicit versioning only for breaking changes (v1/v2).

**Q. What do you keep out of public contracts?**
Internal/unstable details like **DB schema fields**, internal IDs/shard keys, implementation-specific flags, internal state machine internals, and infra concerns (retries/timeouts/trace IDs as required fields).


# 15.2 Versioning strategy, backward compatibility

## Intuition

APIs evolve; clients lag.

## Concepts

**Common strategies**
- URL versioning: `/v1/orders`
- header versioning: `Accept: application/vnd.foo.v2+json`
- field-level compatibility: add optional fields

**Compatibility rules**

- ✅ Add optional fields
- ✅ Add new endpoints
- ✅ Make validation stricter only with caution
- ❌ Remove/rename fields
- ❌ Change meaning of a field silently

## Practical example

- `discount` added as optional field: safe
- changing `amount` from “paise” to “rupees”: breaking

## Interview questions

**Q. What changes are backward compatible?**
Adding new **optional** fields/endpoints, adding enum values cautiously, and loosening validation (accept more inputs) are usually safe; renaming/removing fields, changing meanings, or making optional → required is breaking.

**Q. How do you deprecate safely?**
Announce deprecation, keep old behavior working for a window, add a replacement (`v2`/new field), support both in parallel, log/monitor usage, then remove only after clients migrate.


# 15.3 Pagination, filtering, sorting at LLD level

## Intuition

Lists scale. Your API must support retrieving subsets predictably.

## Concepts

**Pagination styles**
- Offset pagination: `?page=2&size=50` (“give me 20 orders starting from the 41st.”)
	- **Pros:** simple to implement and easy for UI “page number” (`page=3`) 
	- **Cons:** gets slow at large offsets (DB has to skip a lot), and results can **shift** if new rows are inserted/deleted—so users may see duplicates/missed items while paging.

- Cursor pagination: `?cursor=abc&limit=50`(“give me the next 20 after (createdAt, id)”.)
    - **Pros:** fast and stable for infinite scroll; avoids duplicates/misses much better when data changes
    - **Cons:** a bit more complex; not great for “jump to page 10”; you must define a **consistent sort order** and use a tie-breaker (e.g., `createdAt DESC, id DESC`).

**Filtering/sorting**
- define allowed fields explicitly (avoid arbitrary SQL exposure)
- validate sort keys and directions

## Practical example (cursor)

```bash
GET /orders?limit=50&cursor=opaqueToken
-> returns { items: [...], nextCursor: "..." }
```

## Real-world use cases

- activity feeds
- admin dashboards
- search results

## Interview questions

**Q. Offset vs cursor—trade-offs?**
Offset gets slow at large offsets (DB has to skip a lot), and results can **shift** if new rows are inserted/deleted—so users may see duplicates/missed items while paging. Cursor is a bit more complex; not great for “jump to page 10”; you must define a **consistent sort order** and use a tie-breaker (e.g., `createdAt DESC, id DESC`)

**Q. How do you prevent clients from sorting on any DB column?**
Expose an **allowlist** of supported `sortBy` keys, validate direction (`asc|desc`), **map** those keys to approved internal/indexed fields, and **reject** anything else to prevent arbitrary DB-column sorting and performance abuse.

# 15.4 Idempotent commands and safe retries

## Intuition

Clients retry. Networks fail. If writes aren’t idempotent, you get duplicates.

## Concepts

- For write endpoints: accept `Idempotency-Key` header
- Store (key → result) and return same result on retry
- For PUT vs POST:
    - `PUT /resource/{id}` is naturally idempotent if designed correctly
    - `POST` needs explicit idempotency key

## Practical example

- `POST /payments` with `Idempotency-Key: xyz`
- second call returns same `paymentId` and status

## Interview questions

**Q. How do you implement idempotency keys?**
For write endpoints we accept a Idempotency-Key header and store(key->result) and return same result on retry. Key should be scoped (per user/endpoint) and stored atomically with the write.

**Q. What TTL do you pick and why?**
 Pick a TTL that covers realistic retry windows—**typically 24 hours** for payments/orders—so late retries still dedupe, but storage doesn’t grow forever (shorter like 1–2 hours for low-risk ops, longer if clients can retry later).


# 15.5 DTO vs domain separation

## Intuition

Domain model ≠ API model. If you mix them, you lock your core to external clients.

## Concepts

- **DTO**: shaped for transport (API), versionable, may be optional fields
- **Domain**: shaped for invariants (strict), cohesive, no “maybe null” everywhere
- Mapping happens in application layer / controllers.

## Practical example

- `CreateOrderRequestDTO` → validate → `CreateOrderCommand` → `Order` aggregate

## Why it’s valuable in real systems

- You can evolve the public API (add optional fields) without breaking core logic.
- Different clients can have different DTO shapes (web vs partner) mapping to the same domain.
- Domain stays stable while outside contracts change.
## Real-world use cases

- evolving public API without rewriting domain
- integrating multiple client types (web/app/partners)

## Interview questions

**Q. Why not expose domain objects directly?**
Because external clients need different shapes and versioning, and exposing domain directly **leaks internals/invariants** and locks your core model to the public API.

**Q. Where should mapping live?**
In the **boundary layer**: controllers or the application/use-case layer (DTO ↔ command/domain), not inside domain entities.