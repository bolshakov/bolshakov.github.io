---
layout: post
title: "A Sliding Window on Redis Without Storing the Events"
date: 2026-10-01 00:00:00 +0000
---

Stoplight is a circuit breaker for Ruby. Stoplight 6.0 changes how its Redis data store keeps the sliding-window
metrics behind the `error_rate` strategy, so Redis memory per circuit breaker no longer grows with request rate. It
depends only on the window size.

A "light" is one breaker, wrapping one dependency, and `error_rate` opens it when the failure ratio inside the window
crosses a threshold. Here is the Redis memory used per light on Stoplight 5.x and on 6.0:

| window | req/s  | events in window | Stoplight 5.x | Stoplight 6.0 |
|--------|--------|------------------|---------------|---------------|
| 1m     | 10     | 600              | 74.9 KB       | **11.6 KB**   |
| 1m     | 100    | 6,000            | 725.3 KB      | **11.7 KB**   |
| 3m     | 10     | 1,800            | 204.3 KB      | **53.3 KB**   |
| 3m     | 100    | 18,000           | 2,092.6 KB    | **51.7 KB**   |
| 5m     | 10     | 3,000            | 371.3 KB      | **88.0 KB**   |
| 5m     | 100    | 30,000           | 3,356.0 KB    | **88.0 KB**   |
| 10m    | 10     | 6,000            | 753.5 KB      | **172.6 KB**  |
| 10m    | 100    | 60,000           | 6,672.1 KB    | **174.6 KB**  |

The 5.x column grows with request rate. The 6.0 column does not: a one-minute window costs 11.6 KB at ten requests 
a second and 11.7 KB at a hundred. Memory is now a function of `window_size`, roughly 200 to 300 bytes per second of 
window that saw traffic. Throughput is unchanged.

This post describes how the old design ended up proportional to traffic, and what the new one does instead. The 
algorithm itself is quite standard, but Redis nudges you away from it.

## The report

