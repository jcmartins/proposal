# Proposal: Sensu 2.x Filters

Author(s): Greg Poirier, Sean Porter

Last updated: 2017-10-23

Discussion at https://github.com/sensu/sensu-go/issue/474.

## Abstract

Implement 1.x Filters in 2.x using the govaluate library to provide something akin to 1.x's ability
to evaluate arbitrary Ruby statements. We introduce a limited dot-notation format for inspecting an
event's attributes and expose in documentation as much of govaluate as we feel initially comfortable.

## Background

Sensu 1.X provides Event Filters, a Sensu event pipeline primitive/component that can negate (drop) 
events that match configured attributes, stopping them from being mutated and handled. See the 
documentation link below for a more detailed explanation.

[Sensu 1.X Event Filters](https://sensuapp.org/docs/1.0/reference/filters.html#what-are-sensu-filters)

Sensu 2.X will need to implement filters differently, as Golang is not able to do some of the things that 
Ruby can do, e.g. eval(). We can streamline filters in 2.X, using one or more govaluate statements, 
to be evaluated with event data. The statements are AND'd. Filters will be implemented within pipelined.

## Proposal

### Filter Type

Sensu 1.x filters are configured as follows:

```json
"production_filter": {
  "negate": false,
  "attributes": {
    "client": {
      "environment": "production"
    }
  }
}
```

If true, `negate` indicates that the filter rejects events matching these attributes. 
In contrast, 2.x filters have an `action` field that specifically states `allow` or `deny`.

Filters in Sensu 2.x look like this:

```golang
type EventFilter struct {
  // Name is a unique identifier for the filter.
  Name string `json:"name"`

  // Organization is the organization to which this filter belongs
  Organization string `json:"organization"`

  // Environment is the environment to which this filter belongs.
  Environment string `json:"environment"`

  // Statements is an array of boolean expressions that are &&'d together
  // to determine if the event matches this filter.
  Statements []string `json:"statements"`
  
  // Action is one of "allow" or "deny" and specifies whether events are passed
  // through the filter or blocked by the filter.
  Action string `json:"action"`
}
```

### API

Sensu 2.x filters will be managed via the API (as opposed to configuration files in
Sensu 1.x). The API is fairly straightforward and closely resembles the `handlers` API.

Filters are namespaced by organization and environment with the `org` and `env` query
parameters.

GET /filters - Return an array of all filters currently configured.

GET /filters/:name - Return a single, named filter.

PUT/POST /filters/:name - Create/Update a single, named filter.

DELETE /filters/:name - Delete a single, named filter.

### Filter statements

In Sensu 1.x, if an attribute began with `eval:`, Ruby would `eval()` the following
statement and return a value which was used to determine if the attribute matched.

Go does not have this "luxury", and so we are introducing two mechanisms for replicating
functionality similar to that which is available in Sensu 1.x. Sensu 2 filters will have
a list of statements which are logically AND'd together after individual evaluation.
These statements contain tokens and operators as provided by the [govaluate](https://github.com/Knetic/govaluate) 
library. 

In this sense, tokens are any dot-notated Sensu Event attribute. For example `event.check.state == "passing"`
would first access the event's check object and retrieve the state of the check, then do an equality
comparison to the string "passing". Govaluate provides us the ability to easily parse
these statements and access tokens for substitution.


## Rationale

### Backward compatible options

There are two mechanisms for providing backward compatibility. A separate running Ruby VM that we communicate
with over sockets or an embedded Ruby VM within Sensu 2.x. We, hilariously, actually have these options! There 
exist libraries for embedding Ruby VMs in Go (see: https://github.com/AlekSi/gomruby).

These are obviously untenable. A separate Ruby VM is operational and packaging complexity we are unwilling
to introduce, and an embedded Ruby VM is out of the question.

### Not backward compatible options

We could introduce Runtime `eval` into Go using a library like [apaxa-go/eval](https://github.com/apaxa-go/eval).
However, we don't believe that forcing users to learn Golang's syntax or exposing raw Go structs to users is a 
good user experience.

There's a decent amount of common ground between govaluate and Ruby eval that allows people to compose fairly
sophisticated boolean expressions to allow or deny events based on them. Furthermore, we can introduce more
complex functionality into the filter statements at a later time.

## Compatibility

This fundamentally changes the way that filters work in Sensu 2.x. Eval is no longer available, and
the data type itself is changing. The latter is at least partially addressable with a `sensuctl` import
routine, but the former will require considerable documentation and user education to address.

## Implementation

### Filter type

```golang
type EventFilter struct {
  // Name is a unique identifier for the filter.
  Name string `json:"name"`

  // Organization is the organization to which this filter belongs
  Organization string `json:"organization"`

  // Environment is the environment to which this filter belongs.
  Environment string `json:"environment"`

  // Statements is an array of boolean expressions that are &&'d together
  // to determine if the event matches this filter.
  Statements []string `json:"statements"`
  
  // Action is one of "allow" or "deny" and specifies whether events are passed
  // through the filter or blocked by the filter.
  Action string `json:"action"`
}
```

### Validation

`Action` may only be one ["allow" or "deny"].

`Statements` must be one or more strings. Each statement should be run through govaluate's parser
to validate it. This is easily accomplished by attempting to create an `EvaluableExpression`:

```golang
if _ err := NewEvaluableExpression(statement); err != nil {
  // Failed to parse, return err indicating invalid
  return err
}
```

### API

Filters are namespaced by organization and environment with the `org` and `env` query
parameters.

GET /filters - Return an array of all filters currently configured.

GET /filters/:name - Return a single, named filter.

PUT/POST /filters/:name - Create/Update a single, named filter.

DELETE /filters/:name - Delete a single, named filter.

Filters should be namespaced in etcd using a key builder within the "filters" path.

### Pipelined



### 1.x -> 2.x translation

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not
know the solution. This section may be omitted if there are none.]
