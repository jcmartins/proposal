# Proposal: Sensu Extended Attributes

Author(s): Eric Chlebek

Last updated: 2017-11-21

Discussion at https://github.com/sensu/sensu-go/issues/586

## Abstract

Allow users to set extended attributes on selected resources in Sensu, and
reference these attributes in filters and mutators.

This proposal uses the word "extended" instead of "custom". This is part of
the proposal, to change the wording. This way, we can indicate to customers
that we support the essential feature they want from 1.x, but in a different
way.


## Background

Extended attributes will necessarily involve dynamic behaviour and reflection,
by their nature. Luckily, we can leverage some existing libraries to take the
burden of maintaining piles of reflection away from us.

In Sensu 1.x, existing _custom_ attributes look like the following:
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
structure. Sensu allows users to define custom logic expressions that can
access these custom attributes.

## Proposal

Having arbitrary fields is something of an anti-pattern in Go, but there are
some things we can do to support them. One, already implemented, is using
`govaluate`, which sets up a number of approaches for doing just this.

Another library we can utilize is `jsoniterator`, which lets users arbitrarily
iterate through JSON data structures instead of unmarshalling in a single shot.

By combining these two libraries, we can implement a custom unmarshaler and
marshaler for any data types that want to use extended attributes.

### Reflection-based encoding and decoding helpers

By writing a few helpers, we can take the brunt of the task away from type
implementers. I've included a complete, somewhat-tested proof-of-concept
here: https://github.com/sensu/sensu-go/pull/599

TODO:
1. Support querying nested extended attribute structures. (Done.)
1. Optimize the code by reducing reflection and parsing as much as possible.
(Not yet attempted.)
1. Better naming. (Somewhat done.)
1. Unit tests. (Done, but could be better.)

### Govaluate

Govaluate as of this writing does not allow accessing nested maps or
`Parameters` types. Therefore, we need to fork and fix govaluate to work how we
would like it to.

PR: https://github.com/Knetic/govaluate/pull/84

## Rationale

There are essentially three ways to accomplish this type of dynamic behaviour
at runtime in Go.

1. Reflection
1. Code generation and execution out-of-process
1. Embedded VM, ala lua or v8

In our case, reflection is probably the design decision that
fits best with the libraries we've already chosen, and would allow us to
implement a solution without relying on embedding a VM or running things
outside the main executable's process.

## Compatibility

This feature is necessary to facilitate 1.x parity.

## Implementation

1. Implement a library for dealing with marshaling and unmarshaling data types with extended attributes. Make sure the library handles govaluate integration. (https://github.com/sensu/sensu-go/issues/586)
1. Fork and patch govaluate to support nested Parameters. Try to get it merged upstream. (https://github.com/Knetic/govaluate/pull/84)
1. Make use of the library in the Check data type. (https://github.com/sensu/sensu-go/issues/603)
1. Make use of the library in the Environment data type. (https://github.com/sensu/sensu-go/issues/604)
1. Add support for setting extended attributes in the CLI for Check and Environment. (https://github.com/sensu/sensu-go/issues/458)
