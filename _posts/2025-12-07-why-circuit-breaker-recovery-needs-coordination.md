---
layout: post
title: "Why Circuit Breaker Recovery Needs Coordination"
date: 2025-12-06 00:00:00 +0000
---

Every circuit breaker tutorial shows the same state machine. CLOSED means healthy, OPEN means failing, HALF-OPEN means 
testing recovery. Three states, a few transitions, done.

The state machine is trivial. The coordination problem hiding beneath it is not.

The moment you have concurrent execution--threads in a single process, workers across multiple servers--recovery becomes 
a coordination problem. Multiple actors racing to probe, racing to evaluate, racing to transition. Without coordination, 
your carefully configured recovery threshold becomes a suggestion.

This post walks through how we reasoned about this problem in [Stoplight], and why we landed on serialized recovery 
probes as the solution.

## The problem: uncoordinated recovery

Half-open state exists to answer a question: has the downstream service recovered? You send a probe request, observe 
the result, decide whether to restore traffic.

The implicit assumption in every state diagram: one probe at a time. One evaluation. One decision.

With concurrent execution, this assumption breaks immediately.

### Thundering herd

Your payment service fails at 10:00:00. The circuit opens. You've configured a 60-second cool-off, so at 10:01:00 the 
circuit becomes eligible for probes.

At 10:01:01, fifty workers check the circuit state. All fifty see "cool-off expired, eligible for probe." All fifty 
send requests to the payment service.

If the service is still struggling, all fifty fail. Fifty users get errors. Fifty transactions time out. Fifty retries 
queue up. The cascade you installed a circuit breaker to prevent just happened--during recovery testing.

This isn't hypothetical. Any multithreaded or distributed circuit breaker without coordination has this problem. The 
thundering herd happens precisely when it hurts most: against a service that's trying to recover.

### Racing decisions

[Thundering herd] is obvious. The subtler problem is what happens when concurrent probes return different results.

Service is flaky during recovery--some requests succeed, some fail. Two workers probe simultaneously:

```
10:01:01.000  Worker A probe: SUCCESS → decides CLOSED
10:01:01.000  Worker B probe: FAILURE → decides OPEN
10:01:01.050  Worker A writes: state=CLOSED
10:01:01.055  Worker B writes: state=OPEN
```

Network timing determines whether your circuit recovers. The outcome is disconnected from any coherent view of service 
health.

You can't debug this from logs--each worker made a locally correct decision based on what it observed. The race is 
invisible at the application layer.

## Serialize the entire probe

You might think atomic state transitions would help--use CAS, only transition if state is what you expect. But the race 
happens before the transition and during the probe itself. Fifty workers checking "eligible to probe?" and all seeing 
"yes" is the problem. Atomic writes can't fix concurrent reads.

The fix: serialize at the probe level, not the transition level.

One worker acquires a lock before probing. It holds the lock through the entire sequence: probe, evaluate, transition 
if needed, release. Every other worker that attempts to probe while the lock is held gets an immediate answer: "someone 
else is probing, treat this as OPEN."

Here's the racing decisions scenario with serialized probes:

```
10:01:01.000  Worker A acquires lock, begins probe
10:01:01.000  Worker B sees lock held → immediate fallback
10:01:01.050  Worker A probe: SUCCESS
10:01:01.055  Worker A writes: state=CLOSED, releases lock
```

Both races vanish: 

- **Thundering herd**: One probe reaches the downstream. The other 49 requests get immediate fallback--no waiting on a struggling service.
- **Racing decisions**: One worker observes and decides. No conflicting writes.

The workers that don't get the lock get *better* behavior than before. Instead of slow probes to a struggling service, 
they get immediate fallback. The circuit breaker does its job: protecting your system from downstream failure.

### Same reasoning, different primitives

This isn't distributed-systems-specific. In a threaded environment, you'd use a Mutex. Lock before probe, unlock after 
transition. The primitive is simpler, but the reasoning is identical: serialize the entire recovery sequence, not 
just individual operations.

Distributed systems need distributed primitives--a Redis lock with TTL instead of a mutex. But the coordination strategy 
is the same. 

### The cost

There isn't much of one. The circuit is OPEN. Every request is already hitting fallback--that's what OPEN means. The 
lock just decides which single request gets to probe. The other 49 were going to get fallback anyway. Now they get it 
immediately instead of after waiting on a struggling service.

Probe throughput drops to one at a time. But you only need a few successful probes to know recovery is safe. The 
other concurrent requests aren't gathering information--they're adding risk. 

The lock is held for the duration of one probe and bounded by your probe timeout, which should be short anyway. 

## Conclusion

The state machine is the spec. The coordination strategy is the implementation.

If you're building or operating circuit breakers with any form of concurrent execution--threads or processes--audit your 
recovery path. Single-threaded tests won't find these problems. You need concurrent load during recovery, and even then 
the symptoms are "config doesn't quite work" rather than crashes.

This ships in Stoplight 5.7. [Link to PR] if you want the implementation details.

[Stoplight]: https://github.com/bolshakov/stoplight
[thundering herd]: https://en.wikipedia.org/wiki/Thundering_herd_problem
[Link to PR]: https://github.com/bolshakov/stoplight/pull/524

