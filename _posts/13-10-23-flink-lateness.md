---
title: "How do out-of-order events affect Flink timely processing?"
date: 2023-10-13
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
3. When using a timestamp as event time, events may arrive onto the source stream out of order - i.e a "newer" event has a timestamp that is earlier than one in a previous message
4. Flink supports lateness by configuring a delay on [watermarking](https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/dev/table/sql/create/#watermark)
    - A delay of `5 minutes` means an event can arrive up to 5 minutes late.
    - If an event moves the watermark to `11.05`, and the next event has an event time of `11.01`, it will be accepted as it's within the "lateness" bound.
    - However, a message of `10.59` is "too late" and is discarded
  
So, when I started to use windowing - I explored scenarios such as:

1. Count how many events arrived in an hour window
    - Given an event stream with times - `[11:00, 11:11, 11:32, 11:59, 12:01]` - the `11:00 - 11:59` window closes when the `12:01` event is read, with a count of `4`
    - Given lateness of `5 minutes` and an event stream with times - `[11:00, 11:11, 11:32, 11:59, 12:01, 12:02, 11:59, 12:05, 11:59]` - the `11:00 - 11:59` window closes when the `12:05` event is read, with a count of `5`. The second `11:59` is discarded as "too late" - the watermark was progressed to `12:05`.
  
    
