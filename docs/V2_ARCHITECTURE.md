# BACnet Endpoint / Config-Node Refactor Requirements

## High-level goal

Refactor the current BACnet node architecture so that shared BACnet runtime components are owned by proper
Node-RED config nodes rather than hidden in flow context.

This redesign will support three clean usage patterns built around a local BACnet device:

1. **Remote BACnet interaction**
   The flow initiates BACnet requests to other devices.

2. **Local BACnet device exposure**
   The flow exposes a local BACnet device and optional local object model.

3. **Gateway / proxy composition**
   The flow combines local device behavior and outbound BACnet requests to implement proxy or gateway logic.

The existing `bacnet_gateway` node should be updated to support the new config-node model while preserving
backward compatibility through deprecated fallback to the current flow-context behavior.

---

## New Project Code Layout

| Directory   | Purpose                                                     |
|-------------|-------------------------------------------------------------|
| nodes/      | Node-RED node `.js` / `.html` files, including config nodes |
| lib/        | Runtime classes and shared internal logic                   |
| resources/  | Vendored deps, static assets, bundled support files         |
| icons/      | Node icons                                                  |
| examples/   | Example flows                                               |
| locales/    | i18n (future)                                               |
| test/       | Node-RED node and end-to-end tests                          |

- As legacy nodes are updated, components will be moved into this structure



## Architectural requirements

Backward compatibility should be preserved wherever practical.
This redesign is intended to align the package more closely with both standard Node-RED module patterns
and BACnet's native device/object model.

Existing nodes may be marked as deprecated and may emit warnings encouraging users to migrate to the 
new model, but existing flows should not be forced to migrate immediately.


### Current design limitations
The library currently stores shared runtime objects such as:

- `bacnetClient`
- `bacnetServer`
- `bacnetConfig`


in flow context and retrieves them implicitly from operational nodes.

This creates hidden coupling, ambiguous ownership, and behavior that depends on shared implicit
state rather than explicit node relationships. It also diverges from the standard Node-RED pattern
of using config nodes to own shared runtime components.

As a result, it is difficult to support multiple independent BACnet device runtimes cleanly
within the same flow or workspace, because these runtime objects are effectively treated as shared
singletons. Moving them into proper config nodes would make ownership explicit, better support
concurrency and multiple instances, and allow flow authors to compose gateway and server
architectures in combinations that are currently awkward or impossible.


## Unified BACnet Device Model

### 1. Introduce a unified BACnet device model

The redesign should treat the primary shared runtime object as a local BACnet **device**, not as separate first-class client and server objects.

A BACnet device may:

- originate BACnet service requests to other devices
- respond to requests addressed to itself
- identify itself on the network through standard BACnet device-management services
- optionally expose a local object model, including objects such as Analog Inputs, Analog Values, Binary Inputs, and Binary Values

In this model, "client" and "server" are implementation roles or behaviors of a BACnet device, not the primary architectural units.

Each local BACnet device uses:

- one `bitpool-bacnet-device`

This configuration node represents one local BACnet device and owns:

- transport configuration
- Device Object identity
- optional local object definitions
- shared runtime state needed for both outbound and inbound BACnet behavior

This shared device model also provides the foundation for two distinct categories of operational nodes:

- nodes that manipulate the local device's own object/property model
- nodes that initiate BACnet service requests to remote devices

These node families should remain distinct in naming and behavior, even though both operate against the
same underlying `bitpool-bacnet-device` runtime.

---


### 2. Gateway composition model

In the current implementation, "gateway" is used as the package's central BACnet runtime node. It combines 
a collection of behaviors into a single node, including transport and device settings, discovery, and optional
local BACnet server functionality.

In practice, the term "gateway" is often used loosely to describe several different
kinds of behavior. For clarity, this design document distinguishes between the
following categories.

#### 1. Application-layer proxying or forwarding

A request is received by the local BACnet device, cannot or should not be
satisfied from its own local object model, and is instead fulfilled by obtaining
the data from another device or system. The response is then returned through the
local device, even though the actual data originated elsewhere.

In this model:

- the incoming request is addressed to the local device
- the local device decides whether to answer from its own local object model
- if it does not answer locally, it obtains the data elsewhere and returns the
  response as though it came from the local device

This is the model that best matches a synthetic or virtual BACnet device whose
objects are backed by flow logic, cached values, or other remote systems.

Who-Is / I-Am behavior in this model is straightforward:

