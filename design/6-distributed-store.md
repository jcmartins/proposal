# Proposal: Distributed Storage for Sensu Backends

Author(s): Greg Poirier, Sean Porter

Last updated: 2017-11-28

Discussion at https://github.com/sensu/sensu-go/issues/6.

## Abstract

An embedded, distributed data store, will allow Sensu 2.x to operate without requiring an external configuration synchronization mechanism. Etcd provides an implementation of such a data store that is embeddable within other Golang processes that provides the necessary sufficient durability, consistency, and transaction isolation guarantees for operating a Sensu 2.x cluster.

## Background

One of the primary goals of the Sensu 2.x project is to reduce the operational burden and complexity of Sensu installations. In Sensu 1.x, operators are _effectively_ required to operate an external configuration management system (e.g. Chef, Puppet, Ansible), an external transport (e.g. RabbitMQ), and a data store (Redis). 

Typically, Sensu is used in environments with existing configuration management systems, but this is effectively a barrier to entry for users of the proeduct. A distributed data store can assume the responsibility for replicating configuration and ensuring consistent state across all Sensu 2.x Backend servers. This removes that requirement and lessens the barrier to entry for potential Sensu users--and, in some cases, for users who have _no_ monitoring for their production environment today.

While running a single instance of Redis is fairly trivial, many users are concerned with having a single point of failure in highly available production Sensu installations. This leads to extraordinary complexity in their production environments necessitated by Redis--which was never originally intended to be run in any highly available configuration. Furthermore, many users have concerns about the durability of the data stored within Redis which has been shown to suffer catastrophic data loss in certain configurations [1].

Redis also suffers from many as-of-yet unsolved security issues. There is no proper authentication or authorization mechanism built-in to Redis. For users with data integrity concerns (i.e. most users in production environments), this is intractably problematic. Solving this issues requires sophisticated technology investment by users involving multiple external services.

## Proposal

We propose embedding an Etcd distributed key-value store within the Sensu Backend processes instead of using Redis to store the state of the monitoring system and requiring configuration management to synchronize configuration files between Backends. Etcd is a distributed datastore written in Golang based on the RAFT consensus protocol [2] and BoltDB [3]. It provides an [Embed API](https://godoc.org/github.com/coreos/etcd/embed) for embedding Etcd processes within other Golang processes.

## Rationale

Embedding Etcd centralizes the operational complexity of Sensu to only Sensu. Users who do not require highly available Sensu environments can simply run a single Sensu Backend process simplifying the installation process from multiple components (Sensu, Redis, Configuration Management) to a single component (Sensu). In highly available environments, there is some operational complexity assumed, but this is localized only to Sensu.

Since Sensu 2.x is written in Go, and the goal is to embed the distributed key-value store within Sensu, it is a requirement that either the key-value store have a native implementation in Go or we must use CGO to link the Go binary with a library for the embedded store. This leaves us with very few acceptable "pre-packaged" options, and if we limit those options further yet to in-memory datastores, in Go, that include consensus protocols we are left with only a single option.

It's also noteworthy, that the most likely alternative to this is building our own distributed database likely atop BoltDB as well. This would be the equivalent of writing a competitor to Etcd while adding little to no value to our own product.

## Compatibility

Not compatible with Sensu 1.x.

## Implementation

[A description of the steps in the implementation, who will do them, and when.

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not
know the solution. This section may be omitted if there are none.]

# References

[1] Jepsen: Redis, Kyle Kingsbury, https://aphyr.com/posts/283-jepsen-redis
[2] In Search of an Understandable Consensus Algorithm, Ongaro and Ousterhout, https://raft.github.io/raft.pdf
[3] BoltDB, https://github.com/boltdb/bolt