[Issue #872] was reported against Stoplight 5.8 with this configuration:

```ruby
Stoplight.configure do |config|
  config.traffic_control = :error_rate
  config.window_size = 300
  config.threshold = 0.5
end
```

A five-minute window, and Redis memory that grew with traffic rather than with the window. Every request left a 
member in a sorted set, so with thousands of active circuit breakers Redis' memory requirements scaled with the 
request rate.

## Sorted sets are the obvious first design

If you need a sliding window on Redis, the sorted set is probably the first structure you look at. Score is 
the timestamp, each member is one event, and `ZRANGEBYSCORE` or `ZCOUNT` answers "how many events in the last N 
seconds" directly. The data structure was built for range-by-score queries - perfect fit for a time window.

Our initial design kept one sorted set of failures per light. `record_failure` did `ZADD`, trimmed by score, and capped 
by rank at the threshold. `record_success` deleted the set. This was correct for the only strategy that existed 
back then, consecutive errors, where a single success resets the error history. Memory was bounded by the threshold, 
a failure count for that strategy.

The `error_rate` strategy broke that. A ratio needs both successes and failures retained, so deleting the set on 
success was no longer an option. The store was redesigned around sorted sets of events: one member per event, 
a random ID with the timestamp as its score, around 120 bytes each, and a `ZCOUNT` over the window on read.

One set member per event means memory is proportional to request rate. The 5.x column at the top shows it: ten times 
the traffic, ten times the memory, at every window size. At 50 requests a second a five-minute window is 15,000 
events and around 1.8 MB per light. Across a few thousand lights that is gigabytes, for a quantity that fits in two 
integers.

## The textbook window, which the memory store already uses

The usual way to implement a sliding window without storing events is to bucket time, keep a counter per bucket, 
and keep a running total. A write increments the current bucket and the total. Eviction walks buckets that have 
aged out of the window, subtracts each one's count from the total, and drops it. A read returns the total.

Stoplight's in-memory store does exactly this, in about 40 lines:

```ruby
def increment
  timestamp = @clock.monotonic_seconds
  slide_window!(timestamp - @window_size)
  @buckets[timestamp.to_i] += 1
  @running_sum += 1
end

def sum_in_window
  slide_window!(@clock.monotonic_seconds - @window_size)
  @running_sum
end
```

Ruby's Hash preserves insertion order, so `slide_window!` peeks at `@buckets.first`, and while that bucket is 
older than the boundary, subtracts its count and `shift`s it off. Reads and writes are O(1) amortized, and memory 
is bounded by the number of seconds in the window, regardless of how many events were recorded in each.

The running total is the part that is easy to miss. Without it the counters alone still bound memory, but a 
read has to sum every bucket inside the window, and that cost grows with `window_size`. With it, a read returns 
one number, and the per-bucket counters exist only so that eviction knows how much to subtract.

Nothing about this is specific to the in-memory data store. The question was only how to express it in Redis.

## Translating it to Redis

The idea of per-second counters in a Redis hash, with a sorted set alongside as an index, was proposed by 
[@sirwolfgang] in [#283] in the spring of 2025, while the `error_rate` store was being designed. It was 
benchmarked against the sorted sets of events at the time and set aside: that version summed the buckets on read, which 
was slower than a `ZCOUNT` over the events. With the running total, a read is an `HMGET` of two fields after eviction, 
and throughput is on par with the event sets.

The 6.0 layout, per light, is one hash and one sorted set:

- the hash holds `bucket:<ts>:s` and `bucket:<ts>:f` counters, one pair per second that saw traffic, plus 
  `total_successes` and `total_failures` and the metadata fields (`last_error_at`, `consecutive_errors`, and so on)
- the sorted set holds the bucket names, scored by their timestamp, and nothing else

The sorted set is still there, but it no longer contains events. It's used for the insertion order the Ruby Hash 
gave the memory store for free. Redis hashes are unordered, so without an index there is no way to ask "which fields 
are older than this boundary" without scanning every field. `ZRANGE ... BYSCORE` answers that in one call.

Bucket names are timestamps, so a script could walk the clock instead, from the oldest bucket it knows about up to 
the boundary. But that visits every empty second too, and after an idle gap those are most of them. The index lists 
only the seconds that saw traffic.

Here is `record_success.lua`, slightly simplified. Every write runs eviction first, then increments:

```lua
local window_size  = tonumber(ARGV[1])

local metrics_key  = KEYS[1]
local ts_index_key = KEYS[2]
local time         = redis.call('TIME')
local request_ts   = tonumber(time[1]) + tonumber(time[2]) / 1000000.0
local bucket_ts    = math.floor(request_ts)

evict_buckets(metrics_key, ts_index_key, bucket_ts - window_size)

local bucket = 'bucket:' .. bucket_ts
redis.call('HINCRBY', metrics_key, bucket .. ':s', 1)
redis.call('HINCRBY', metrics_key, 'total_successes', 1)
redis.call('ZADD', ts_index_key, bucket_ts, bucket)
```

`evict_buckets` is the helper shared by the read and write scripts. The next section shows it.

## Eviction

`evict_buckets` runs at the top of every script that touches the window. The read script, `metrics_snapshot`, 
evicts before it returns the totals, and the write scripts evict before they increment. That is the same shape as 
the memory store, where `slide_window!` is the first line of both `increment` and `sum_in_window`.

```lua
local EVICT_BATCH_SIZE = 1000

local function evict_buckets(metrics_key, ts_index_key, window_start)
  local buckets = redis.call('ZRANGE', ts_index_key, '-inf', window_start, 'BYSCORE')
  if #buckets == 0 then
    return
  end

  local failures, successes = 0, 0

  for from = 1, #buckets, EVICT_BATCH_SIZE do
    local to = math.min(from + EVICT_BATCH_SIZE - 1, #buckets)

    local fields = {}
    for i = from, to do
      fields[#fields + 1] = buckets[i] .. ':f'
      fields[#fields + 1] = buckets[i] .. ':s'
    end

    local values = redis.call('HMGET', metrics_key, unpack(fields))

    for i = 1, #values, 2 do
      failures = failures + (tonumber(values[i]) or 0)
      successes = successes + (tonumber(values[i + 1]) or 0)
    end

    redis.call('HDEL', metrics_key, unpack(fields))
  end

  redis.call('HINCRBY', metrics_key, 'total_failures', -failures)
  redis.call('HINCRBY', metrics_key, 'total_successes', -successes)
  redis.call('ZREMRANGEBYSCORE', ts_index_key, '-inf', window_start)
end
```

The cost of one call is `O(K log W)`, where `K` is the number of stale buckets that had writes and `W` is `window_size`. In 
steady traffic `K` is small, usually one or zero, because every call evicts whatever the previous one left behind. After 
a long idle gap, the next call pays for the whole backlog at once, and that backlog is bounded by `W`. Amortized over a 
long sequence of calls the cost is `O(log W)`: each bucket is evicted once, and it was created once. The memory store's 
`slide_window!` does the same eviction work, minus the log factor from the sorted-set index. None of it depends on 
request rate.

The batching was not in the first version. `HMGET` and `HDEL` take their fields through Lua's `unpack`, which is 
capped at ~8,000 arguments. Two fields per bucket meant any window over 3,999 seconds could hit the cap during a 
backlog eviction. Batches of 1,000 buckets keep each call at 2,000 arguments.

## Summary

Nothing in the final design is new. Per-second counters, a running total and eviction from the front are what the 
textbook sliding window algorithm does. What Redis changed was the route there. Sorted sets answer range-by-score 
queries so well that storing the events looks like the natural design, and it took a memory report to notice that 
nobody ever needed the events, only their count. Hashes are unordered, so the sorted set came back, holding 
a few hundred bucket names instead of the events themselves.

[Issue #872]: https://github.com/bolshakov/stoplight/issues/872
[#283]: https://github.com/bolshakov/stoplight/issues/283#issuecomment-2849373776
[@sirwolfgang]: https://github.com/sirwolfgang
