# Proposal: Proxy Check Requests Filters

Author(s): Simon Plourde

Last updated: 2017-01-23

Discussion at https://github.com/sensu/sensu-go/issues/598.

## Abstract

Use govaluate statements to filter entities when issuing proxy check requests.

## Background

Sensu 1.x uses a hash of `client_attributes` in order to match clients to these
attributes, which also support the `eval` function:

```
{
  "checks": {
    "check_arista_eth0": {
      [...]
      "proxy_requests": {
        "client_attributes": {
          "keepalives": false,
          "device_type": "router",
          "device_manufacturer": "arista",
          "subscriptions": "eval: value.include?('dc-01')"
        }
      }
    }
  }
}
```

Supporting the same hash in Sensu 2.x would actually be a complex task since we
have to deal with nested attributes in the standard and custom attributes.

## Proposal

The proposed change consists of replacing the `client_attributes` hash with a
list of govaluate statements, similar to what Filters do in Sensu 2.x:

```
{
  "checks": {
    "check_arista_eth0": {
      [...]
      "proxy_requests": {
        "entity_filters": [
          "entity.Keepalives == false",
          "entity.DeviceType == \"router\"",
          "entity.DeviceManufacturer == \"arista\"",
          "\"dc-01\" in entity.Subscriptions"
        ]
      }
    }
  }
}
```

## Rationale

The `dynamic` package has been wrote with the use of govaluate in mind so if we
wanted to keep the same structure as in Sensu 1.x, we would either have to
replicate what govaluate does or loop through the provided attributes and write
our own govaluate statements behind the scenes, which would probably result in a
complex and error-prone codebase.

## Compatibility

This proposal constitutes a breaking change since the syntax used for matching
entities is changed, so proxy check requests could not be automatically
imported. I don't believe this functionality is widely used but this is totally
subjective and I don't believe we have empirical evidence to support that.

## Implementation

This proposal would be implemented through
https://github.com/sensu/sensu-go/issues/899, which is already in progress (I
have a working proof-of-concept using govaluate).

We already had in mind to (support
govaluate)[https://github.com/sensu/sensu-go/issues/900] but this proposal would
make govaluate expressions the standard.

