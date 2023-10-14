---
title: "How do out-of-order events affect Flink timely processing?"
date: 2023-10-13
layout: post
---

When I first started to explore Flink, I began with the SQL interfaces. I'm lazy; the SQL client makes it incredibly easy to get started without needing to setup an IDE and work with the Flink Java libraries.

After convincing myself of the basics of stream processing - sources, sinks, basic stateless operators like filtering - I moved onto the more advanced topic of temporal processing; operators that specifically use time in their execution.

Flink has a good introduction to [time](https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/concepts/time/) in the context of stream processing jobs. My takeaway from the first reading was relatively simple:

1. A stream of events moves forwards in time. Depending on the source connector, you can configure the "event time" to be one of:
    1. The time the event arrived at the source (connector time)
    2. The time the event was consumed by Flink (processing time)
    3. A timestamp inside the event payload/metadata
2. Temporal operators need to know what the current time is for the stream - the "latest/most recent" event time seen. Flink calls this a `watermark`. Notably - this is not _clock time_ - it is progressed only by a new event arriving at the source stream.
    - In my explorations, it was easiest to demonstrate this by using "impossible" timestamps in the events - e.g `01-01-3000` (not much has changed, but they lived underwater)
    - Even with a data stream with these dates, Flink will happily output results - it's processing the stream and doesn't inherently care about system/clock time
3. When using a timestamp as event time, events may arrive onto the source stream out of order - i.e a "newer" event has a timestamp that is earlier than one in a previous message. By default, out of order messages would be discarded.
4. Flink supports out of order via `lateness` - a delay on [watermarking](https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/dev/table/sql/create/#watermark)
    - A delay of `5 minutes` means an event can arrive up to 5 minutes late compared to the current watermark
    - If an event moves the watermark to `11.05`, and the next event has an event time of `11.01`, it will be accepted as it's within the "lateness" bounds
    - However, a message of `10.59` would be "too late" and is discarded
  
So, I started to explore scenarios to validate my understanding. For example - `partition the stream into hourly windows and count the number of events in each hour`.
- Given an event stream with times - `[11:00, 11:11, 11:32, 11:59, 12:01, 11:59]` - the `11:00 - 11:59` window closed (with a count of `4`) when the `12:01` event was processed. The second `11:59` was discarded.
- Given lateness of `5 minutes` and an event stream with times - `[11:00, 11:32, 11:59, 12:01, 12:02, 11:59, 12:05, 11:59]` - the `11:00 - 11:59` window closed (with a count of `4`) when the `12:05` event was processed.
    - The `12:02` event didn't close the window, as `5 minutes` lateness meant Flink needed to wait until the watermark reached `12:05` before it could start to discard events
    - The first `11:59` was included in the window because the watermark was only at `12:02`
    - The second `11:59` was discarded as "too late" because the watermark had progressed to `12:05`

I had convinced myself that Flink discarded messages at ingestion time based off the current watermark. However, while discussing lateness with a colleague - his explorations had shown that until a window closes, the event stream can be entirely out of order and still be processed - i.e `[11:01, 11:59, 11:23, 11:01, 11:15]` is perfectly acceptable.

It turns out I was a victim of confirmation bias. Because I had understood lateness to refer to the difference between the event time and the current watermark, I had only sent events that were within the lateness bounds - i.e `11:59` appeared to be discarded after `12:05` with a lateness of `5 minutes`. Naively, I had not tried sending earlier events (e.g `11:45` once the watermark was at `12:02`) to see what happened.

I had overlooked a key phrase in the documentation - `once a watermark reaches an operator, the operator can advance its internal event time clock to the value of the watermark`. 

This nuance has a startling effect on my previous explorations. It is the choice of an operator as to whether its processing is affected by the current watermark. Flink is not discardng messages at ingestion. As such:
1. A simple SQL expression of `SELECT * FROM event_stream` will happily process (and output) out of order messages - both with and without lateness bounds.
2. Given lateness of `5 minutes` and an event stream with times - `[11:00, 11:59, 12:02, 11:01, 11:25, 11:01, 12:05]` - ALL events that fit into the `11:00-11:59` window are counted. Lateness simply controls the delay until a window closes.
3. If the lateness delay is larger than the window - e.g 1 hour lateness compared to 5 minute window size - the watermark has to progress 1 hour before that 5 minute window closes. So if the event stream was populated in real time, lateness introduces a 1 hour lag on the results.
