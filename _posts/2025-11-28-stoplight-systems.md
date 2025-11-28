---
layout: post
title: "Systems: Composition Roots for Stoplight 6.0"
date: 2025-11-28 00:00:00 +0000
---

In the Stoplight 6.0 roadmap, I outlined three major changes: enforcing configuration consistency, UTC timestamps, and 
cleaner configuration API. These improvements share a common foundation--an architectural pattern called systems.

This document explains the architecture behind those changes. If the roadmap answered "what problems are we solving?", 
this answers "why is this the right solution, and how does it work?"

## The Composition Root Pattern

The fundamental problem isn't configuration conflicts or test isolation--those are symptoms. The root issue is 
architectural: Stoplight lacks a composition root.

In dependency injection, the composition root is where you wire up your object graph. For example, in Rails, it's the 
Application class. These patterns exist because complex systems need a place where infrastructure, configuration, and 
application logic come together--a boundary that owns lifecycle and enforces consistency.

Stoplight has been missing this layer. We had pieces (data stores, lights, notifiers) but no architectural structure 
that owns their relationships. This creates a cascade of problems: if nothing owns the data store, how do you prevent 
accidental recreation? If nothing enforces configuration consistency, how do you cache safely? If nothing provides 
scope boundaries, how do you isolate tests?

Systems are Stoplight's composition root. The architectural foundation that makes everything else in 6.0 possible.

## How One Constraint Enables Everything

Systems enforce a single rule: **within a system, the same light name must have the same configuration**. This 
constraint cascades into multiple benefits.

**Benefit 1: Safe caching**

If configurations can differ, caching light instances is unsafe:

```ruby
Stoplight("api", threshold: 5)   # Cache this?
Stoplight("api", threshold: 10)  # Oops, they'd get threshold: 5
```

But if the system enforces consistency, the second call errors immediately. Now caching is safe--same name always means 
same instance. Result: ~5x performance improvement in hot paths.

**Benefit 2: Architectural prevention of bugs**

The Memory recreation trap:

```ruby
Stoplight("api", data_store: Memory.new)  # New instance
Stoplight("api", data_store: Memory.new)  # Different instance
```

Can't happen architecturally. The system owns ONE data store instance. Lights don't create stores--they receive them from
the system. The bug becomes impossible because the composition root owns infrastructure lifecycle.

**Benefit 3: Optimization opportunities**

Because the system creates lights, it can select optimal implementations based on configuration. A light with 
`traffic_control: :consecutive_errors` doesn't need sliding window maintenance. A light with 
`traffic_control: :error_rate` doesn't need consecutive failure tracking. The system can provide specialized 
implementations without conditional logic in the light itself.

Additionally, in-memory stores can use per-light coordination instead of one global lock, since each light receives 
its own store wrapper from the system.

This is elegant: the constraint doesn't just prevent bugs--it creates the architectural foundation for multiple 
optimizations. One rule, cascading benefits.

## Why Systems, Not Alternatives?

We explored several approaches before settling on systems as composition roots.

**Alternative 1: Namespaced configuration**

```ruby
Stoplight.configure(:payments) { |c| c.threshold = 3 }
Stoplight("api", namespace: :payments)
```

Configuration is still global but indexed by symbols. Easy to typo (`:payment` vs `:payments`). Where does the data 
store live? Who owns it? The symbol doesn't give you an object to work with. You can't pass `:payments` to a function 
and have it be meaningful. It's not first-class.

**Alternative 2: Per-light data stores with identity comparison**

```ruby
Stoplight("api", data_store: payment_redis)
Stoplight("api", data_store: payment_redis)  # Same instance?
```

How do you ensure both use the same instance? Object identity comparison (`store1.equal?(store2)`) is fragile. What 
about connection pools? Wrapped clients? Decorators like `redis-namespace`? This approach makes data store identity the 
programmer's problem, and object identity is not a reliable solution.

**Why systems won the design**

Systems provide:

- **First-class objects**: `MyApp::PAYMENTS` is a constant you can reference, pass around, mock in tests
- **Clear ownership**: The system owns the data store. One instance, explicitly provided to all lights
- **Encapsulation**: All payment configuration lives in one place--the system definition
- **Type safety**: Misspell `:payments` and it's a runtime error. Reference `MyApp::PAYMENS` and Ruby catches it at load time

The cost is one more abstraction layer. The benefit is architectural clarity: systems are the composition root, lights 
are what they compose.

## Implementation Mechanics

Here's how the core mechanism works:

```ruby
class System
  def light(name, **settings)
    normalized = normalize_settings(settings)
    
    @lights.compute(name) do |(existing_light, existing_settings)|
      if existing_light && !settings.empty? && normalized != existing_settings
        raise ConfigurationError, "Light '#{name}' exists with different config"
      end
      
      existing_light || light_factory.build_with(name: name, **settings)
    end
  end
end
```

This provides:

- **Thread-safe caching**: Uses `Concurrent::Map#compute` for atomic check-and-create
- **Conflict detection at creation**: Same name + different config = immediate error with both configs shown
- **The factory pattern**: System delegates to `LightFactory` which knows how to wire up lights with the system's infrastructure.
  You can use different factories for different use-cases. For instance, load configuration from a persisted data store.
