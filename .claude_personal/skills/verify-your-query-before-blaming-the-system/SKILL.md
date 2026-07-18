---
name: verify-your-query-before-blaming-the-system
description: Use when verifying a code change end-to-end against an external system (Langfuse traces, DB rows, log aggregator, message queue) and your query for the expected data comes back empty. Before assuming the system-under-test failed to write the data, prove the query itself is right. This forbids the "restart the app, add debug logging, try one more flag" loop that burns time chasing a bug that only exists in the query filter.
---

# Verify your query before blaming the system

The trap: you make a change, run it against a real system, query for the resulting data, get an empty result, and start "fixing" the system-under-test. Nine times in ten the system worked and the query was wrong (wrong filter, wrong time window, wrong type, wrong ID). Every "let me try one more thing" on the SUT side is wasted.

## The rule

**When your verification query returns empty, do NOT touch the system-under-test again until you have proven the query is correct.**

## How to prove the query

Before you restart the app, add a flag, tweak env vars, or grep the logs harder, do all of these:

1. **Drop every filter.** Re-run the query with no `--type`, no `--from-date`, no `--limit 1`, no `WHERE`. Just the broadest possible read of the target table/endpoint. If the data appears here, your original filter was wrong. Stop; look at what fields the row actually has vs. what you filtered on.

2. **Check clock/timezone.** If the query has any time bound, compute the same bound both ways (local time, UTC) and confirm they agree with what the SUT would have written. A UTC/local mismatch alone can hide today's data behind a "yesterday-only" window.

3. **Confirm the write path succeeded end-to-end at the transport layer.** For HTTP: was there a 2xx? For a queue: did the ack land? A single well-placed `curl -sv` or one line of transport-level log is worth more than another restart of the app.

4. **Sanity-check the target ID/scope.** Are you querying the same tenant/project/database/topic the SUT actually wrote to? Two accounts, two regions, two schemas — this is where the loop most often lives.

Only after all four come back consistent are you allowed to conclude the SUT didn't write the data.

## Anti-pattern this kills

> query empty → restart the app → still empty → add `DEBUG` env var → still empty → suspect batching → wait longer → still empty → suspect region → check region → still empty → …

Every step assumed the query was correct. None of them were the bug. The bug was `--type GENERATION` (or its equivalent) filtering out the row that was actually there.

## When to skip this skill

If the query returns partial data — some rows show, some don't — the filter is probably fine and the SUT probably is misbehaving on a subset. Investigate the SUT. This skill is specifically for "query returned zero" panics.
