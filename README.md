# Zenoh CAN Link Proposal

This document proposes extending Zenoh to devices on a CAN bus. Doing so
requires designing an addressing scheme and a data framing and segmentation
format.

## Background

Zenoh is a flexible and efficient network protocol supporting pub/sub and query
patterns on a unified key-space. It works across multiple network topologies
including peer-to-peer, client-router, hierarchical, brokered, and mesh, and on
multiple transports including TCP, UDP, QUIC, and serial. It is low overhead and
scales from high performance and high bandwidth systems down to highly
constrained environments such as microcontrollers.

CAN is a very common, low cost, reliable, and robust bus allowing multiple
devices to communicate with each other. It is nearly ubiquitous in the
automotive, aerospace, marine, robotics, and industrial spaces. Nodes may share
a CAN bus with no master due to its frame arbitration scheme, which ensures that
the highest-priority frame always gets through.

### Challenges

CAN poses several challenges for a generic communications protocol:

* Classic CAN only transfers payloads up to 8 bytes, while CAN FD supports up
  to 64, but there is no defined segmentation scheme for larger payloads.
* CAN 11-bit and 29-bit frame IDs are used both for priority and to identify the
  frame, but do not inherently address a source or destination.
* CAN only defines how frames get onto the wire, leaving addressing, data
  format, and node discovery to higher-layer protocols.

## Requirements

The link should:

### Topologies and Roles

* Support Zenoh's peer-to-peer topology: peers on the same bus communicate
  directly, with no router required.
* Support client-router topologies: clients on the bus connect to a router on
  the same bus.
* Allow a router or peer to bridge a CAN bus to other links (Ethernet, serial,
  etc.), so that hierarchical, brokered, and mesh topologies can include CAN
  segments.
* Allow multiple concurrent sessions per node, to any mix of peers, clients, and
  routers on the bus.

### Communication

* Carry the full Zenoh protocol, supporting pub/sub and query patterns without
  CAN-specific changes to the layers above the link.
* Support unicast sessions between any pair of nodes, and multicast for group
  communication such as scouting and discovery.
* Carry Zenoh datagrams larger than a single CAN frame, including Zenoh's
  control messages, up to the link's MTU.
* Preserve Zenoh's priorities, so that higher-priority traffic is favored when
  the bus is contended.

### Addressing and Configuration

* Give every node an address that is unique on its bus.
* Allow nodes to obtain an address automatically, without per-device
  configuration, while still supporting devices that need a fixed address.
* Support nodes joining and leaving the bus at any time without disrupting
  existing sessions.
* Not require a master node for data exchange.

### Robustness

* Tolerate lost or corrupted frames, including one or more missing segments of a
  larger datagram, by detecting the failure and discarding the incomplete
  datagram rather than delivering bad data or stalling the link.

### Constraints

* Suit constrained devices: small code and RAM footprint, and low per-frame
  overhead.
* Work on a standard CAN FD bus shared with other nodes, with node counts in the
  tens of devices.

## Design Summary

* CAN FD only.
* Priority is encoded in the top bits of the CAN identifier, so that CAN
  arbitration orders frames by Zenoh priority.
* Segmentation is implemented at the link layer, because Zenoh's control
  messages do not fit into one frame.
* Node IDs are allocated dynamically by a central allocator.
* Every node has a 7-bit ID (up to 127 devices per bus, which is likely more
  than needed), used for both unicast and multicast.
* Multicast uses 11-bit identifiers. Unicast and node ID allocation use 29-bit
  identifiers.
* Every frame starts with a header byte. The first frame of a datagram also
  carries the datagram length: 1 byte for a single-frame datagram, 2 bytes for
  a multi-frame datagram.
* The header byte carries start-of-transfer and end-of-transfer flags and a
  segment index.
* Multi-frame datagrams carry a 2 byte CRC.
* Node IDs are assigned based on a 16 byte per-device unique ID.
* There is one ID allocator per bus (no redundancy).
* Dynamic and static node IDs come from separate pools: dynamic IDs are 1-63 and
  static IDs are 64-127.

## Addressing

### Multicast Identifier

Multicast frames use an 11-bit identifier to carry priority and sender node ID:

| Bits | Field | Description |
|------|-------|-------------|
| 10:8 | Priority | Zenoh priority |
| 7 | *Reserved* | *Reserved* |
| 6:0 | Source | Sender node ID (1-127) |

With equal priorities, the lower sender node ID wins arbitration. A receiver may
filter frames based on sender node ID.

### Unicast Identifier

Unicast frames use 29-bit identifiers to carry priority, sender and receiver
node IDs, a frame kind, and a direction bit:

