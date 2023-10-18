---
title: "Exploring how Flink handles late events"
date: 2023-10-17
layout: post
description: Flink has a concept of time that is not always the same as real world time. This article is exploring what causes an event to be considered late and how Flink jobs take account of lateness.
---
## Watermarks

https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/concepts/time/#event-time-and-watermarks

> A Watermark(t) declares that event time has reached time t in that stream, meaning that there should be no more elements from the stream with a timestamp `t’ <= t`.

As events are consumed from a stream, Flink is tracking the _latest_ event time seen. Or in epoch terms, Flink is tracking the _largest_ epoch number seen. This does not mean that a watermark progresses in system clock time - if a Flink job does not receive an event for 5 "real" minutes, the watermark remains unchanged. Only a new event can progress the watermark. In addition - event time does not have to map directly to the system clock.

_Side note - if you placed a stick in the sea at low tide and then returned the next day - during high tide the water would have risen and marked a higher level - literally a water mark._

Additionally, events do not have to be produced in real time either. This can be clearly demonstrated by testing using data with timestamps from the year 3000. If you produced an event with timestamp `3000-01-01 00:00:55`, waited 5 real seconds, then produced an event with timestamp `3000-01-31 23:59:59` Flink will not bat an eyelid.

By itself, the watermark is simply additional metadata for the stream. Flink operators have their own opinions over how/if they respond when a watermark increases.

## Late elements

https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/concepts/time/#lateness

> It is possible that certain elements will violate the watermark condition, meaning that even after the Watermark(t) has occurred, more elements with timestamp t’ <= t will occur.

