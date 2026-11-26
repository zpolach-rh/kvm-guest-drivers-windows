# NetKVM programmer documentation

This index collects programmer documentation for the NetKVM Windows
virtio-net miniport. The guides are ordered from the overall design to
specific runtime behavior.

## Start here

1. [Architecture](ARCHITECTURE.md) — functional blocks, RX/TX direction,
   VirtIO boundary, and the adapter lifecycle. Start here when learning how
   the driver is organized.
2. [Initialization and VirtIO feature negotiation](INITIALIZATION.md) —
   `DriverEntry`, adapter creation, PCI resources, feature selection, queue
   setup, and capability reporting. Use this when following the startup path
   through the code.
3. [NetKVM device power management](POWER_MANAGEMENT.md) — Windows D0–D3 states,
   NetKVM power transitions, `Suspended` versus `FastSuspend`, and the limits
   of Modern Standby support.
4. [Driver compatibility](DRIVER_COMPATIBILITY.md) — Win10/Win11 build
   targets, compile-time NDIS gates, runtime version selection, Poll Mode
   availability, and the HCK/toolchain compatibility approach.

## Diagnostics and package reference

- [Tracing](Tracing.md) — how to collect WPP and kdprint traces while
  investigating the driver.
- [NetKVM package Readme](Readme.md) — configuration-tool usage and SR-IOV
  failover behavior. This file is included in release packages and retains
  its release-facing format.
