# NetKVM driver compatibility

NetKVM supports every released Windows 10 and Windows 11 version, including
releases no longer serviced by Microsoft.
A build label such as `Win10 Release` describes the binary's
compilation settings; it is not the minimum Windows release on which
that binary can run. This page separates compile-time NDIS support, the NDIS
version found on the host OS, the version reported by the driver, and the
conditions that enable individual features.

## Four different version decisions

| Decision | Where it happens | What it controls |
| --- | --- | --- |
| Build target | The `Win10 Release` project defines `NDIS685_MINIPORT=1`; `Win11 Release` defines `NDIS686_MINIPORT=1`. Both define the same `NTDDI_VERSION`. | NDIS declarations and `#if` branches available to each binary. See [NetKVM-VS2015.vcxproj](../NetKVM-VS2015.vcxproj). |
| OS NDIS version | `SetRuntimeNdisVersion` calls `NdisGetVersion` and stores the actual major/minor version as `osmajor`/`osminor`. | `CheckOSNdisVersion` tests what the running system provides. See `SetRuntimeNdisVersion` in [ParaNdis6_Driver.cpp](../wlh/ParaNdis6_Driver.cpp) and `CheckOSNdisVersion` in [ndis56common.h](../Common/ndis56common.h). |
| Effective miniport version | `SetRuntimeNdisVersion` selects a version for the miniport, which `DriverEntry` puts in `NDIS_MINIPORT_DRIVER_CHARACTERISTICS`. | `CheckNdisVersion` and NDIS capability/structure-revision decisions. The effective version may be lower than the OS version. See `SetRuntimeNdisVersion` and `DriverEntry` in [ParaNdis6_Driver.cpp](../wlh/ParaNdis6_Driver.cpp), and `CheckNdisVersion` in [ndis56common.h](../Common/ndis56common.h). |
| Feature activation | Adapter configuration, NDIS version checks, initialization results, and sometimes VirtIO feature negotiation. | Whether a compiled feature actually runs. See [initialization and VirtIO feature negotiation](INITIALIZATION.md). |

For example, a binary compiled with NDIS 6.85 headers can still select an
older effective NDIS interface when it loads on an older Windows 10 release.
The code groups older OS versions at supported thresholds (6.83, 6.80, or
6.30), rather than always reporting the exact OS minor version. This is how
the project can retain one Win10 build while supporting older Windows 10
releases; the build target must not be read as a minimum OS requirement.

## Which compile-time gates still distinguish the builds?

In the current NetKVM source, `NDIS_SUPPORT_NDIS6`, `NDIS_SUPPORT_NDIS61`,
`NDIS_SUPPORT_NDIS620`, `NDIS_SUPPORT_NDIS630`, `NDIS_SUPPORT_NDIS650`,
`NDIS_SUPPORT_NDIS660`, `NDIS_SUPPORT_NDIS680`, `NDIS_SUPPORT_NDIS683`,
`NDIS_SUPPORT_NDIS684`, and `NDIS_SUPPORT_NDIS685` all select the same branch
in both builds. Their alternatives for older NDIS build targets are not part
of either current binary. This observation concerns preprocessing only: it
does not make the runtime `CheckNdisVersion` tests obsolete.

`NDIS_SUPPORT_NDIS686` is the meaningful difference. It defines
`PARANDIS_SUPPORT_POLL` only in the Win11 build. The Win10 build compiles poll
stubs; the Win11 build compiles `NdisRegisterPoll`, `NdisRequestPoll`, and the
poll RX/TX path. See `PARANDIS_SUPPORT_POLL` in [ndis56common.h](../Common/ndis56common.h),
`NdisPollHandler::Register` and `NdisPollHandler::HandlePoll` in
[ParaNdis_Poll.cpp](../wlh/ParaNdis_Poll.cpp), and `CParaNdisTemplatePath::FireDPC` in
[ParaNdis-AbstractPath.h](../Common/ParaNdis-AbstractPath.h).
`NDIS_MINIPORT_MINOR_VERSION` is also a compile-time constant (85 or 86) used
by `SetRuntimeNdisVersion`; it is distinct from an `#if` feature gate.

The Win10 build keeps Poll Mode API calls out of the binary, including when
that binary runs on older Windows 10 releases. The older OS versions are
handled by runtime version selection, not by compiling another Win10 binary.
This page records the existing compile-time branches; it does not propose a
change to the compatibility policy or to the source code.

## NDIS releases relevant to this code

| Release | Microsoft's documented OS association | NetKVM consequence |
| --- | --- | --- |
| 6.85 | Windows 10 21H2 | Win10 compile target; the driver still supports older Windows 10 releases. |
| 6.86 | Windows 11 21H2 | Win11 compile target; poll code is present. |
| 6.87 | Windows 11 22H2 | Win11 build normally reports its compiled 6.86 level. |
| 6.88 | Windows Server version 23H2 | No separate NetKVM compile target. Microsoft's NDIS table does not list this as a Windows 11 release. |
| 6.89 | Windows 11 24H2 | Special version-reporting path in the current Win11 build; current OS-version threshold for attempting Poll Mode. |

See Microsoft's [NDIS version overview](https://learn.microsoft.com/en-us/windows-hardware/drivers/network/overview-of-ndis-versions).
This table shows representative version associations, not the full NetKVM OS
support matrix or a statement about where Poll Mode is enabled.

