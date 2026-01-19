# Part 4: Technical Communication

## Task 4.1: Scenario Response

**Scenario:** The reviewer asks:
> "Why did you choose this specific PR over the others? What made it comprehensible to you, and what challenges do you anticipate in implementing it?"

---

## Response

I went with PR #232 because I could actually follow what it was doing. Some of the other PRs were about protocol versions or wire format stuff, and I'd have spent ages just understanding the existing code before even looking at the changes.

This one's goal is clear - add `assign()` so users can pick specific partitions to consume. I've used the Java Kafka client before which has the same thing, so I already knew how it should behave. That helped a lot.

The scope's reasonable too. Changes are mostly in the consumer module, not spread across the whole codebase.

What worries me though - the state management between subscribe and assign modes. When you switch modes, you gotta clean up properly. Pending fetches, buffers, coordinator state. Miss something and you get weird bugs that only show up in specific call sequences.

Offset handling's tricky too. If someone uses `assign()` with a `group_id`, commits should still work. But the commit logic might assume you did the normal group join. Need to make sure that doesn't break.

My approach: read through how `subscribe()` manages state first, then write tests for edge cases before coding - switching modes mid-fetch, resuming from committed offsets, assigning bad partitions. Tests catch issues early.

---

## Integrity Declaration

"I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words."
