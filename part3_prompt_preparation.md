# Part 3: Prompt Preparation

## Task 3.1: Comprehensive Prompt Documentation

**Selected PR:** PR #232 - Manual Partition Assignment for AIOKafkaConsumer  
**PR URL:** https://github.com/aio-libs/aiokafka/pull/232  
**Repository:** aio-libs/aiokafka

---

## 3.1.1 Repository Context

So aiokafka is basically the async version of the kafka-python library. If you've ever tried to use Kafka with asyncio, you'll know the pain - the standard library blocks, and that completely messes up your event loop. This library solves that problem.

It gives you two main classes to work with: AIOKafkaProducer for sending messages and AIOKafkaConsumer for reading them. Under the hood it deals with all the Kafka wire protocol stuff - connecting to brokers, serializing messages, handling responses, managing consumer group coordination, tracking offsets, all of that.

The people using this are mostly backend Python developers who are building things like event-driven microservices, data pipelines, log aggregation systems, or real-time analytics. Basically anyone who needs to move messages through Kafka but is also using asyncio for their application. These tend to be high-throughput systems where you can't afford to have your code sitting around waiting on network calls.

What makes it useful is that it fits naturally into async Python. Instead of blocking or messing around with thread pools, you just `await consumer.getmany()` and it works alongside your other async code - HTTP handlers, database queries, WebSocket connections, whatever.

The consumer has two ways to get partitions: automatic assignment through consumer groups (which is the normal way most people use Kafka) and manual assignment where you tell it exactly which partitions to read. PR #232 is about adding that second option.

---

## 3.1.2 Pull Request Description

This PR adds an `assign()` method to AIOKafkaConsumer. Basically it lets you manually pick which partitions you want to consume instead of letting Kafka's coordinator decide for you.

**How it worked before:**
You'd call `subscribe(['my-topic'])` and that puts you in a consumer group. The group coordinator figures out which partitions each consumer gets, and that can change whenever someone joins or leaves the group (rebalancing). You don't really have control over which specific partitions you end up with.

**How it works after:**
Now you can do `consumer.assign([TopicPartition('my-topic', 0), TopicPartition('my-topic', 1)])` and you'll get exactly those partitions. No coordinator involvement, no rebalancing, you know exactly what you're reading from.

**Why would you want this?**

Honestly there are a few scenarios I can think of. One is replay - say you need to go back and re-read messages from a specific partition starting at some offset. With subscribe you might not even get that partition. With assign you just grab it directly.

Another is static partition assignment. Some systems are set up where each instance always handles the same partitions - like instance 0 gets partitions 0-2, instance 1 gets 3-5, etc. This is pretty common in Kubernetes setups with stable pod names.

Also useful for debugging. If there's something weird in partition 7, you want to point a consumer at just that partition to check it out.

The implementation makes subscribe and assign mutually exclusive - calling one clears the other. Otherwise you'd have confusing situations where both are sort of active.

---

## 3.1.3 Acceptance Criteria

Here's what I think needs to work for this to be considered done:

✓ **AC1:** When you call `assign([TopicPartition('topic', 0)])`, the consumer should only fetch from that partition. It should NOT send JoinGroup or SyncGroup requests - basically skip the whole group coordination protocol.

✓ **AC2:** When you call `assign()` after previously calling `subscribe()`, the subscription should be cleared. The consumer should switch to manual assignment mode.

✓ **AC3:** When you call `subscribe()` after previously calling `assign()`, the assignment should be cleared. The consumer should switch to group mode and do the normal join process.

---

## 3.1.4 Edge Cases

### Edge Case 1: Assigning a partition that doesn't exist

Someone calls `assign([TopicPartition('my-topic', 99)])` but the topic only has 10 partitions.

The consumer should raise an error - something like `UnknownTopicOrPartitionError` - with a message that makes it obvious which partition was the problem. This matters because typos happen all the time, and partition counts can change. You don't want to sit there wondering why you're not getting any messages when actually you just typoed the partition number.

### Edge Case 2: Broker failover while consuming

You're consuming from partition 0, and suddenly the broker hosting it dies. Leadership moves to a replica on another broker.

The consumer should detect this (probably through a NOT_LEADER_FOR_PARTITION error or similar), refresh its metadata to find the new leader, reconnect to that broker, and continue consuming. No crash, no data loss, no manual intervention needed. This happens in production all the time - rolling restarts, hardware failures, whatever. Manual assignment shouldn't make the consumer less resilient than subscribe mode.

### Edge Case 3: Calling assign while a fetch is in progress

You call `getmany()` and it's sitting there waiting for messages. Then you call `assign()` with different partitions before the fetch completes.

The consumer should handle this cleanly. Either cancel the pending fetch or let it finish and return whatever it got. But the next fetch should use the new assignment. And crucially, you shouldn't get messages from the old partitions leaking into results after the assignment change. In real applications, assignment changes might come from external signals or admin APIs, not just sequential code - so the consumer might not be idle when you reassign.

---

## 3.1.5 Initial Prompt

I need to add manual partition assignment to AIOKafkaConsumer. The library is aiokafka - it's an asyncio client for Apache Kafka in Python.

**Context**

Right now the consumer only supports `subscribe()` where you join a consumer group and the coordinator decides which partitions you get. That works fine for most cases, but sometimes you need to specify exactly which partitions to consume. Think replay scenarios, static partition mapping, debugging specific partitions, or cases where partition assignment is managed externally.

The Java Kafka client has `assign()` for this. I need to implement the equivalent for aiokafka.

**What needs to be built**

Add two methods to AIOKafkaConsumer:

1. `assign(partitions)` - accepts a list of TopicPartition objects. The consumer should fetch only from those partitions. No group protocol at all - don't send JoinGroup or SyncGroup requests.

2. `assignment()` - returns the set of currently assigned partitions, whether they came from `assign()` or from the group coordinator.

There's also some internal work needed:
- Track whether we're in "subscribe mode" or "assign mode" - they're mutually exclusive
- Update the fetcher to build requests for explicitly assigned partitions
- Make sure offset operations (seek, position, commit) all work properly with manual assignments

**Behavior requirements**

- `assign()` takes effect immediately, start fetching from those partitions
- Calling `subscribe()` after `assign()` should clear the assignment and switch to group mode
- Calling `assign()` after `subscribe()` should clear the subscription and switch to manual mode
- Handle broker leadership changes gracefully - detect stale leader, refresh metadata, reconnect
- Offset commits should work when `group_id` is set, even in assign mode
- `assign([])` with empty list should stop all fetching

**Edge cases to handle**

- Non-existent partition: raise a clear exception, don't silently fail
- Reassignment during active fetch: complete or cancel cleanly, don't leak messages from old assignment
- Multiple topics in one assignment: should work fine
- Restart after offset commit: should resume from committed position

**Where to make changes**

Main files that need modification:
- `aiokafka/consumer/consumer.py` - the new public methods go here
- `aiokafka/consumer/subscription_state.py` - tracking which mode we're in
- `aiokafka/consumer/fetcher.py` - building fetch requests for assigned partitions
- `tests/test_consumer.py` - need test coverage for all this

Follow whatever patterns already exist in the codebase. Use asyncio properly. Add type hints. Write docstrings for the public methods. And don't break existing subscribe behavior - that needs to keep working exactly as before.

---

## Integrity Declaration

"I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words."
