# Proposal: Agent StatsD Listener

Author(s): Sean Porter

Last updated: 2017-12-08

Discussion at https://github.com/sensu/sensu-go/issues/85.

## Abstract

The Sensu Agent provides a customized StatsD implementation, SensuStatsD. SensuStatsD listens for statistics, like counters and timers, sent to the Sensu Agent by tooling and application instrumentation over UDP or TCP. It periodically creates Sensu Events containing metric aggregates for the received statistics and transports the Events to a Sensu Backend. SensuStatsD provides a number of StatsD protocol enhancements, such as support for tags.

## Background

Sensu 1.x does not provide a built-in input specifically for metric data. The Sensu 1.x client does provide a check result input socket, which users have historically used for metric data (via check output).

StatsD is a popular and easy means of instrumenting application code and extracting metrics. In order to support the StatsD protocol as a means of metric data input in Sensu 1.x, a [Sensu StatsD extension was created](https://github.com/sensu-extensions/sensu-extensions-statsd). The extension runs within the Sensu 1.x client process, creating a StatsD TCP & UDP listener, and attempts to adhere to [Etsy's metric type specifications](https://github.com/etsy/statsd/blob/master/docs/metric_types.md). The extension receives StatsD statistics, parses them, and periodically sends Sensu check results containing metric data in check output (Graphite plaintext format).

## Proposal

Provide a StatsD implementation in the Sensu 2.0 Agent. Supporting the StatsD protocol for metric data input allows users to leverage a plethora of application instrumentation libraries and tooling. The Sensu Agent StatsD implementation can extend the protocol to provide additional functionality. DataDog, a SaaS monitoring company, already provides [a StatsD implementation with protocol extensions](https://github.com/DataDog/dd-agent/blob/master/dogstatsd.py) in their monitoring agent. We can reimplement nearly all of their extensions, such as tag support.

## Rationale

The Sensu 2.0 Agent provides an Events HTTP API. Users can use this API to create Sensu Events containing metrics. User applications would need Sensu Event specific instrumentation in order to accomplish this. StatsD provides several metric types and aggregation capabilities, the instrumentation would need to reimplement this functionality in order to achieve feature parity. Some things would still not be possible, such as a centralized metric counter or aggregate for more than one application instance.

## Compatibility

A built-in StatsD implementation with protocol extensions would allow users to leverage existing StatsD instrumentation libraries and tooling. The implementation can adhere to [Etsy's metric type specifications](https://github.com/etsy/statsd/blob/master/docs/metric_types.md) in order to maintain absolute compatibility.

## Implementation

Add a StatsD implementation to the Sensu 2.0 Agent, a new and separate TCP/UDP listener. Extend to the protocol to support tags. Create Sensu Events containing the produced metrics, using the [Sensu Metric Format](7-sensu-metric-format.md) to define them (e.g. `Event.Metrics.Data=[]`). Provide a means of configuring which Event Handlers the Events are sent to, for example, a Sensu Agent CLI argument to specify one or more Handlers `-statsd-event-handler=influxdb -statsd-event-handler=graphite` which sets the value of `Event.Metrics.Handlers=[]`. There may be some implementation overlap with [Check output metric extraction](https://github.com/sensu/sensu-go/issues/745).

The following StatsD implementations may be of some help.

- https://github.com/sensu-extensions/sensu-extensions-statsd
- https://github.com/DataDog/dd-agent/blob/master/dogstatsd.py

## Open issues (if applicable)

https://github.com/sensu/sensu-go/issues/85