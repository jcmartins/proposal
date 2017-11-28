# Proposal: Agent Transport

Author(s): Sean Porter, Greg Poirer

Last updated: 2017-03-15

## Abstract

The Sensu Agent Transport is the method of communication between Sensu Agents and the Sensu Backends. It is capable of traversing complex network topologies while providing end-to-end encryption. The same method of authentication and authorization is used as Sensu API clients. Connections are initiated by the Sensu Agent, only the Sensu Backend needs to expose a port and listen for connections, reducing the operational overhead and attack surface. The Sensu Agent only needs to maintain its transport connection to one of the available Sensu Backends at a time, but may connect to another in the event of connectivity issues.

## Background

The next generation of Sensu must retain the transport and messaging functionality available in the current version of Sensu. This includes end-to-end encryption, user authentication and authorization, high availability, and message routing (i.e. publish/subscribe topics).

Today, Sensu leverages third party software, such as RabbitMQ, to provide secure connectivity and messaging between Sensu components. Unfortunately, third party software (e.g. RabbitMQ) introduces a significant barrier to entry, operational burden, and potential security vulnerabilities. Users are expected to have the knowledge and experience necessary to deploy, secure, and scale the third party software. In order to address this problem, Sensu Inc. provides training courses and materials for specific third party software within the scope of a Sensu installation, namely RabbitMQ and Redis. Although training is available, a good majority of bugs and support tickets filed are still due to third party software issues and difficulties operating it.

One of the primary drivers behind Sensu 2.0, the next generation of Sensu, is reducing Sensu's operational overhead. Today, the transport represents the largest source of such friction for new users in their production environments.

## Proposal

Use a simple carriage-return, line-feed delimited JSON protocol, over a WebSocket connection for Sensu Agent to Backend communication. Using a WebSocket allows us to leverage the Sensu API authentication and authorization mechanism, providing full RBAC for Sensu Agent connections. TLS can be used to encrypt communication and x509 certificates can be used to verify the Sensu Agent and Backend peers' identities. Message routing can be implemented within the Sensu Backend process. A Sensu Backend only needs to route messages to its connected Agents, little to no coordination is required between Backends.

The WebSocket Protocol enables two-way communication between a client running untrusted code in a controlled environment to a remote host that has opted-in to communications from that code. The security model used for this is the origin-based security model commonly used by web browsers. The protocol consists of an opening handshake followed by basic message framing, layered over TCP. The goal of this technology is to provide a mechanism for browser-based applications that need two-way communication with servers that does not rely on opening multiple HTTP connections (e.g., using XMLHttpRequest or IFrames and long polling).

## Rationale

### Simplicity

Using a simple line protocol over a WebSocket and implementing message routing in the Sensu Backend allows us to avoid writing a distributed messaging/queuing system.

### Community

Using a simple line protocol over a WebSocket will allow us to provide a Sensu communication library to the community, for custom tooling.

## Compatibility

The Sensu 2.0 Agent Transport eliminates the need for third party software like RabbitMQ. It is not compatible with Sensu 1.x.

Using Websockets allows us some noteworthy forward compatibility options. The Websocket protocol is independent of the encoding used to communicate between Sensu agent and backend processes. Because Websockets use standard HTTP headers to negotiate their connections, we can allow for multiple versions of the transport to be deployed in a single production installation. This gives Sensu the flexibility to change the encoding used to improve performance in the future.

## Implementation

[gorilla/websocket](https://github.com/gorilla/websocket) is the underlying library that the Sensu Agent Transport is based on. This library provides WebSocket upgrade and handshake functionality an HTTP session as provided by `net/http` or another HTTP server like `gorilla/mux`.

There are a few things worth understanding about the behavior of a [Conn](https://godoc.org/github.com/gorilla/websocket#Conn) object. First, it is basically a wrapper around a [net.Conn](https://godoc.org/net#Conn) that handles formatting and framing messages for the Websocket wire protocol as specified in [RFC 6455](https://tools.ietf.org/html/rfc6455). It provides a couple of different methods for sending and receiving messages over the `net.Conn`, but we are primarily concerned with the following four:

- ReadMessage()
- WriteMessage()
- NextReader()
- NextWriter()

You can lookup their documentation fairly easily in the godoc for [gorilla/websocket](https://godoc.org/github.com/gorilla/websocket). What is important to understand, however, are the interactions between these methods and the underlying net.Conn.

### A note about concurrency

First, and foremost, it should be understood (and the gorilla/websocket documentation makes this abundantly clear) that a `Conn` object cannot be used concurrently by multiple goroutines. This will cause a `panic()` and halt the calling goroutine. Each of these messages is effectively calling `Read()` and `Write()` directly on the underlying `net.Conn`. The Websocket protocol is a framed protocol, and the websocket `Conn` object is responsible for ordering and assembling Websocket frames. Thus, concurrent access breaks any guarantees to consistent framing of messages across the transport.

### Reading

Reading from the underlying net.Conn is handled by a [bufio.Reader](https://godoc.org/bufio#Reader). When calling `ReadMessage()` or `NextReader()`, the call will block indefinitely until the next message is received. There's no way of getting around this that does not corrupt the state of the underlying connection. Eventually, the connection will be closed, and the read method should return an error. There might be a temptation to make a call to `SetReadDeadline()` for read attempts, allowing the call to return, this would be a mistake. The websocket `*Conn` does not handle this scenario well at all.

### Writing

Writing is buffered by a byte array, but messages that exceed the size of that array will still be sent in a single frame. If during the process of writing the buffered data to the network (for a single message), the `net.Conn` object returns an error from a call to its `Write()` method, then either the `Writer` returned from `NextWriter()` or `WriteMessage()` itself will return an error.

In either the case of reading or the case of writing, we should only ever lose a single message. The WebSocket `*Conn` object does not make any attempt to buffer whole messages and the library should be thought of as synchronous streaming communication.

## Open issues (if applicable)

N/A
