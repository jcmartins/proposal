# Proposal: Sensu Backend Events Errors Storage

Author(s): Simon Plourde

Last updated: 2017-11-10

Discussion at https://github.com/sensu/sensu-go/issues/459.

## Abstract

Capture all errors through the [Sensu
Pipeline](https://github.com/sensu/engineering/wiki/Sensu-Pipeline) while
processing an event in the backend and store them so they can be easily queried.

## Background

Sensu v1 currently logs most backend event errors into its log files. While this
approach is required and useful in some cases, the user experience could be
improved, especially when running fairly large clusters (e.g. an operator has
to determine which machine processed a specific event, login that machine
using SSH and analyze the log files to retrieve the corresponding entries).

## Proposal

The proposed change consist of temporarily adding any exception, while handling
events through the Sensu Pipeline, to an _error_ Store so they can be easily
retrieved.

### Error Type

We would need to introduce a new type for errors that look like this:

```
{
  "error": {
    "component": "filter",
    "message": "attribute type mismatch"
    "name": "filter_production",
  }
  "event": { // That would essentially be the already existing Event struct
    "check" : [...],
    "entity" : [...],
    "timestamp": 1509734718
  }
}
```

### Store

An error is located at the following path in the Store:

```
/errors/{organization}/{environment}/{entity}/{check}/{timestamp}

# E.g.
/errors/acme/dev/i-424242/check_cpu/1509734718

```

We also attach a lease to every key added within the errors Store, using a TTL
of 8 hours, so we don't end up creating a log management system within Sensu;
log files would still have to be used in case a user need to go further back in
time.

### API

Sensu v2 errors will be managed via the API, which is pretty straightforward.

- Filters are namespaced by organization and environment with the org and env
query parameters.
- `GET /errors` returns an array of all errors
- `GET /errors/:entity` returns an array of all errors for a given entity
- `GET /errors/:entity/:check` returns an array of all errors for given
entity & check
- `GET /errors/:entity/:check/:timestamp` returns a single error
- `DELETE /errors/:entity` deletes all errors for a given entity
- `DELETE /errors/:entity/:check` deletes all errors for a given entity and check
- `DELETE /errors/:entity/:check/:timestamp` deletes a specific error

Adding and updating errors is not possible through the API, since it's
reserved to the backend processes. That being said, it could always be possible
to allow agents to add errors to the Store if required.

### Error CLI

Sensu v2 errors can be managed through the CLI, which implements the
corresponding API endpoints:

- `sensuctl error list` returns all errors
- `sensuctl error list [ENTITY]` returns all errors for a given entity
- `sensuctl error list [ENTITY] [CHECK]` returns all errors for a given entity
and check
- `sensuctl error info [ENTITY] [CHECK] [TIMESTAMP]` returns a single error
- `sensuctl error delete [ENTITY]` deletes all errors for a given entity
- `sensuctl error delete [ENTITY] [CHECK]` deletes all errors for a given entity
and check
- `sensuctl error delete [ENTITY] [CHECK] [TIMESTAMP]`

### Event CLI

In addition to the standard subcommands to cover the errors API, the `event
info` subcommand requires modification so that we also indicate to the user if
errors were recently recorded for this event. From this point, the user can then
decide to run the `error info` subcommand.

## Rationale

### Storing errors within the event

The initial intention was to add errors directly within events, something along
the lines of:

```
{
  "entity": {
    "...": "..."
  },
  "...": "..."
  "pipeline_errors": [
    {
      "component": "filter",
      "thing": "filter_production",
      "message": "attribute type mismatch..."
    }
  ]
}
```

**Advantages**

- Any error encountered is right in your face when looking at an event.

**Disadvantages**

- We only keep the last event, so we would need to mutate the event every
 time we receive a new one in order to keep an history of past events.
- Difficult to aggregate in order to get a better picture of the situation
(e.g. happen consistently across all events).
- It would require synchronization between eventd and pipelined via a mutex in
etcd.

## Compatibility

This is a new feature in Sensu v2.

## Implementation

In order, here's the user stories required for this feature:

1. Add the error type (see [details](#error-type)):
https://github.com/sensu/sensu-go/issues/551
2. Add the error Store (see [details](#store)):
https://github.com/sensu/sensu-go/issues/552
3. Add the error API (see [details](#api)):
https://github.com/sensu/sensu-go/issues/553
4. Add the error CLI command (see [details](#error-cli)):
https://github.com/sensu/sensu-go/issues/554
5. Update the `event info` subcommand (see [details](#event-cli)):
https://github.com/sensu/sensu-go/issues/555
