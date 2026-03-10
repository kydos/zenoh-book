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

## Zenoh Genesis

Around 2009–2010, while working on extremely large-scale systems spanning military, aerospace, and smart city deployments, it became clear that no existing protocol could cover the full spectrum from microcontroller to datacenter. Each protocol solved part of the problem and failed somewhere else on the stack.

**DDS** provided location transparency for data in motion and a data-centric model, but could not scale down to constrained hardware and lost its location-transparency guarantees the moment data was stored — at which point applications were forced back to centralized cloud storage.

**CoAP** was effective for integrating small devices with web applications but remained inherently client/server and cloud-centric.

**MQTT** introduced a lightweight broker-based pub/sub model suited to cloud-dependent applications. Its broker topology, however, created what could be called the MQTT paradox: two devices on the same local network still route every message through a broker running on a cloud server thousands of kilometres away, adding latency and creating a single point of failure.

The result of combining these protocols to cover a full system was what could be called the **Digital Frankenstein** era: large-scale cloud-to-microcontroller systems duct-taped together from a series of protocol stacks, each covering one network segment, with no unified semantics across the whole.

Zenoh was designed to close that gap. The goal was a single protocol that works efficiently from microcontroller to datacenter, with no topological constraints, and that provides unified abstractions for data in motion (pub/sub), data at rest (distributed queries), and location-transparent computation.

The name itself encodes the design intent. It references Zeno of Elea — the pre-Socratic philosopher known for his paradoxes of infinity — and Zenon of Citium, the founder of Stoicism. It also stands for **ZEro Network OverHead**, reflecting the protocol's efficiency-first design philosophy.

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

## When to Use Zenoh

Use Zenoh when:

- You need publish/subscribe, queryable storage, and request/reply in a single protocol. Zenoh covers all three without requiring separate middleware stacks or translation bridges between them.
- You need to span microcontrollers, embedded gateways, and cloud servers with one protocol and one API. Zenoh-Pico runs on bare-metal targets with tens of kilobytes of RAM; the same wire protocol connects to a datacenter router.
- You need zero-copy shared-memory transport for high-throughput, low-latency communication between processes on the same host — camera pipelines, LiDAR streams, ML inference data.
- You need to integrate other protocols. Zenoh is designed to act as a protocol backbone: connectors exist for DDS, MQTT, ROS 2, and REST, allowing existing systems to participate in a Zenoh keyspace without being rewritten.
- You need to span edge, fog, and cloud across heterogeneous network segments — WiFi, cellular, Ethernet, serial — with automatic topology-aware routing and no broker single point of failure.
- You are building a system where data must be stored, queried, and streamed with consistent semantics. Zenoh's queryable storage model avoids the split between a messaging layer and a separate database layer.
- You need efficient operation under intermittent connectivity. Zenoh routers buffer and forward when endpoints reconnect; storages answer queries from local replicas when upstream is unavailable.

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
