# Version 2 Node Specifications

## `bitpool-bacnet-device`

### Purpose

The `bitpool-bacnet-device` config node should present the local BACnet device
in terms that make sense to a BACnet user.

It acts as both:

- the definition of the local BACnet participant on the network
- the design-time definition of any local BACnet objects that participant
  exposes

### Editor layout

#### Endpoint configuration
- transport type (for example UDP/BACnet-IP, serial/MS-TP later)
- transport binding details
- CIDR or similar network filtering if supported

#### Local device information
- device instance
- device name
- network number as applicable
- whether local server/object exposure is enabled
- other local device identity fields as needed

#### Local object configuration
- tabs or equivalent sections for object families such as:
    - Analog Inputs
    - Analog Values
    - Binary Inputs
    - Binary Values
    - Multi-state objects later
    - device-level properties

## `bitpool-bacnet-local-put`

TBD

## `bitpool-bacnet-local-get`

TBD

## `bitpool-bacnet-read-property`

For remote reads, TBD

## `bitpool-bacnet-write-property`

For remote writes, TBD

## Future nodes

### `bitpool-bacnet-subscribe-cov`

Initiates a BACnet Change of Value (COV) subscription against a remote BACnet
device using a `bitpool-bacnet-device`.

This node is intended for use when the flow wants to receive value-change
updates from a remote device without polling the property continuously.

### `bitpool-bacnet-notification-in`

Receives inbound BACnet notifications associated with subscriptions, events, or
other BACnet notification services and emits them into the flow.

This node is intended to give the flow a clear entry point for handling
asynchronous BACnet notifications separately from ordinary read/write request
behavior.
