# Introduction to Zenoh

This chapter defines what Zenoh is, where it comes from, how it is structured internally, and when it is the right tool for the job.

## What Zenoh Is

Zenoh is a unified protocol for publish/subscribe, querying, and distributed storage. All three operations share a single hierarchical keyspace, so a publisher, a subscriber, and a queryable storage node can all operate on the same named resource without any translation layer between them.

The three fundamental operations are:

- **Put** — write a value to a key. Any subscriber matching that key receives the sample. Any storage node covering that key persists it.
- **Subscribe** — receive a stream of samples as they are published to keys matching a key expression.
- **Get** — issue a query against a key expression and receive replies from queryables and storage nodes that hold matching data.

The three corresponding abstractions are:

- **Publisher** — declares intent to write to a key expression and sends samples efficiently.
- **Subscriber** — declares interest in a key expression and receives samples from publishers.
- **Queryable** — registers a handler for queries arriving on a key expression and sends replies back to the requester.

Zenoh is not a broker protocol. There is no central message broker that all traffic passes through. Sessions form direct peer-to-peer connections when possible, and route through infrastructure nodes (routers) only when topology requires it. This design lets Zenoh run on microcontrollers with tens of kilobytes of RAM, on embedded Linux gateways, and on datacenter servers, all using the same wire protocol and the same API surface.

## Protocol Genealogy

Zenoh descends from two earlier Eclipse projects: FIWARE NGSI and FNMS, the latter being a ZeroMQ-based middleware. Both were designed for IoT and machine-to-machine communication, and both exposed limitations when extended to edge-to-cloud scenarios: NGSI was REST-only with high per-message overhead, and FNMS required a broker topology that did not scale well across heterogeneous networks.

Zenoh was designed to learn from three well-established protocols:

- **DDS (Data Distribution Service)** — Zenoh adopts the data-centric model and the concept of a shared keyspace (analogous to DDS topics and the global data space). It discards the heavy QoS negotiation machinery and the RTPS wire format.
- **MQTT** — Zenoh adopts the lightweight publish/subscribe model and the hierarchical topic structure. It extends the model with wildcard queries and queryable storage, which MQTT does not natively support.
- **REST** — Zenoh adopts the request/reply semantics of HTTP GET, treating distributed storage as a resource-addressed archive rather than an ephemeral message queue.

The result is a protocol that can unify telemetry (pub/sub), command-and-control (get/reply), and data storage (queryable) in a single abstraction, without forcing the programmer to bridge between three different middleware layers.

## Core Architecture

Zenoh defines three operational modes for a session:

- **Peer** — the default mode. A peer participates in multicast scouting to discover other peers on the same network segment, then establishes direct connections. No infrastructure node is required. Suitable for local networks and development environments.
- **Client** — a thin mode for constrained nodes. A client connects to one or more routers and delegates all routing to them. Clients do not route for other nodes.
- **Router** — a full infrastructure node. Routers connect to other routers and serve clients. They forward messages between network segments and can host storage nodes.

Transport flexibility is a first-class design goal. The same Zenoh session can communicate over:

- TCP (reliable, ordered, connection-oriented)
- UDP (unreliable, used for multicast scouting and datagram transports)
- QUIC (reliable, multiplexed, with built-in TLS)
- TLS (TCP with certificate-based authentication)
- Shared memory (SHM, zero-copy on the same host)
- Serial (for embedded links)

The transport layer is selected by the endpoint URI scheme (`tcp/`, `quic/`, `tls/`, `shm/`). Multiple transports can be active simultaneously on a single session.

Wire overhead is minimal by design: a Zenoh data message requires as few as 4 bytes of framing overhead. The session is the entry point to everything. A session is opened with a configuration, and all publishers, subscribers, and queryables are declared on that session. All transport, routing, and resource management flow through it.

## Performance

Zenoh's wire format is intentionally compact. Comparing minimum framing overhead:

| Protocol | Minimum framing overhead |
|----------|--------------------------|
| Zenoh    | 4 bytes                  |
| MQTT 5   | 5+ bytes                 |
| DDS/RTPS | 36+ bytes                |

This difference compounds at high message rates. Smaller frames mean more messages fit in a single network packet, reducing syscall frequency and CPU time spent in the network stack.

Benchmark figures from the Zenoh introductory paper and zenoh.io:

- **Latency**: sub-10 microseconds on loopback when shared memory transport is active.
- **Throughput**: exceeds 3.5 Gbit/s with SHM on the same host, because the payload never crosses the user/kernel boundary.
- **Latency vs DDS on LAN**: approximately 5x lower, attributable to lighter framing and the absence of RTPS discovery overhead on the data path.

These numbers reflect best-case configurations. Latency over TCP on a real network will be higher, dominated by OS scheduling and TCP stack behavior rather than Zenoh's own overhead.

## When to Use Zenoh

Use Zenoh when:

- You need publish/subscribe and queryable storage in a single protocol without running separate middleware for each.
- You need zero-copy shared-memory transport for high-throughput, low-latency communication between processes on the same host.
- You need to span edge, fog, and cloud with a unified protocol — Zenoh's router topology handles heterogeneous network segments transparently.
- DDS is too heavy for your embedded or resource-constrained environment; Zenoh's minimal footprint and wire overhead make it viable where DDS is not.
- You need a protocol that handles intermittent connectivity gracefully, because routers can buffer and forward when endpoints reconnect.

Avoid Zenoh when:

- You require full DDS QoS compliance (deadline, liveliness with watchdog timers at the DDS API level, ownership strength, etc.). Zenoh provides liveliness primitives but does not expose the full DDS QoS surface.
- You already operate a stable MQTT broker infrastructure and have no edge-to-cloud query requirements, no SHM use cases, and no plans to extend the system to constrained nodes. In that scenario, migrating to Zenoh adds complexity without clear benefit.

## Key Terminology

| Term              | Meaning                                                                                                                                |
|-------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Session           | The top-level handle to a Zenoh network. All operations are performed through a session.                                               |
| Key Expression    | A string, possibly containing wildcards, that identifies one or more resources in the keyspace.                                        |
| Publisher         | A session entity that sends data samples to a key expression.                                                                          |
| Subscriber        | A session entity that receives data samples matching a key expression.                                                                 |
| Queryable         | A session entity that handles incoming queries on a key expression and sends replies.                                                  |
| Query             | A request issued via `session.get()` to retrieve data from queryables and storage nodes.                                               |
| Sample            | The unit of data delivery: a key expression, a payload, a timestamp, an encoding, and a kind (Put or Delete).                         |
| ZBytes            | The payload container. Holds arbitrary bytes; constructed and consumed via `From`/`Into` conversions.                                  |
| Encoding          | Advisory metadata attached to a sample describing the format of the payload (e.g., `TEXT_PLAIN`, `APPLICATION_JSON`).                 |
| Liveliness Token  | A session-scoped token whose presence signals that a participant is alive; subscribers detect its appearance and disappearance.        |
