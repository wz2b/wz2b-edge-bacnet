## Device / endpoint model

The redesign should treat BACnet participation in terms of a local
**device / endpoint**, not separate first-class client and server
objects.

A BACnet device may:

- originate requests to other BACnet devices
- respond to requests addressed to itself
- identify itself on the network
- optionally expose a local object registry (for example AVs, AIs,
  BVs, etc.)

In this model, "client" and "server" are behaviors of a BACnet device,
not the primary architectural units.

### Required runtime ownership

Each local BACnet endpoint uses:

- one `bitpool-bacnet-endpoint`

This endpoint represents one local BACnet participant and owns:

- transport binding
- local BACnet device identity
- optional local BACnet object registry/cache
- shared runtime state needed for both outbound and inbound BACnet
  behavior

This allows the same endpoint to be used for:

- pure outbound BACnet interaction
- pure local device behavior
- composite gateway / bridge workflows built from multiple nodes

### Local object model

The `bitpool-bacnet-endpoint` config node is not just transport or
connection settings. It should also persist the local BACnet object
model exposed by that endpoint when local server behavior is enabled.

This includes object families such as:

- Analog Inputs
- Analog Values
- Binary Inputs
- Binary Values
- Multi-state objects later
- device-level properties

An endpoint is still a BACnet device even if it does not expose a rich
local object registry.

For example, an endpoint may have:

- transport settings
- a Device Object identity
- a device instance
- the ability to originate requests
- the ability to answer identification/discovery requests

without necessarily exposing local AV / AI / BV objects of its own.

### Persistence model

These local object definitions should be stored directly in the config
node definition so they are saved with the Node-RED flow.

Do **not** require a separate file for normal operation unless there is
a compelling existing mechanism that must be preserved.

The local object model should be stored sparsely.

Do **not** preallocate large arrays of object slots.

Only store the objects that actually exist.

Example conceptual structure:

```json
{
  "analogValues": [
    { "instance": 1, "name": "RoomTempSP", "presentValue": 72.0, "units": 64 }
  ],
  "binaryValues": [
    { "instance": 1, "name": "Occupied", "presentValue": true }
  ]
}