| Bits | Field | Description |
|------|-------|-------------|
| 28:26 | Priority | Zenoh priority |
| 25:23 | Kind | `000` for unicast session, `001` for ID allocation |
| 22 | Direction | Set on frames sent by the accepting side |
| 21:14 | *Reserved* | *Reserved* |
| 13:7 | Source | Sender node ID (1-127) |
| 6:0 | Destination | Receiver node ID (1-127) |

As with multicast, the lower sender node ID wins arbitration at equal
priorities. The Direction bit ensures that two sessions with opposite Source and
Destination are handled correctly. There is only one session per pair of nodes
and direction. A receiver may filter frames based on destination node ID.

## Framing and Segmentation

The link reports an MTU of 2048 bytes and segments data at the link layer. Each
Zenoh datagram is carried either in a single frame (up to 62 bytes) or split
across multiple frames (up to 2048 bytes).

Every frame begins with a header byte:

| Bits | Field | Description |
|------|-------|-------------|
| 7 | SOT | Start of transfer, set on a datagram's first frame |
| 6 | EOT | End of transfer, set on a datagram's last frame |
| 5:0 | Index | Frame's position within the datagram, starting at 0 |

### Single-Frame Datagram

* Header byte (SOT = 1, EOT = 1, Index = 0)
* Length (1 byte)
* Payload (0-62 bytes)
* Padding (zero padding up to the next CAN FD frame size)

### Multi-Frame Datagram

A multi-frame datagram is split into a first frame, zero or more middle frames,
and a last frame.

First frame:

* Header byte (SOT = 1, EOT = 0, Index = 0)
* Length (2 bytes)
* CRC (2 bytes)
* Payload (59 bytes)

Middle frames:

* Header byte (SOT = 0, EOT = 0, Index = 1 to N-2)
* Payload (63 bytes)

Last frame:

* Header byte (SOT = 0, EOT = 1, Index = N-1)
* Payload (1 to 63 bytes)
* Padding (zero padding up to the next CAN FD frame size)

## Node ID Allocation

A device joining a bus requests a node ID from the bus's central allocator by
sending a request containing its unique ID and an optional preferred node ID.
The allocator responds with the assigned node ID. The unique ID must be stable
across device power cycles.

The device repeats the request every 600 ms plus a random 0-400 ms until it
receives a response containing its unique ID. It then uses the assigned node ID
until it is reset or power cycled.

The allocator listens to all traffic, never hands out a node ID it has seen in
use, and only hands out IDs from the dynamic pool.

### Request

Identifier:

| Bits | Field | Description |
|------|-------|-------------|
| 28:26 | Priority | 7 |
| 25:23 | Kind | `001` |
| 22 | Direction | 0 (request) |
| 21:0 | Discriminator | Portion of unique ID, or random |

Payload:

| Byte | Field | Description |
|------|-------|-------------|
| 0 | Message | `0x01` (request) |
| 1:16 | Unique ID | Node's unique hardware ID |
| 17:18 | Node ID | Requested node ID, or 0 for none |

### Response

Identifier:

| Bits | Field | Description |
|------|-------|-------------|
| 28:26 | Priority | 7 |
| 25:23 | Kind | `001` |
| 22 | Direction | 1 (response) |
| 21:14 | *Reserved* | *Reserved* |
| 13:7 | Source | Allocator's node ID (1-127) |
| 6:0 | Destination | `0x00` |

Payload:

| Byte | Field | Description |
|------|-------|-------------|
| 0 | Message | `0x02` (response) |
| 1:16 | Unique ID | Echoed unique hardware ID |
| 17:18 | Node ID | Assigned node ID |

## Possible Optimizations

### 11-bit Unicast Identifier for Single-Router Segments

On a client-router segment with a single router, clients only exchange unicast
traffic with that router, so one end of every unicast session is implied. The
29-bit unicast identifier, which carries both a source and a destination, is
more than is needed. Unicast frames on such a segment could instead use an
11-bit identifier carrying only the client's node ID:

| Bits | Field | Description |
|------|-------|-------------|
| 10:8 | Priority | Zenoh priority |
| 7 | Direction | Set on frames sent by the router |
| 6:0 | Client | Client node ID: the sender when Direction is 0, the receiver when Direction is 1 |

Compared to the 29-bit format, this would shorten the identifier on the wire,
and as a standard-length identifier it would be cheaper for devices to filter.

Open questions:

* How are these frames distinguished from multicast frames, which use the same
  identifier length?
* How do nodes learn that the segment has a single router and switch to this
  format, and what happens if a second router appears?
