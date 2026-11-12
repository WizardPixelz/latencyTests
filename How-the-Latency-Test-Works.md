# How the Latency Test Works

**In one sentence:** the producer stamps each event's send time in a message header, the consumer notes when each event arrives and logs the difference, and after the run a report script lines up every event's time from fastest to slowest and reads off the percentiles.

## The pieces

| Piece | What it does |
| --- | --- |
| Producer | Sends the event files. Stamps each event with an ID and its send time, in a message header. |
| Broker (Kafka) | Checks each event against its schema, stores it, and confirms it to the producer. |
| Consumer | Receives each event, notes the receive time, and logs the latency against the event's ID. |
| Report script | After the run, matches the producer's and consumer's logs and works out the percentiles. |

The timing goes in a **header**, not in the event's content, so the payload still matches its schema and nothing about the event changes.

## Step by step

1. **The producer stamps and sends.** Just before each event goes out, the producer adds a header with a run ID, an event ID and the send time. It also logs when the broker confirms the event (the broker's share of the time).
2. **You send the events.** For example, 100 events at a steady pace.
3. **The consumer receives and logs.** For each event, it notes the receive time, reads the send time from the header, and logs the difference (receive time − send time) against the event ID. That difference is the event's latency.
4. **The report matches the logs.** After the run, `latency_report.py` matches the producer's log to the consumer's log by event ID. This shows anything lost (sent but never received) or duplicated (received twice).
5. **The report sorts and reads off the percentiles.** It lines up every event's latency from fastest to slowest and reads the time at set spots in that line.

## What gets recorded

| What | Recorded by | When |
| --- | --- | --- |
| Event ID and send time | Producer, in the header and its log | Just before sending |
| Broker confirmation time (send-to-ack) | Producer, in its log | When the broker confirms |
| Receive time | Consumer | The moment the event arrives |
| Latency (receive − send) | Consumer, in its log | The moment the event arrives |
| Lost, duplicated, rejected counts | Report | After the run |
| p50, p95, p99, max | Report | After the run |

## From latencies to percentiles

Each event isn't given a percentile. The percentiles are **spots in the sorted line**, and you read the time at each spot. With 100 events:

```
fastest                                              slowest
  #1 ......... #50 ......... #95 ...... #99   #100
               p50           p95        p99   max
```

| Percentile | Spot (of 100) | What it means |
| --- | --- | --- |
| p50 (median) | 50th | The typical event: half were faster, half slower |
| p95 | 95th | The upper end of normal: only 5 in 100 were slower |
| p99 | 99th | The rare slow ones: only 1 in 100 was slower |
| max | 100th | The single slowest event, often a one-off |

- The percentage is measured against **all the events in that run**, not against a baseline or a target.
- The resulting time is then **compared with the target** (the SLA), for example "p95 under the target".
- More events make the tail trustworthy: a few hundred for p95, about a thousand for p99.
- p50 is used instead of the average, because a few slow events drag an average up.

## How the runs use it

| Run | Pace | Purpose |
| --- | --- | --- |
| Baseline | Slow | The system's normal p50 and p95 |
| Delay check | Slow, plus a known added delay | Every percentile should rise by exactly the delay: proves the measurement is accurate |
| A1 | Light load | Latency on a normal day |
| A2 | Near the 5.3 events-per-second limit | Whether latency climbs as traffic grows |

## What counts as a pass

- Every event sent arrives exactly once: nothing lost, duplicated or rejected.
- p95 (and p99, if the team wants it) is under the agreed target.

## Still to confirm with the team

- The latency target (SLA) and which percentile it applies to.
- That extra message headers don't affect schema validation.
- Whether "done" means the consumer received the event, or finished processing it.
