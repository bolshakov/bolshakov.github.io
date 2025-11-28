---
layout: post
title: "Stoplight 6.0: Fixing Legacy Design Decisions"
date: 2025-11-27 00:00:00 +0000
---

I want to talk through some changes we're planning for Stoplight 6.0 and get your feedback before we finalize anything.
There are three design decisions from Stoplight's early days that keep causing problems in production. They're not bugs 
exactly -- more like compromises that made sense when the library was younger but now create subtle issues at scale. 

We've been working around them for years, but I think it's time to fix them properly. They all require breaking changes
but the migrations are straightforward, mostly mechanical adjustments. Let me walk through what we're thinking and why.

**Target release**: Q1 2026 (before March DST transitions)
**Ruby support**: 3.2+ through 6.x series

## Enforcing Configuration Consistency

Here's something that's been possible since early versions:

```ruby
Stoplight("payment-api", threshold: 5)

# Somewhere else in the codebase
Stoplight("payment-api", threshold: 10)
```

Both lights share the same name and internal state, but they expect different behavior. One thinks it should trip 
at 5 errors, the other at 10. This results in unpredictable decisions. Sometimes the circuit trips early, sometimes 
late. You can spend hours debugging what looks like flaky circuit behavior before discovering the real issue: conflicting
configurations.

Here's another problematic pattern we've seen:

```ruby
light1 = Stoplight("api", traffic_control: :consecutive_errors, threshold: 5)
light2 = light1.with(traffic_control: :error_rate, threshold: 0.3)
```

