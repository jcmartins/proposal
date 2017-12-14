# Proposal: Sensu Metric Format

Author(s): Sean Porter

Last updated: 2017-12-07

Discussion at https://github.com/sensu/sensu-go/issues/7.

## Abstract

The Sensu Metric Format is a data structure for defining one or more metric points. Each metric point has a name, tags, value, and timestamp. This format is used when transporting metrics within Sensu Events ("Metrics": []). Interoperability with metric tools is achieved through format translation, on ingress at the Sensu Agent, and on egress at the Sensu Backend.

## Background

Sensu 1.x was designed and optimized for service checks. Some metric data was expected within service check execution output (e.g. Nagios PerfData), Sensu event mutators and handlers were expected to transform and send the data to a TSDB (e.g. Graphite). As Sensu adoption grew, it quickly became apparent that users wanted to use Sensu for metric collection. The community created tooling with a focus on metric collection and delivery with Sensu, such as WizardVan (authored by Greg Poirier), which improved Sensu's capabilities. While many users are very successful in using Sensu 1.x for metric collection, there remains room for improvement in regards to the user experience and performance.

## Proposal

Create a Sensu Metric Format for defining one or more metric points. Use the format when transporting metrics within Sensu Events. Using a single metric format within Sensu enables data validation and performance optimization. Sensu Event Mutator and Handler authors only need to concern themselves with a single metric format (consistency!). Having a defined metric format enables instrumentation libaries to create and send Sensu Events containing metrics to a Sensu Events API. This makes metrics first-class.

## Rationale

### Use Metrics 2.0

Instead of defining an new metric format, Sensu 2.0 could use [Metrics 2.0](http://metrics20.org/), "an emerging set of conventions, standards and concepts around timeseries metrics metadata". While the goal and idea of a common modern format is fantastic, the collaboration on and adoption of the [spec](http://metrics20.org/spec/) does not seem to be there. The flat map of tags is not ideal, there is no indication of the nature of a tag (describer or value?).

### Use InfluxDB

The InfluxDB metric data model is an option, the format is capable of defining one or more measurements, with each measurement representing one or more metric points. Each measurement has a name, tags, fields, and timestamp. This format exchanges "value" for "fields", key/value pairs (string=float) for metric data (e.g. value=0.5, user=24.8, temperature=56.5). Using this format would enable Sensu to fully support and utilize the InfluxDB metric data model.

Pros:

- Fully support the InfluxDB data model
- Multiple data points that share a common set of tags (efficiency)

Cons:

- Requires more explanation
- Potentially more difficult to translate to and from
- Fields may be excessive in many cases

### Do Nothing

Not implementing a metric format would force users to push metrics through Sensu using Event Check output.

## Compatibility

Given the Sensu 2.0 Event still contains Check output, users can still push metrics through Sensu using the same method used in 1.x. However, there is an issue with this, as the Sensu 2.0 Event does not provide an attribute to indicate that the Check output contains metric data (1.x used check `"type": "metric"`). Currently, the Backend inspects Event "Metrics" to determine if there are metric points to go through the Event pipeline. This attribute would need a proposal. A GitHub issue has been filed for this, [Check output metric extraction](https://github.com/sensu/sensu-go/issues/745).

## Implementation

A metric format capable of representing one or more metric points. The metric data can originate from a variety of metric tools and formats.

Each metric point has a name, tags, value, and timestamp.

### Name

The metric name provides a sense of metric identity and context (e.g. name=cpu_usage.user). Because this proposed spec has tags, the complete metric identity does not need to be encoded in the metric name.

### Tags

Tags are key/value pairs (string=string) for metric metadata (e.g. host=server01, region=us-west-1, cpu=1). Tags are used in combination with the metric name to identify a metric and provide additional context. A tag can be considered a search index.

### Value

The metric value (a float, e.g. 24.8).

### Timestamp

The UTC date and time, a Unix nanosecond timestamp, associated with the metric point.

### Examples

Note: I have only used JSON to represent the Sensu Metric Format in this particular example.

```
[
  {
    "name": "cpu_usage.user",
    "tags": [
      {
        "name": "host",
        "value": "server01"
      },
      {
        "name": "region",
        "value": "us-west-1"
      },
      {
        "name": "cpu",
        "value": "1"
      }
    ],
    "value": 24.8,
    "timestamp": 1512684071469000000
  },
  {
    "name": "cpu_usage.system",
    "tags": [
      {
        "name": "host",
        "value": "server01"
      },
      {
        "name": "region",
        "value": "us-west-1"
      },
      {
        "name": "cpu",
        "value": "1"
      }
    ],
    "value": 0.5,
    "timestamp": 1512684071469000000
  }
]
```

Graphite equivalent:

```
us-west-1.server01.cpu1.cpu_usage.user 24.8 1512684071
us-west-1.server01.cpu1.cpu_usage.system 0.5 1512684071
```

OpenTSDB equivalent:

```
cpu_usage.user host=server01,region=us-west-1,cpu=1 24.8 1512684071
cpu_usage.system host=server01,region=us-west-1,cpu=1 0.5 1512684071
```

InfluxDB equivalent:

```
cpu_usage.user,host=server01,region=us-west-1,cpu=1 value=24.8 1512684071469000000
cpu_usage.system,host=server01,region=us-west-1,cpu=1 value=0.5 1512684071469000000
```

Prometheus equivalent:

```
cpu_usage.user{host="server01", region="us-west-1", cpu="1"} 24.8
cpu_usage.system{host="server01", region="us-west-1", cpu="1"} 0.5
```

### Transforming Formats

Some metric formats make it difficult (or impossible) to extract tags, particularly the Graphite plaintext format.

For example, without understanding exactly what each Graphite metric path segment represents, the metric name (cpu_usage.user), and the tags (server01, us-west-1, cpu1) cannot be extracted confidently.

```
us-west-1.server01.cpu1.cpu_usage.user 24.8 1512684071
```

In the Sensu Metric Format:

```
[
  {
    "name": "us-west-1.server01.cpu1.cpu_usage.user",
    "tags": [],
    "value": 24.8,
    "timestamp": 1512684071469000000
  }
]
```

The OpenTSDB format allows for better tag extraction.

```
cpu_usage.user host=server01,region=us-west-1,cpu=1 24.8 1512684071
```

```
[
  {
    "name": "cpu_usage.user",
    "tags": [
      {
        "name": "host",
        "value": "server01"
      },
      {
        "name": "region",
        "value": "us-west-1"
      },
      {
        "name": "cpu",
        "value": "1"
      }
    ],
    "value": 24.8,
    "timestamp": 1512684071469000000
  }
]
```

## Open issues (if applicable)

https://github.com/sensu/sensu-go/issues/7
