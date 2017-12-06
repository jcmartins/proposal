# Proposal: Sensu Metric Format

Author(s): Sean Porter

Last updated: 2017-12-06

Discussion at https://github.com/sensu/sensu-go/issues/7.

## Abstract

The Sensu Metric Format is a data structure for defining one or more metric points. Each metric point represents a measurement, with a tag set, field set, and timestamp. This format is used when transporting metrics within Sensu Events ("Metrics": []). Interoperability with metric tools is achieved through format translation, on ingress at the Sensu Agent, and on egress at the Sensu Backend. This format was inspired by the InfluxDB line protocol.

## Background

[An introduction of the necessary background and the problem being solved by the proposed change.]

## Proposal

[A precise statement of the proposed change.]

## Rationale

[A discussion of alternate approaches and the trade offs, advantages, and disadvantages of the specified approach.]

## Compatibility

[A discussion of the change with regard to the [Compatibility](https://github.com/sensu/proposal/#compatibility) section of the README.]

## Implementation

[A description of the steps in the implementation, who will do them, and when.

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not
know the solution. This section may be omitted if there are none.]