Both lights share the same underlying data structures, but they interpret that data completely differently. 
One counts consecutive failures ("has it failed 5 times in a row?"), the other calculates error rates ("are 30% of 
recent requests failing?"). Their behaviors interfere with each other, making decisions less predictable. 

There's one more catastrophic trap that's easy to fall into:

```ruby
# This looks reasonable but is completely broken
Stoplight("api", data_store: Stoplight::DataStore::Memory.new).run { call_api }
Stoplight("api", data_store: Stoplight::DataStore::Memory.new).run { call_api }
```

Each call creates a *brand new* Memory store instance. The circuit has no memory at all-every call starts from scratch 
with an empty store. Your circuit will never trip because failures never accumulate. You're paying the overhead of a 
circuit breaker with zero protection.

We want to make these errors: `Stoplight::Error::ConfigurationError` raised immediately when you try to create a 
light with a name that already exists but different settings. Same name must mean identical behavior. No ambiguity, 
no surprises.

- `Stoplight::Light#with` is removed (it encouraged dangerous config mutation)
- Same circuit name = same configuration, guaranteed and enforced
- Configuration conflicts raise `Stoplight::Error::ConfigurationError` immediately

### The Performance Benefit

Beyond preventing bugs, this constraint unlocks something powerful: **we can now safely cache light instances**.

Think about it. If every call to `Stoplight("api")` is guaranteed to produce the same configuration, we can create the 
light once and reuse it. Previously, this was impossible:

```ruby
Stoplight("api", threshold: 5)   # First call
Stoplight("api", threshold: 10)  # Different config!
```

If we cached the first instance, the second call would silently get `threshold: 5` instead of the 
requested `threshold: 10`. That's worse than not caching at all.

But with configuration conflicts prohibited, caching becomes safe:

```ruby
Stoplight("api")  # Creates light, caches instance
Stoplight("api")  # Returns cached instance
Stoplight("api")  # Returns cached instance
```

This means the most common Stoplight pattern just got dramatically faster:

```ruby
def pay_order
  Stoplight("api").run(fallback) do 
    call_payment_gateway
  end
end
```

Every call to `pay_order` used to create a new light instance—parsing configuration, setting up data store connections, 
initializing notifiers, building the execution strategy. Now? After the first call, it's essentially free. 
**Result: approximately 5x performance improvement** in your hottest code paths, with zero code changes required.

Of course, you can still define lights once and reuse them explicitly if you prefer:

```ruby
def initialize
  @api_circuit = Stoplight("api")
end

def pay_order
  @api_circuit.run(fallback) { call_payment_gateway }
end
```

But you don't have to. The inline syntax now has the same performance characteristics.

This is what we mean by constraints enabling optimizations. By enforcing single configurations per name, we 
simultaneously eliminate a class of bugs *and* unlock significant performance gains. 

### Why Performance Matters

Most Stoplight use cases wrap network calls—payment gateways, external APIs, database queries. When the protected 
operation takes 50-200ms, Stoplight's overhead is noise in comparison.

But Stoplight is a general-purpose library. We want to unlock use cases where every millisecond counts:

- **Internal microservices** where you're wrapping gRPC calls with <5ms latencies. A 0.8ms circuit breaker overhead on 
  a 3ms service call is 26% of your operation time.
- **Cache layers** protecting Redis or Memcached operations that complete in sub-millisecond timeframes. When you're 
  measuring operations in microseconds, overhead becomes a real bottleneck.
- **High-throughput background jobs** processing millions of operations per hour. Even small per-operation overhead 
  compounds: 0.8ms × 1,000,000 = 13+ minutes of pure overhead per hour.

We want to enable use cases we haven't imagined yet. If someone wants to wrap individual database queries or protect 
hot cache paths, Stoplight shouldn't be their bottleneck. Performance isn't everything, but it's table stakes for a 
general-purpose reliability library.

The 5x improvement from instance caching is just one piece. We're also refactoring data stores to use focused 
strategy objects instead of monolithic classes with nested conditionals. This unlocks further performance gains, 
especially under high concurrency. But that's internal implementation detail—what matters to you is that the library 
gets faster without you changing any code.

### What the Migration Will Look Like

When 6.0 ships, upgrading will be straightforward: update the gem, run your test suite, and look for `ConfigurationError` 
exceptions. Each error points to a light name being used with different configurations.

The fix depends on intent. If the differences are intentional-you genuinely need different behavior-use distinct names 
that reflect the purpose:
```ruby
# Different names for different tolerance levels
Stoplight("payment-api:checkout", threshold: 5)
Stoplight("payment-api:refund", threshold: 10)
```

If the differences are accidental—copy-paste errors or defaults that drifted-consolidate to shared configuration:

```ruby
# Extract to prevent configuration drift
EMAIL_CIRCUIT_CONFIG = { threshold: 5, window_size: 300 }
Stoplight("email-service", **EMAIL_CIRCUIT_CONFIG)

# Or reuse light instance
@email_service_light = Stoplight("email-service", threshold: 5, window_size: 300)
```

If you're using `#with` for variations, you likely want distinct lights with clear names. 

The errors are loud and immediate-you'll find them in your first test run, not in production. We estimate most 
applications will need no more than 1 hour to audit and consolidate configurations.

## No More DST Surprises

Circuit breakers shouldn't behave differently when clocks change. Yet that's exactly what happens when you use local 
time for timestamp-based metrics.

Picture this: it's 2:00 AM on March 8, 2026. Your lights have 5 minutes of accumulated metrics. Clocks spring forward 
to 3:00 AM. Now you're comparing error rates across a discontinuity--some buckets are "in the past" by an hour, 
others aren't. The math breaks.

Or consider distributed deployments: US transitions March 8, Europe transitions March 29. For three weeks, coordinating 
light state across regions means dealing with servers in different DST states looking at the same moments in time.

The fix is simple: use UTC for all internal timing. UTC doesn't have DST transitions. It's the same everywhere, 
always. No discontinuities, no cross-region coordination issues.

### Why This Is a Breaking Change

Timestamps are embedded in Redis keys to organize time-bucketed metrics. A key might look like:

```
stoplight:v5:metrics:api:1709884800
```

That number is a Unix timestamp representing a specific time bucket. Switching from local time to UTC means these 
timestamps change. It's like changing your database's primary key structure—the data won't automatically migrate.

The good news is that Redis keys auto-expire, so the old keys automatically disappear. The old and new versions will use 
different key namespaces (`v5` vs `v6`). You can safely run them side-by-side during deployment. They won't interfere 
with each other.

### What to Expect After Upgrading

When you upgrade, your circuits start fresh:
- No accumulated error counts
- All circuits begin in the `green` (closed) state
- Recovery and failure thresholds reset

For most applications, this is fine. Circuit breakers are designed to adapt quickly--that's their whole purpose. Within 
minutes of real traffic, they'll have enough data to make good decisions.

But if you're protecting critical services where even a brief learning period matters, plan your upgrade during a 
low-traffic window. Give your circuits time to accumulate data before peak hours hit.

## Cleaner Configuration API

With configuration conflicts prohibited and instance caching enabled, we can now clarify what gets configured where. 
Right now, the `Stoplight()` factory accepts everything at once, mixing concerns:

```ruby
Stoplight("api",
  threshold: 5,              # Behavioral: affects circuit decisions
  window_size: 300,          # Behavioral: affects circuit decisions
  notifiers: [SlackAlert],   # Observability: where to send alerts
  tracked_errors: [ApiError] # Runtime: varies per operation
)
```

This conflates settings that operate at completely different layers. Let me break it down:


### Behavioral Settings: Define Light Behavior

These define how a circuit makes decisions. They're fundamental to what the circuit *is*

- `cool_off_time` - How long to wait before attempting recovery
- `threshold` - How many failures trigger the circuit
- `recovery_threshold` - How many successes needed to recover
- `window_size` - Time window for tracking failures
- `traffic_control` - Strategy (consecutive vs. error rate)
- `traffic_recovery` - Recovery strategy

These stay at the light level: `Stoplight("api", threshold: 5, window_size: 300)`

Think of these as defining what circuit is--how cautious it is, how quickly it recovers, what time window it considers. 
Change these, and you've fundamentally changed how the circuit behaves.

### Observability Settings: Monitor Light Health

These are about monitoring and don't change light behavior:

- `error_notifier` - Called when the data store itself fails
- `notifiers` - Called on state changes (open/closed transitions)

These stays into global configuration only:

```ruby
Stoplight.configure do |config|
  config.error_notifier = ->(error) { Sentry.capture_exception(error) }
  config.notifiers = [SlackNotifier.new]
end
```

You typically set these once for your entire application. They're about "where do I send alerts?" not "how does this 
light behave?"

### Runtime Settings: Vary Per Execution

These change based on the specific operation you're protecting:

- `fallback` - What to do when the circuit is open
- `tracked_errors` - Which errors to track
- `skipped_errors` - Which errors to ignore

These move to `#run`:

```ruby
def purchase
  Stoplight("payment-api").run(->(error) { purchase_using_fallback_gateway }, tracked_errors: [PaymentGatewayError]) do
    # main gateway
  end
end

def refund
  Stoplight("payment-api").run(->(error) { refund_using_fallback_gateway }, tracked_errors: [RefundError]) do
    # main gateway
  end
end
```

Same circuit protecting the same underlying service, but purchases and refunds need different fallback behavior. Refunds 
might also care about `RefundError` while purchases don't. This is operation-specific context, not circuit identity. It 
belongs at the call site, not in the light definition.

### Why This Change Matters

This isn't just about organization--it's about making impossible states impossible. When notifiers were configurable 
per-light, you could accidentally create this:

```ruby
# In one file
Stoplight("api", notifiers: [SlackNotifier])

# In another file  
Stoplight("api", notifiers: [PagerDutyNotifier])
```

Now you have two configurations for the same light, and they conflict. By moving notifiers to global configuration, 
this entire class of problem disappears.

Similarly, when `tracked_errors` was configurable at light creation, you'd be tempted to create separate lights for 
the same service just to track different error types. That's backwards--the service protection is the same, only the 
operation context differs.

### What Changes You'll Need to Make

When 6.0 ships, you'll need to:

1. Move notifier configuration from individual lights to `Stoplight.configure`
2. Move `tracked_errors` and `skipped_errors` from light creation to `#run` calls
3. Remove any `#with` or `#with_*` method calls (replaced by distinct light names where needed)

These are mechanical changes. We'll provide deprecation warnings in 5.x releases so you can migrate incrementally 
before 6.0 lands. The core execution pattern—`Stoplight("name").run(fallback) { block }`—remains unchanged.


## Why Bundle These Changes?

The UTC switch is our forcing function. We're going to break Redis compatibility anyway to fix the DST issues. New key 
schemas mean circuits lose their history regardless.

If we're asking you to upgrade and accept that cost, we might as well fix the other legacy issues at the same time. 
The alternative would be having multiple breaking releases other the next year. One upgrade beats three.

The changes aren't dramatic. Configuration audit for conflicts, move notifiers to global config, pass `tracked_errors` 
to `#run` instead of light creation. Most codebases will need a few hours of work, not days.

### DST Timeline Creates Urgency

March 2026 sees DST transitions two weeks apart: US on March 8, Europe on March 29. This creates a complex testing 
window where some production systems are in DST while others aren't.

We want to release UTC support *before* these transitions to avoid:
- Users experiencing metric discontinuities
- Emergency patches during DST weekends
- Another year of "why did my circuit breaker act weird when clocks changed?"

This is the last good window before another full year of DST complexity.

### Configuration Conflicts Are Silent Footguns

We've been debugging mysterious circuit behavior ourselves, only to discover conflicting configurations: 

- "Why does my circuit trip at 3 errors sometimes and 5 errors other times?"
- "My window size seems to change randomly"
- "Circuits behave differently in tests vs. production"

All caused by inadvertent configuration conflicts. Better to fix it now than let more users discover it the hard way.

## What's Not Changing

Core Stoplight patterns remain stable:

- Light creation syntax stays familiar: `Stoplight("name", **options)`
- Execution remains unchanged: `light.run(fallback) { block }`
- Global configuration via `Stoplight.configure` works the same way
- Data store configuration interface for stays compatible
- Admin UI continues working

We're enforcing better practices and removing footguns.

## A Note on Architecture

Sharp-eyed readers will notice I haven't mentioned how we're preventing the Memory store recreation bug or how we're 
handling data store configuration. We're introducing an internal architecture called "systems" that provides clean 
composition roots for circuit breakers.

For 95% of users, this is invisible--the default behavior handles everything through global configuration. But if you're 
building multi-tenant platforms, microservices with different SLAs, or need guaranteed test isolation, systems solve 
real architectural problems.

I'll write about systems separately once we've validated the design in practice. For now, know that the architecture 
enables everything described here while remaining completely optional for most use cases.

## Timeline and Your Feedback

We're targeting Q1 2026 release, ideally before the March 8 DST transition. That gives us Q4 2025 to validate the 
systems architecture internally and ship deprecation warnings in the 5.x series. It gives you 2-3 months to see the 
warnings in your applications and plan your upgrade.

But this is the planning stage. We're finalizing these decisions through Q4 2025, and your production experience 
matters more than our theoretical designs.

### Questions I have for you:

1. Are we missing legitimate use cases for `#with`? Some pattern where mutation actually makes sense that we 
  haven't considered?
2. Is moving notifiers to global configuration going to break your setup? Are you doing something clever with 
  per-circuit notifiers that we should preserve?
3. Does the bundling make sense? Or would you rather take these changes separately even if it means multiple upgrades 
  over the next year?
4. What's your upgrade timeline like? If we ship Q1 2026 with deprecations starting Q4 2025, is that realistic for
  you to test and deploy?
5. Do you have circuits protecting critical services where a brief learning period after upgrade would be problematic? 
  We want to understand these use cases to provide better guidance.

Share your thoughts in our [discussion forum] or open a GitHub issue. The whole point of this writeup is to hear what
we're missing before we commit to these changes. Your production experience shapes what we build.

**Tëma Bolshakov**  
*Stoplight Maintainer*


[discussion forum]: https://github.com/bolshakov/stoplight/discussions/533
