# Proposal: Sensu Metric Format

Author(s): Sean Porter

Last updated: 2017-12-06

Discussion at https://github.com/sensu/sensu-go/issues/7.

## Abstract

The Sensu Metric Format is a data structure for defining one or more metric points. Each metric point represents a measurement, with a name, tag set, field set, and timestamp. This format is used when transporting metrics within Sensu Events ("Metrics": []). Interoperability with metric tools is achieved through format translation, on ingress at the Sensu Agent, and on egress at the Sensu Backend. This format was inspired by the InfluxDB line protocol.

## Background

Sensu 1.x was designed and optimized for service checks. Some metric data was expected within service check execution output (e.g. Nagios PerfData), Sensu event mutators and handlers were expected to transform and send the data to a TSDB (e.g. Graphite). As Sensu adoption grew, it quickly became apparent that users wanted to use Sensu for metric collection. The community created tooling with a focus on metric collection and delivery with Sensu, such as WizardVan (authored by Greg Poirier), which improved Sensu's capabilities. While many users are very successful in using Sensu 1.x for metric collection, there remains room for improvement in regards to the user experience and performance.

## Proposal

Create a Sensu Metric Format for defining one or more metric points. Use the format when transporting metrics within Sensu Events. Using a single metric format within Sensu enables data validation and performance optimization. Sensu Event Mutator and Handler authors only need to concern themselves with a single metric format (consistency!). Having a defined metric format enables instrumentation libaries to create and send Sensu Events containing one or more metric points to a Sensu Events API. This makes metrics first-class.

## Rationale

### Use Metrics 2.0

Instead of defining an new metric format, Sensu 2.0 could use [Metrics 2.0](http://metrics20.org/), "an emerging set of conventions, standards and concepts around timeseries metrics metadata". While the idea of a common modern format is great, the collaboration on and adoption of the [spec](http://metrics20.org/spec/) does not seem to be there.

### Do Nothing

Not implementing a metric format would force users to push metrics through Sensu using Event Check output.

## Compatibility

Given the Sensu 2.0 Event still contains Check output, users can still push metrics through Sensu using the same method used in 1.x. However, there is an issue with this is, as the Sensu 2.0 Event does not provide an attribute to indicate that the Check output contains metric data (1.x used check `"type": "metric"`). Currently, the Backend inspects Event "Metrics" to determine if there are metric points to go through the Event pipeline. This needs a proposal.

## Implementation

A metric format capable of representing one or more metric points. The metric data can originate from a variety of metric tools and formats.

Each metric point has a name, tag set, field set, and timestamp.

### Name

The metric name provides a sense of metric identify and context (e.g. name=cpu_usage). Because this proposed spec has a tag set, the complete metric identity does not need to be encoded in the metric name.

### Tag Set

Tags are key/value pairs (string=string) for metric metadata (e.g. host=server01, region=us-west-1, cpu=1). Tags are used in combination with the metric name to identify a metric and provide additional context. A tag can be considered a search index.

### Field Set

Fields are key/value pairs (string=float) for metric values (e.g. user=24.8, system=0.5, idle=73.6)

### Timestamp

The UTC date and time, a Unix nanosecond timestamp, associated with the metric point.

### Examples

Note: I have only used JSON to represent the Sensu Metric Format in this particular example. The actual implementation is likely a Go slice & struct.

```
[
  {
    "name": "cpu_usage",
    "tags": {
      "host": "server01",
      "region": "us-west-1",
      "cpu": 1
    },
    "fields": {
      "user": 24.8,
      "system": 0.5,
      "idle": 73.6
    },
    "timestamp": 1512684071469000000
  }
]
```

Graphite equivalent:

The metric name, tags, fields combine to produce the Graphite metric path, producing multiple metric points.

```
us-west-1.server01.cpu_usage.cpu1.user 24.8 1512684071
us-west-1.server01.cpu_usage.cpu1.system 0.5 1512684071
us-west-1.server01.cpu_usage.cpu1.idle 58.0 1512684071
```

OpenTSDB equivalent:

The metric name and fields combine to produce multiple metric points that share the same tags.

```
cpu_usage.user host=server01,region=us-west-1,cpu=1 24.8 1512684071
cpu_usage.system host=server01,region=us-west-1,cpu=1 0.5 1512684071
cpu_usage.idle host=server01,region=us-west-1,cpu=1 73.6 1512684071
```

## Open issues (if applicable)

https://github.com/sensu/sensu-go/issues/7