Flink acknowledges that conditions can arise that cause an event to reach the stream after a more recent event. This is much more likely when using `event time` as the timestamp is in the payload and is not necessarily related to the time it was produced or received. With `processing time` Flink assigns a timestamps from its local clock when the event is consumed (and it's unlikely that clock will go backwards).

> Late elements are elements that arrive after the system’s event time clock (as signaled by the watermarks) has already passed the time of the late element’s timestamp.

For example - you have three trucks - `red, green, blue` - reporting their location every 30 seconds onto the same event stream. If there is a network delay - an event from one truck may arrive on the stream after the others have produced more events.

This could result in the following stream:

`[{red: 10:45:00}, {green: 10:45:00}, {blue: 10:45:00}, {red: 10:45:30}, {green: 10:45:30}, {red: 10:46:00}, {green: 10:46:00}, {blue: 10:45:30}, {blue: 10:46:00}]`

Some of the `blue` events are late as they have `t' <= watermark(t)` (technically the `green` events are late too as they also have `t' <= watermark(t)` but as you'll see later it's only when `t' < watermark(t)` that they are treated differently to "on time" events).

Also note - all of these events could have been produced to the event stream within seconds of one another - late is in regards to the watermark. In the same regard - if all three trucks queued up a day of readings and then sent them to the stream - there could be many sequences of late events. Worst case - the stream could contain a sequence of ALL red, then ALL green, then ALL blue - thus making every green and blue event late.

Late events however are not necessarily a problem. For example, a Flink job that only uses stateless operators (filters, splits, transforms) has no reason to notice or care about late events. The following SQL query will output ALL the events in the order they arrived on the red/green/blue stream above.

```sql
SELECT * from stream_with_late_blue_events
```

_Side note - https://github.com/ververica/flink-sql-cookbook/blob/aa0bd978b1116a073e094ceea44ecd659ae3cefd/other-builtin-functions/03_current_watermark/03_current_watermark.md - shows `WHERE log_time <= CURRENT_WATERMARK(log_time)` as the way to remove late events (if you cared)._

However, when dealing with windowed processing - lateness becomes an important consideration.

## Windows, watermarks and lateness

### Window basics

https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/concepts/time/#windowing

> it is impossible to count all elements in a stream, because streams are in general infinite (unbounded). Instead, aggregates on streams (counts, sums, etc), are scoped by windows, such as “count over the last 5 minutes”, or “sum of the last 100 elements”.

I like to think of windows as "buckets" - as Flink consumes an event it is placed into the appropriate bucket (or buckets if using sliding windows). So when using time based windows - the event is placed into the bucket labelled `contains events with time t where x <= t < y`.

_Side note - you can control the window size - but Flink determines the start/end times of the window using the stream contents. For example, a `30 minute window` could have a start/end time of `10:21:00 - 10:51:00` - which becomes more obvious when you use atypical window sizes like `47 seconds`._

https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/dev/datastream/operators/windows/#window-lifecycle

> a window is created as soon as the first element that should belong to this window arrives, and the window is completely removed when the time (event or processing time) passes its end timestamp

When an event is read that progresses the watermark past the end timestamp of the window - Flink puts a lid on the bucket and can now process the contents.

Note - many scenarios involve processing streams of real time data - such that the event time is progressing in line with clock time, and so windows appear to close in real time (e.g with `5 minute windows` the latest window to close will be the `last 5 minutes` of real time). But historical streams of data will open and close windows instantly as the stream is consumed and sporadic streams of data will pause and resume when a new batch of events is received.

### Late events

https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/dev/datastream/operators/windows/#window-assigners

> All built-in window assigners (except the global windows) assign elements to windows based on time, which can either be processing time or event time.

https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/dev/datastream/operators/windows/#allowed-lateness

> By default, late elements are dropped when the watermark is past the end of the window.

As such, the inverse also applies - the window assigners will assign late events to a window as long as the watermark has not passed the end of the window.

With a 24 hour window that opened at `00:00:00`, and a current watermark of `23:59:59` - an event could be nearly 24 hours _late_ and still be assigned to the open window. This does make sense - if you're aggregating over a 24 hour window, it doesn't matter if the event is late by 10 seconds, 30 minutes, or 15 hours - it still should affect the window result.

In a flow with multiple windowed operators it also makes sense that "lateness" is not a quality of the event; each operator can decide whether an event is too late for its function.

### Allowed lateness

A late event is one that has an event time less than or equal to the current watermark. So an event's "lateness" could be considered a measure of _how late_ the event is compared to other events on the stream - the difference between the current watermark and the event time.

However, we've seen that Flink will happilly process late events regardless of the event's "lateness". Events are only dropped when they cannot be assigned to an open window. So "lateness" in the context of windows is actually about whether the event arrived after the window was closed.

Flink's documentation does appear to support this, even if the wording is a bit obtuse.

https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/dev/datastream/operators/windows/#allowed-lateness

> Allowed lateness specifies by how much time elements can be late before they are dropped, and its default value is 0. Elements that arrive after the watermark has passed the end of the window but before it passes the end of the window plus the allowed lateness, are still added to the window.

We know that a late element is not dropped if it can be assigned an open window. So when Flink says allowed lateness refers to `how much time elements can be late` - _late_ must be referring to elements that cannot be assigned an open window. In the case of a 12 hour window - both `00:00:01` and `11:59:59` are _late_ if the watermark is at `12:00:00` - the `00:00:00-11:59:59` window has closed.

When you configure lateness - you are asking Flink to delay _closing_ any window until the watermark progresses past the window end PLUS the "allowed lateness".

So in the above example - if the allowed lateness was 5 minutes, as long as the watermark has not reached `12:05:00` - both `00:00:01` and `11:59:59` can be assigned into the open `00:00:00-11:59:59` window.

Note again - this does not mean Flink delays closing the window by 5 real world minutes. It means 5 minutes of event time.

### What is the value?

"Allowed lateness" feels like it was designed for real time streams - and where a delay in sending an event caused it to narrowly miss a window. Maybe there was a blip such that `blue` truck's `00:59:59` event arrived on the stream after `red` truck sent an event that closed the `00:00:00 - 00:59:59` window.

It has the same effect regardless of whether the window was 24 hours or 1 minute - it's letting Flink delay closing the window to catch these boundary events.

If you assume a real time stream, then it makes sense that you would set the value to allow for a known/guestimated possible network delay (e.g `30 seconds`). What's still unclear is how to best decide on the value in other use cases.