- the local device should answer `Who-Is` for its own Device Object
- the local device is not required to forward `Who-Is` or `I-Am` simply because
  it proxies application data from elsewhere

**This is the closest match to what this library does today when it exposes a
local BACnet server and populates local points from Node-RED messages.**

#### 2. Lower-level message forwarding between BACnet transports or networks

A BACnet message is received on one side of the system and forwarded to another
BACnet transport or network with minimal interpretation of its application-layer
meaning.

In this model:

- the forwarding logic is concerned primarily with getting the message to the
  correct destination
- replies must be returned to the original sender
- the forwarding path must preserve enough request/response context to route the
  reply correctly
- the forwarding logic does not primarily operate by recreating object/property
  semantics in flow logic

This kind of forwarding raises additional questions around discovery and device
management traffic, for example:

- whether `Who-Is` messages should be forwarded across the boundary
- whether `I-Am` messages should be repeated back across the boundary
- whether request/response correlation must be tracked explicitly in order to
  return replies to the correct originator

This is a different problem from application-layer proxying and should not be
assumed to be the same thing.

#### 3. General bridging or routing behavior

BACnet traffic is passed between different parts of a system so that devices on
one side can communicate with devices on the other.

In this model:

- the local system acts as a communication path between BACnet participants
- traffic may cross transport or network boundaries
- some form of routing or path knowledge may be required so that requests and
  replies can move between the correct BACnet networks

This raises broader questions than simple application proxying, including:

- whether discovery traffic such as `Who-Is` should cross the boundary
- whether `I-Am` responses should be observed, repeated, or cached
- whether the local system must learn or maintain knowledge about which BACnet
  devices are reachable on which side of the boundary
- whether the local system is acting more like BACnet network infrastructure
  than like an ordinary BACnet device

This category is related to, but not identical with, lower-level message
forwarding. It may require separate design treatment.

#### Relevance to this Version 2 redesign

The primary Version 2 design described in this document is based on
**application-layer proxying / forwarding**, using a local BACnet device and
supporting operational nodes.

Lower-level forwarding and routing / bridging behavior may be desirable in the
future, but they represent different architectural problems and should not be
collapsed into the same implementation model by accident.

In the Version 2 model, gateway behavior is treated as a **composite use case**, not as the primitive runtime type.

There is no single monolithic node that defines all gateway behavior. Instead, gateway behavior is constructed
from one or more BACnet device nodes and supporting operational nodes, combined at the flow designer's discretion.

This allows the architecture to support multiple kinds of gateway behavior without forcing all of them into one
node or one implementation model.

### 3. Client/Server module decomposition

In the Version 2 model, client and server are not treated as separate first-class
runtime objects. Instead, they are understood as behaviors of a BACnet device,
and the user interacts with those behaviors through focused operational nodes.

This replaces the current monolithic approach with a smaller set of nodes whose
responsibilities are explicit and easier to combine.

#### Local device/object access

The new model should provide individual nodes for interacting with the local
BACnet device's own object/property memory.

These nodes are used when the flow needs to:

- read values from the local device's object model
- update values in the local device's object model
- manipulate local BACnet-visible state without initiating a remote BACnet
  request

These nodes operate directly against the local device configured by
`bitpool-bacnet-device`. They are intended for local object/property access only.

#### Remote BACnet service access

The new model should also provide individual nodes for initiating BACnet service
requests to remote devices.

These nodes are used when the flow needs to:

- read properties from remote BACnet devices
- write properties to remote BACnet devices
- initiate outbound BACnet interactions that are not satisfied from the local
  device's own object model

These nodes should handle the full remote operation path needed for their
purpose, rather than depending on a separate gateway node to wrap that behavior.

#### Discovery ownership

Discovery remains an important shared concern, but it should not require a
separate monolithic gateway node.

In the Version 2 model, discovery behavior and any associated discovery cache
should be owned by the `bitpool-bacnet-device` configuration node as part of its
runtime implementation.

This means:

- operational remote read/write nodes may rely on the device's shared discovery
  and discovery-cache state
- discovery logic is implemented once at the device level rather than recreated
  independently by each operational node
- users interact with focused read/write nodes, while shared discovery behavior
  remains attached to the device that participates on the BACnet network

This decomposition keeps the protocol participation centered in the BACnet
device while exposing smaller, clearer operational nodes for both local and
remote behavior.
---
