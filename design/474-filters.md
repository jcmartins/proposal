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

We propose implementing a somewhat modified version of Sensu 1.x filters that makes the
behavior of the filter and the expression of the filter's intent more straightforward.
This proposal does not include the Sensu 1.x filter "when" functionality (i.e. time-based
filter is out of scope for this issue).

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

Sensu 2.x filter statements must evaluate to boolean expressions. This means that they
require a boolean operator in the expression. Sensu 1.x allowed "truthiness" to determine if
the filter matched. For example, the _existence_ of a field would be enough for it to to match.
In Sensu 2.x, you would have to explicitly do a type-appropriate comparison against the field's
zero-value. For example, if you wanted to make sure the check had _some_ output, you would use
the statement `event.Check.Output != ""`.

In this sense, tokens are any dot-notated Sensu Event attribute. For example `event.Check.State == "passing"`
would first access the event's check object and retrieve the state of the check, then do an equality
comparison to the string "passing". Govaluate provides us the ability to easily parse
these statements and provides access to the event structure when we pass it in as a parameter.

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

When an event comes through the pipeline, load its associated filter(s) from Etcd and
create "machines" that take as input the event and return whether or not that event
should be passed through to the next pipeline "stage." We can add caching for reuse later, 
but for now let's accept the GC pressure rather than having to worry about the complexity 
introduced by cache coherency.

A filter machine applies one or more filter statements (respresented by govaluate
`EvaluableExpression`s) to an event. These statements may include references to fields on the Event
structure in dot-notation. Govaluate provides access to struct fields and methods, see its
documentation on [accessors](https://github.com/knetic/govaluate#accessors) for more information.

When a machine receives an event, it will pass the event along to each govaluate `EvaluableExpression`
as a parameter, making it accessible to the filter statement.

Concretely:

```golang
  expr, err := govaluate.NewEvaluableExpression(`event.Check.Environment == "production"`)
  // handle err
  
  result, err := expr.Evaluate(map[string]interface{}{"event": event})
  // handle err

  match, ok := result.(bool)
  // if !ok, then we should consider that an error, filter expressions
  // must evaluate to boolean values.
  
  // if match && filter.action == allow, don't filter the event, and vice versa
```

### 1.x -> 2.x translation

While we understand it will be a tedious operation, we want to make sure that we provide
an automated import of Sensu 1.x filters into 2.x filters whenever possible. Sensu 1.x 
filters allow you to easily create filters that inspect event attributes by representing
those comparisons as a hash value for the filter's `attributes` field. We should map those
to Sensu 2.x filter expressions whenever possible.

For example, client attribute filters would map to entity comparisons.

```json
{
  "attributes": {
    "client": {
      "environment": "production"
    }
  }
}
```

Becomes:

```json
{
  "statements": [
    "event.Entity.Environment == \"production\""
  ],
  "action": "allow"
}
```

The missing `negate` field in the original filter would become an implicit `allow` action in
Sensu 2.x.

Not all Client or Check fields will be directly mappable. Some of these will be custom
attributes that people set on objects, but those will be dealt with at a later time in
a separate feature. We should reject any filter that uses eval for now.