- **Configuration normalization**: Settings are sorted into canonical form (`{threshold: 5, window: 60}` matches `{window: 60, threshold: 5}`)

The system doesn't manage light internals--it manages the composition. It owns infrastructure (data store, notifiers), 
enforces consistency (conflict detection), and coordinates light creation (factory delegation).

## The Default System

For applications needing one data store and one set of defaults, the global API continues working:

```ruby
Stoplight.configure do |config|
  config.data_store = redis
  config.threshold = 5
end

Stoplight("api").run { call_api }
```

Internally:

```ruby
module Stoplight
  def self.default_system
    @default_system ||= System.new(:default, **configuration.to_h)
  end
end

def Stoplight(name, **settings)
  Stoplight.default_system.light(name, **settings)
end
```

The factory method delegates to the default system. Simple applications get all the benefits without seeing systems 
at all. Complex applications opt into explicit systems when they need architectural boundaries.

For tests, `with_default_system` temporarily replaces the default:

```ruby
RSpec.around do |example|
  test_system = Stoplight.system(:test, data_store: Memory.new)
  Stoplight.with_default_system(test_system) { example.run }
end
```

Every light created during the test--by your code, by library internals, by code under test--uses the test system's isolated store.

## Scaling Patterns

**Multi-tenant platforms:**

```ruby
class TenantCircuits
  def self.for(tenant_id)
    Stoplight.system(tenant_id, data_store: tenant_redis(tenant_id))
  end
end

TenantCircuits.for("acme").light("api").run { ... }
```

Each tenant gets isolated circuit breakers. Failures in one tenant don't cascade to others.

**Microservices with different SLAs:**

```ruby
module Services
  CRITICAL = Stoplight.system(:critical, threshold: 3, cool_off_time: 30)
  STANDARD = Stoplight.system(:standard, threshold: 5, cool_off_time: 60)
  BEST_EFFORT = Stoplight.system(:best_effort, threshold: 10, cool_off_time: 120)
end

Services::CRITICAL.light("payment_gateway").run { ... }
Services::BEST_EFFORT.light("recommendations").run { ... }
```

**Gradual migration:**

```ruby
# Phase 1: Keep using global
Stoplight("api").run { ... }

# Phase 2: Create system, migrate one subsystem
MyApp::PAYMENTS.light("stripe").run { ... }

# Phase 3: Migrate remaining subsystems over time
```

No big-bang rewrite required. Systems compose naturally as your architecture grows.

## Design Questions

Three areas where I want feedback:

**1. Naming**

Is "system" the right term? Alternatives considered:
- `Stoplight.scope(:payments, ...)` / `PaymentsScope.light(...)`
- `Stoplight.context(:payments, ...)` / `PaymentsContext.light(...)`
- `Stoplight.boundary(:payments, ...)` / `PaymentsBoundary.light(...)`
- `Stoplight.namespace(:payments, ...)` / `PaymentsNamespace.light(...)`

"System" conveys "subsystem with shared infrastructure" but might overload a common word. Which reads most naturally?

**2. Configuration override rules**

Should lights override system-level observability? For example:

```ruby
MyApp::PAYMENTS.light("stripe", notifiers: [CustomNotifier.new])
```

My instinct: no. Observability is infrastructure--it belongs at the composition root. If you need different notifiers, 
create a different system. But I can imagine edge cases where per-light notifiers make sense.

**3. Global configuration**

Should global configuration affects systems:

```ruby
Stoplight.configure do |config|
  config.data_store = Stoplight::DataStore::Memory.new
end
```

No doubt it affects the default system, but what about secondary systems:

```ruby
Payments = Stoplight.system(:payments)
```

should `Payments` inherit default data store? My intuition: yes, it should. This would be convenient for users
who want just name isolation, but shared defaults. 

**4. System caching**

Should it be possible to request the same system twice (similar to lights):

```ruby
payments1 = Stoplight.system(:payments)
payments2 = Stoplight.system(:payments) # is it the same system? 
```

I think it should not, this will encourage the same set of problems we already have with light creation--config uniqueness,
data store identity, etc. Instead, for the systems API I suggest explicit interface:

```ruby
Payments = Stoplight.system(:payments) # assign it to a constant
Payments.light("api") # use later
```

**5. Performance validation**

The caching assumes light creation is expensive enough to justify cache lookup overhead. I expect ~5x improvement based 
on profiling, but need benchmarks showing:

- Light creation without caching (baseline)
- Light creation with caching (6.0)
- Cache lookup overhead for repeated access

If numbers don't justify it, systems still solve composition root problems, but we'd reconsider performance claims.

## Timeline & Next Steps

**Q4 2025**: Prototype implementation, dogfood in Stoplight's own test suite and fail-safe mechanisms  
**Q1 2026**: 6.0 release with systems as foundation

I want to use systems internally first--protecting Stoplight's own Redis operations, isolating test suites--and 
discover what breaks before committing to the public API.

If you have production use cases where systems would solve problems you're facing today, let's talk. This design 
isn't final. The best way to validate it is through real usage against real constraints.

Share your thoughts in our discussion forum or open a GitHub issue. I'm particularly interested in feedback on the 
design questions above--these decisions shape the API you'll use in 6.0.

[Stoplight 6.0 roadmap]: https://blog.bolshakov.dev/2025/11/27/stoplight-6-roadmap.html
