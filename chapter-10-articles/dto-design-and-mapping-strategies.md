# DTO Design and Mapping Strategies

## Learning Objectives

- Design data transfer objects as a stable transport shape, distinct from the domain objects they carry.
- Map between domain and DTO with a generated mapper, MapStruct, and control the JSON shape with Jackson annotations.
- Keep the DTO boundary from leaking, and recognize when a mapper and a serializer have turned into a mirror, not a design.

## Introduction

A DTO, data transfer object, is the thing your API actually moves. It is the class across the network, and it sets the shape you promised your callers, which is not the same as the shape of your domain object. Almost everything you learned in the previous chapter, the contract, the stable surface, gets real in the DTO, because the DTO is the contract made of fields.

This article is about that boundary: how to shape the DTO, how to move the domain into it without hand-writing the conversion, and how Jackson turns the DTO into the bytes on the wire.

## Problem Statement

The cheapest-looking choice is to skip the DTO and return the domain object. Spring will happily serialize your `Order` entity, with its lines, its lazy proxies, its framework-specific fields, straight into JSON for the client. It works on day one and it rots in a way you can feel.

It rots because the domain and the wire have separate lives. A domain object carries what the business needs, which is not what the client should see. The `Order` has a `Set<OrderLine>` you manage carefully, a `status` internal enum, a `customerId` back-reference, possibly a lazily-loaded relationship that blows up the moment you serialize it. Hand that whole object to Jackson and the client gets your implementation in JSON form, and you get to debug `LazyInitializationException` when the session is closed during serialization.

The deeper cost is coupling. Once a client is reading `order.supplier.pricing.structure`, your database schema is part of the spoken API, and you can never change a private field name without breaking the city of callers. The DTO exists precisely to absorb that. It is the one place the wire shape lives, so the domain can evolve without each change rippling outward.

## Core Concept

### The DTO is a different kind of thing

A DTO is a dumb, flat, serializable object. Plain final fields, getters, a constructor, maybe a small static factory. It carries no logic and no domain invariant. It exists to be transported and to be stable. That is the whole point: the DTO changes only when the API contract changes, and never otherwise, so the contract has a home that is not the domain. In effect it is a value object that belongs to the transport layer.

There is a naming convention that helps a real codebase. In a clean layered project the mapper lives in a mapping package, the DTOs live next to the controllers, and the domain stays in its own layer. If the mapper code lives in `com.example.api.dto` and the domain in `com.example.domain`, the dependency arrow is clear and testable. You can compile the transport package alone and only depend on the domain, never the other way around.

### Generating the mapping with MapStruct

Writing domain to DTO mappings by hand is a cold room full of boilerplate. Read a field off one object, assign it to a field of another, copy the identifier, repeat it across hundreds of types, and you have the most brittle code in the codebase, code that drifts out of sync the moment either class changes. The fix relies on this tool, a compile-time mapping generator.

```java
@Mapper(componentModel = "spring")
public interface OrderMapper {
    OrderDto toDto(Order order);
    Order toDomain(CreateOrderRequest request);
}
```

That interface is enough. MapStruct inspects the two types at compile time, matches fields by name and type, and generates the implementation class. Mismatched field names, the domain calls it `total`, the DTO calls it `amount`, get a `@Mapping`:

```java
@Mapping(target = "amount", source = "total")
OrderDto toDto(Order order);
```

MapStruct is useful precisely because the mapping goes through the compiler, and a renamed or missing field fails the build instead of failing a caller. It replaced a hand-written layer with generated code that cannot silently drift. The `componentModel = "spring"` part makes the generated implementation a Spring bean, so the controllers autowire `OrderMapper` and never look at the implementation.

### Jackson controls the wire shape

Once the DTO is built, Jackson serializes it to JSON and deserializes that JSON back into the DTO. The DTO is the contract, and Jackson annotations fine-tune how the contract appears on the wire.

The common annotations are small in number and huge in effect. `@JsonProperty` pins a field to an exact name, decoupling the Java field from the JSON key. `@JsonInclude(NON_NULL)` stops null fields from appearing, which matters on the wire when a payload must not contain `"createdAt": null`. Ordering and dates get `@JsonFormat`:

```java
public class OrderDto {
    @JsonProperty("id")
    private Long orderId;

    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'")
    private OffsetDateTime createdAt;

    @JsonInclude(JsonInclude.Include.NON_NULL)
    private String note;

    @JsonIgnore
    private long internalMarker;
}
```

Notice the DTO is still the source of truth. The `@JsonIgnore` hides an internal field from the client, `@JsonProperty` shows the wire name a client actually agreed to. Jackson is not doing the thinking about what the client should see, the DTO design is.

Put the pieces together and the pipeline is a clean three-hop:

