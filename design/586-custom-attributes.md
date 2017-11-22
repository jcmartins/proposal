# Proposal: Sensu Custom Attributes

Author(s): Eric Chlebek

Last updated: 2017-11-21

Discussion at https://github.com/sensu/sensu-go/issues/586

## Abstract

Allow users to set custom attributes on selected entities in Sensu, and
reference these entities in filters and mutators.

## Background

Custom attributes will necessarily involve dynamic behaviour and reflection,
by their nature. Luckily, we can leverage some existing libraries to take the
burden of maintaining piles of reflection away from us.

In Sensu 1.x, existing custom attributes look like the following:
```
{
  "checks": {
    "check_mysql_replication": {
      "command": "check-mysql-replication-status.rb --user sensu --password secret",
      "subscribers": [
        "mysql"
      ],
      "interval": 30,
      "playbook": "http://docs.example.com/wiki/mysql-replication-playbook"
    }
  }
}
```

Notice the 'playbook' field, which is not normally described by the check data
structure. Sensu allows users to define custom logic that access these custom
attributes.

## Proposal

Having arbitrary fields is something of an anti-pattern in Go, but there are
some things we can do to support them. One, already implemented, is using
`govaluate`, which sets up a number of approaches for doing just this.

Another library we can utilize is `jsoniterator`, which lets users arbitrarily
iterate through JSON data structures instead of unmarshalling in a single shot.

By combining these two libraries, we can implement a custom unmarshaler and
marshaler for any data types that want to use custom attributes.

## Reflection-based encoding and decoding helpers

By writing a few helpers, we can take the brunt of the task away from type
implementers. I've included a complete, somewhat-tested proof-of-concept
here: https://github.com/sensu/sensu-go/pull/599

TODO:
1. Support querying nested custom attribute structures. (Done.)
1. Optimize the code by reducing reflection and parsing as much as possible.
(Not yet attempted.)
1. Better naming. (Somewhat done.)
1. Unit tests. (Done, but could be better.)

## Govaluate

The problem with govaluate is that it does not allow accessing nested maps or
`Parameters` types. There are two potential ways we can solve this issue:

1. Fork and fix govaluate.
1. Dynamically generate structs from JSON using reflect.

Both solutions will necessitate reflection, and forking govaluate will incur
additional maintenance overhead, especially if we don't manage to get it merged
into mainline.

## Rationale

There are essentially three ways to accomplish this type of dynamic behaviour
at runtime in Go.

1. Reflection
1. Code generation and execution out-of-process
1. Embedded VM, ala lua or v8

I believe that in our case, reflection is probably the design decision that
fits best with the libraries we've already chosen, and would allow us to
implement a solution without relying on embedding a VM or running things
outside the main executable's process.

You can see the result of my experiment with the first approach in the PR
attached above. The unit tests show it is possible to access arbitrary
nested parameters with govaluate. This is because GetField actually dynamically
generates a struct to pass to govaluate.

*Dynamically generating structs is a brutal, horrible hack*. It would be better
to introduce improved support for the govaluate.Parameters interface within
govaluate, where for some reason it is under-utilized. I suspect that if we can
improve support for govaluate.Parameters, it won't be necessary to generate a
struct at all, and rather just return a tree of lazy jsoniter.Any values,
each of which implements the Parameters interface.

## Compatibility

This feature is necessary to facilitate 1.x parity.

## Implementation

1. Implement a library for dealing with marshaling and unmarshaling data types
with custom attributes. Make sure the library handles govaluate integration.
1. Make use of the library in the Check data type.
1. Make use of the library in the Environment data type.
