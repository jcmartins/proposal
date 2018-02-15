# Proposal: Built-in Filters

Author(s): Sean Porter

Last updated: 2018-02-15

Discussion at https://github.com/sensu/sensu-go/pull/1002

## Abstract

Move the fixed event pipeline filtering logic into built-in named filters, e.g. "is_incident". An event handler will receive every event unless it uses one or more event filters (built-in and/or custom), e.g. `["is_incident", "not_silenced"]. No default filtering logic and built-in filters allow for the complete control of the event pipeline, eliminating the inconsistent and restrictive behaviours of the current event pipeline.

## Background

In the beginning, Sensu was designed to only execute boolean service checks, its event pipeline was optimized for incident processing and management. Built-in logic determined which events indicated an incident or state change (i.e. resolution), only these events would be handled. As Sensu evolved and began to collect other kinds of monitoring data, such as metrics, the built-in logic was altered to optionally allow non incident events to be handled (e.g. check `"type": "metric"`). If a check monitored service availability and collected metrics (a "hybrid check"), its check type had to be "metric", and its incident handlers (e.g. pagerduty) needed custom filters to re-implement the logic for incident and state change determination. The Sensu event pipeline can be difficult to explain to new users, due to the fact it treats certain events differently (incidents vs metrics).

With the Sensu 2.0 Alpha, there currently is no way to handle an event for a check execution that does not indicate an incident (or resolution). There is no check `"type": "metric"` or equivalent feature.

## Proposal

Move the fixed event pipeline filtering logic into built-in named filters. This includes the event filtering functions: `isIncident()`, `isResolution()`, `hasMetrics()`, and `isSilenced()`.

## Rationale

Built-in named filters allows event handlers to dictate exactly which events they receive. This also reduces the number of additional pipeline control features required to circumvent filtering logic, e.g. check "type" or handler "severities".

## Compatibility

This is a breaking change, not compatible with Sensu 1.x nor the current Sensu 2.x Alpha release.

## Implementation

POC implementation and next steps discussion can be found at https://github.com/sensu/sensu-go/pull/1002