## Poll Mode: Windows 11 only in NetKVM

NetKVM compiles Poll Mode API calls only into the Win11 binary. Microsoft's
[`NdisRegisterPoll` API requirements](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/poll/nf-poll-ndisregisterpoll)
list **Windows 11** as the minimum supported client. This guide treats Poll
Mode as unavailable on Windows 10. The Win10 binary therefore contains only
poll stubs and does not depend on these newer API calls; its compatibility
with older Windows 10 releases does not require a separate build for each OS
release.

The Win11 binary does not try Poll Mode on every Windows 11 release.
`ReadNicConfiguration` sets `bPollModeTry` only if `*NdisPoll` is enabled and
`CheckOSNdisVersion(6, 89)` succeeds. VirtIO `VIRTIO_NET_F_STANDBY` suppresses
the attempt. `ParaNdis_FinishInitialization` also requires suitable RSS
queues and successful registration of every poll handler before it sets
`bPollModeEnabled`. There is no separate QEMU/VirtIO Poll Mode feature bit.
See `ReadNicConfiguration`, `ParaNdis_InitializeContext`, and
`ParaNdis_FinishInitialization` in [ParaNdis_Common.cpp](../Common/ParaNdis_Common.cpp),
and `NdisPollHandler::Register` in [ParaNdis_Poll.cpp](../wlh/ParaNdis_Poll.cpp).

*Note: The `CheckOSNdisVersion(6, 89)` test is a NetKVM implementation
restriction. It is stricter than the minimum stated for the Poll Mode API on
Windows 11. Compiling poll code, reporting a miniport version, and actually
enabling Poll Mode are three different events.*

## NDIS 6.89 reporting with a 6.86 build

The Win11 driver is currently built with Visual Studio 2019 and WDK headers
for NDIS 6.86. Poll Mode support was added before moving the project to a WDK
with NDIS 6.89 definitions. A full toolchain migration also involves separate
Visual Studio and WDK build requirements, so the project uses a temporary
version-reporting path for its Windows 11 HCK 2025 target.

In `SetRuntimeNdisVersion`, when the binary is compiled as NDIS 6.86 and the
OS reports NDIS 6.89 or newer, the driver selects an effective miniport
version of 6.89. This allows the current toolchain to produce the Win11
driver for that HCK target. See `SetRuntimeNdisVersion` in
[ParaNdis6_Driver.cpp](../wlh/ParaNdis6_Driver.cpp) and the project settings
in [NetKVM-VS2015.vcxproj](../NetKVM-VS2015.vcxproj).

The current NetKVM source has no separate NDIS 6.89 offload implementation.
Reporting 6.89 does not imply that every optional NDIS 6.89 feature is present.
When the project moves fully to Visual Studio 2022 and a WDK with NDIS 6.89
definitions, the temporary versioning code and compile-time definitions can be
updated.

## UDP offloads: USO versus URO

- **UDP Segmentation Offload (USO, TX):** NetKVM supports IPv4 and IPv6 USO in
  both the Win10 and Win11 builds. Windows documents USO support from
  [Windows 10 version 2004 onward](https://learn.microsoft.com/en-us/windows-hardware/drivers/network/udp-segmentation-offload-uso-).
  The code compiles this path under `PARANDIS_SUPPORT_USO` and requires an
  effective miniport version of at least NDIS 6.83 before reporting it; the
  NDIS threshold and Microsoft's OS support statement are distinct. The
  corresponding `*UsoIPv4` or `*UsoIPv6` setting must be enabled, the host
  must offer the accepted `VIRTIO_NET_F_HOST_USO` feature, and accepted
  `VIRTIO_NET_F_STANDBY` disables USO in the failover configuration. See
  `PARANDIS_SUPPORT_USO` in [ndis56common.h](../Common/ndis56common.h),
  `ParaNdis_InitializeContext` in [ParaNdis_Common.cpp](../Common/ParaNdis_Common.cpp),
  and `FillOffloadStructure` in [ParaNdis6_Oid.cpp](../wlh/ParaNdis6_Oid.cpp).
  On transmit, `CNBL::ParseUSO` reads the NDIS packet metadata and
  `CNB::SetupUSO` prepares a VirtIO UDP segmentation request; see
  [ParaNdis_TX.cpp](../Common/ParaNdis_TX.cpp).
- **UDP Receive Segment Coalescing Offload (URO, RX):** NetKVM does not
  implement or advertise URO. Windows introduced this optional feature in
  [Windows 11 version 24H2 / NDIS 6.89](https://learn.microsoft.com/en-us/windows-hardware/drivers/network/udp-rsc-offload).
  Reporting NDIS 6.89 does not turn it on. NetKVM's existing RSC path is for
  TCP receive traffic, not UDP.

UDP checksum offload is separate from both features: NetKVM reports IPv4/IPv6
UDP checksum capabilities for TX and RX according to configuration and
negotiated device capabilities. See `FillOffloadStructure` in
[ParaNdis6_Oid.cpp](../wlh/ParaNdis6_Oid.cpp).

## Related guides

- [Initialization and VirtIO feature negotiation](INITIALIZATION.md)
- [Architecture and driver workflow](ARCHITECTURE.md)
- [NetKVM device power management](POWER_MANAGEMENT.md)