Diagram: the domain object's journey to the wire

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 380" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#33475b"/>
    </marker>
  </defs>

  <rect x="40" y="140" width="200" height="96" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="140" y="180" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Order</text>
  <text x="140" y="204" text-anchor="middle" font-size="11" fill="#5a6b7a">domain object</text>

  <line x1="240" y1="180" x2="330" y2="180" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="285" y="168" text-anchor="middle" font-size="11" fill="#5a6b7a">MapStruct</text>

  <rect x="332" y="140" width="200" height="96" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="432" y="180" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">OrderDto</text>
  <text x="432" y="204" text-anchor="middle" font-size="11" fill="#5a6b7a">transport shape</text>

  <line x1="532" y1="180" x2="622" y2="180" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="577" y="168" text-anchor="middle" font-size="11" fill="#5a6b7a">Jackson</text>

  <rect x="624" y="140" width="200" height="96" rx="10" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="724" y="180" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">JSON</text>
  <text x="724" y="204" text-anchor="middle" font-size="11" fill="#5a6b7a">on the wire</text>
</svg>
```

The domain object is converted to the DTO by a generated mapper, and the DTO is serialized to JSON by Jackson. The reason for the middle box: a change to the domain never reaches the client, and a change to the contract lives only in the DTO and its mapping, the one seam.

## Real Production Usage

MapStruct is the workhorse of the Java backend for exactly this. It replaces a hand-written or reflection-based bean copy, and most Spring shops that run clean layers run MapStruct with `componentModel = "spring"`. Jackson is the serialization engine under Spring Boot's `@RestController` automatically, via `MappingJackson2HttpMessageConverter`, so every `@RequestBody` and return value filters through the Jackson you have configured.

The habit worth stealing is the separation of the two concerns. MapStruct answers "how do I map the domain to the DTO?" Jackson answers "how is the DTO represented as JSON?" The teams that blur them, putting `@JsonProperty` and mapping logic on the domain object, get a domain that knows the wire, and you are back to the coupling the DTO was supposed to kill.

## Common Mistakes

**Returning the entity, no DTO.** The `@GetMapping` that returns the `Order` entity straight up is the classic. It works, then breaks when a field or a lazy proxy serializes, and it welds your DB to your wire. Even a minimal DTO separates the two.

**Mapping by hand, everywhere.** A hand-written converter per type, or `BeanUtils.copyProperties`, hides rename errors and drifts out of sync. MapStruct's generated code is compile time, so the mismatch surfaces at build.

**DTOs that carry logic or domain state.** The moment the DTO gains methods, constraints, or re-exports an invariant, it is not a DTO, it is a domain dressed for transport, and the two are badly bundled in the wire.

## Interview Perspective

Interviewers digging at DTOs are testing the boundary, not the annotation. A weak answer says "the DTO is the object the API returns." A strong answer says the goal is protecting the domain from the wire, and the transport shape from the domain, that mapping is generated with MapStruct so it breaks on drift, and that Jackson is a separate concern layered over the DTO.

The follow-up that sorts people is "why the DTO at all, why not serialize the entity?" The strong answer walks you through lazy-loading serialization failing, the internal fields leaking to the client, and a schema change rippling outward to every caller. The candidate who says "MapStruct generates it" without mentioning what it protects is nailing the tool and missing the point it serves.

Common follow-ups:

- "A field is named differently in the DB and in the API wire. Where do you fix each?"
- "Your DTO and your entity are copied by hand. What breaks during a refactor, and how does the tooling prevent it?"

## Knowledge Check

1. Return the `Order` entity directly and a `@ManyToOne` relationship is lazily loaded. What happens when Jackson serializes it outside a session, and why does the DTO avoid that particular class of bug?
2. The domain renamed the aggregate method that computes a total, and the DTO field it feeds is now unused. How does MapStruct catch the drift, and what would a `copyProperties` approach have done instead?
3. Why does a clean layered shape put the DTO next to the controller and the domain in its own package, and what does that boundary protect you from?

## Key Takeaways

- The DTO is a stable transport shape, and the domain never has to match the wire.
- MapStruct generates the mapping at compile time, so the drift shows up as a build error, not a runtime surprise.
- Jackson serializes the DTO and the annotations control the JSON name, format, and omission.
- The DTO is what keeps a domain refactor from rippling out to every caller.
- Keeping the compile-time mapping and the wire annotations explicit makes the contract reviewable by anyone, without hunting the build output.

## What's Next

The DTO settles what a single exchange carries. The next article zooms into that exchange, request and response design, the actual shape of what the client sends and the rules for what the server answers with. It covers the request body, the location header, the representation you return, and why the pair has to hold together as one contract.

---

This article explains the DTO as a transport shape and the mapping, in MapStruct and shaped by Jackson, that turns a domain object into the wire. It argues the DTO keeps the domain and the contract apart, and skipping it returns the entity whole.