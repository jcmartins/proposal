# Proposal: Sensu 2.x Extensions

Author(s): Eric Chlebek, Greg Poirier

Last updated: 19/02/2018

Discussion at https://github.com/sensu/sensu-go/issues/1043

## Abstract

This document proposes a specification for the Sensu extensions API. The API
will consist of a client-server gRPC interface that is known to Sensu. Sensu
will use its generated gRPC client to make requests to one or more registered
extension handlers, which may be defined by us or by third parties.

## Background

Sensu 1.x had the ability to execute arbitrary, user defined handlers. This
ability was facilitated by the Ruby VM being able to import and execute
user-defined code. In Sensu 2.x, the Go runtime makes this difficult to
achieve. We would have to embed a VM inside of the Sensu process to achieve
a similar result, which is undesirable for a number of reasons.

## Proposal

Eventd will look for extension handlers in its events with each event emitted.
When an extension handler mapping is found, Eventd will acquire a gRPC client
and connect to the extension handler service if it is registered. The client
will send an EventRequest to the service.

TODO: decide if the service process will be supervised by Sensu, or by an
external provider.

## Rationale

gRPC presents a well-defined and well-understood way to interface with software
components written in a large variety of languages. Instead of inventing a new
way of interop, we can leverage a large body of existing work that is generally
accepted by the industry. Many organizations already have gRPC expertise, and
as such, can implement extension servers for Sensu with a modest amount of
effort.

## Compatibility

This proposal (necessarily) breaks away from Sensu 1.x, and as such
compatibility shims need to be developed in order to support legacy extensions.
We will develop a Ruby service that implements the API, so that existing Ruby
extensions can be supported. This will let us dogfood the API, and also serve
as an example to third-party implementers.

This will very likely break compatibility for existing Ruby extensions that
expect to be able to peer into the Sensu implementation, or modify the running
Sensu instance. However, given the constraints, this system is a best-effort
approach to preserving compatbility for well-behaved Ruby extensions.

## Implementation

1. Design the gRPC API and service definition.
1. Modify eventd to send EventRequests to mapped extension handlers.
1. Implement a Ruby extension server for running Sensu Ruby extensions.
