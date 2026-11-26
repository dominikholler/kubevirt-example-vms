# Cluster validation

Validation started on 2026-11-25 on the supplied disposable OpenShift Virtualization cluster:
OpenShift Virtualization 5.0.1, KubeVirt 1.9.0, Kubernetes 1.36.5, and the default `managed-csi`
Azure block storage class. All test resources use namespace `kubevirt-examples-test-20261125`.
The Server 2008 R2, Server 2012, and Server 2012 R2 examples were added and validated on
2026-11-26 on another cluster built from the same versions and storage class, using
namespace `default`.

**All seven examples passed the installer and clone checks below.** A running VMI alone
is not considered a passing test.

| Example | Observed result |
| --- | --- |
| Server 2025 installer | Passed native generalization and shutdown, creation-time answer, 80 GiB GPT clone expansion, DHCP, guest-agent checks, and device checks. |
| Windows 11 installer | Passed offline WinPE installation, native generalization and shutdown, creation-time answer, 80 GiB GPT clone expansion, DHCP, guest-agent checks, and device checks. |
| Server 2008 SP2 x64 | Passed fresh installation with the cluster preference and modern VirtIO, rename-only Sysprep, creation-time answer, 80 GiB expansion, DHCP, agent 7.6.2, PowerShell 2.0, and KMDF 1.9. |
| Server 2008 SP2 x86 | Passed fresh installation with the cluster preference and modern VirtIO, rename-only Sysprep, creation-time answer, 80 GiB expansion, DHCP, agent 7.6.2, PowerShell 2.0, and KMDF 1.9. |
| Server 2008 R2 x64 | Passed fresh installation with the cluster preference and modern VirtIO, audit-mode driver install (8 installed, `viostor` already bound, `NetKVM` staged by Setup, 0 skipped/failed), native generalization, creation-time answer, 80 GiB expansion, DHCP, agent 7.6.2, and the VirtIO tablet. |
| Server 2012 x64 | Passed fresh installation of Standard (Full) with the cluster preference and modern VirtIO, audit-mode driver install on the modern media (10 installed, `NetKVM` and `viostor` already bound, 0 skipped/failed), native generalization, creation-time answer, 80 GiB expansion, DHCP, and agent 110.2.3. |
| Server 2012 R2 x64 | Passed fresh installation of Standard (Full) with the cluster preference and modern VirtIO, audit-mode driver install on the modern media (11 installed, `NetKVM` and `viostor` already bound, 0 skipped/failed), native generalization, creation-time answer, 80 GiB expansion, DHCP, and agent 110.2.3. |

All seven installer manifests pass YAML/XML parsing and Kubernetes server-side dry-run validation.
All Windows ISO imports succeeded. Modern driver images and the pinned legacy driver image
pulled successfully. VSOCK is absent; no cluster-wide VSOCK feature gate was enabled.

## Findings incorporated into the examples

- Locate post-install media dynamically: Server 2008 assigns a drive letter to System
  Reserved after reboot, shifting the installation CDs.
- Install the remaining legacy drivers online with `pnputil`, including their KMDF
  co-installers, following `virtio-compatibility`. Reboot before Sysprep.
- Use one active MBR Windows partition for legacy guests to keep `C:` stable through
  Sysprep. Propagate script exit codes through the media-discovery loop so Windows Setup
  actually reboots before generalization and completes the KMDF update.
- Use `windows.2k8.virtio` for all three legacy examples. Omit `useVirtioTransitional`: the
  pinned drivers support modern PCI IDs, and the clones boot with non-transitional devices.
  Set `autoattachInputDevice: false` for Server 2008 SP2 because the preference's VirtIO
  tablet and KubeVirt's alternative USB/xHCI tablet have no compatible driver there;
  Server 2008 R2 keeps the tablet because the pinned media ships its `vioinput` driver.
- Use the verified legacy container described in [legacy-windows.md](legacy-windows.md).
  Its 66 `2k8` driver files and 40 `2k8R2` driver files match the reference 1.9.12 ISO by
  name, size, and CRC-32, and its agent is the separately compatible version 7.6.2.
- Check MSI results and use Windows Setup's
  [`WillReboot=OnRequest`](https://learn.microsoft.com/en-us/windows-hardware/customize/desktop/unattend/microsoft-windows-deployment-runsynchronous-runsynchronouscommand-willreboot)
  return-code semantics:
  0 continues, 1 reboots before the next command, and 2 reboots and repeats the command.
  The scripts return 3 on failure so Setup stops for inspection instead of generalizing.
- Rename the cached `Panther\unattend.xml` to `unattend.install.xml` before Sysprep. Supply a separate `unattend.xml` when creating
  each clone, as described in [cloning.md](cloning.md).
- Use Windows Setup's native `Generalize` block for Server 2008 R2 and later, including the
  modern examples. The driver step handles required reboots before a short command renames
  the cached installation answer. The Server 2008 SP2 examples keep the explicit Sysprep
  script that this project verified.
- Install only VirtIO drivers and the QEMU guest agent in the modern examples. Windows 11
  loads only `viostor` in WinPE; `ipconfig /all` confirmed no network adapters during
  installation. Its network driver is installed later in audit mode.
- Give the Server 2025 ISO import 10 GiB to accommodate the ISO and filesystem overhead.
- Select the full-GUI image by name on the Server 2012 evaluation media: `/IMAGE/INDEX` 1
  installs Server Core. `Windows Server 2012 SERVERSTANDARD` and
  `Windows Server 2012 R2 SERVERSTANDARD` select Standard (Full).
- Run the client-server driver step with the modern virtio-win container but the legacy
  `pnputil -i -a` syntax on the 2012 pair. Those OSes verify the media's SHA-256 catalogs
  natively, and the modern `/add-driver /install` syntax is unavailable before Windows 10
  1607.
- Use the answer file's native `Generalize` block for the 2012 pair, as for Windows 11 and
  Server 2025. No Sysprep script or second reboot is needed beyond the driver reboot.
- Handle catalogs that the guest cannot verify when Setup already staged the package: in
  the pinned media, `NetKVM`'s catalog is SHA-256 signed through a root that a fresh
  Server 2008 R2 lacks. Windows Setup injects the same package during offline servicing,
  so the Server 2008 R2 script counts a package already present in the driver store as
  installed. Treating that skip as a failure aborted Setup with "could not parse or
  process unattend answer file ... for pass [auditUser]".
- Skip the Windows Management Framework download on Server 2008 R2 (inbox PowerShell 2.0)
  and keep its VirtIO tablet enabled, because the pinned media ships a Server 2008 R2
  `vioinput` driver. `/IMAGE/NAME` selects `Windows Server 2008 R2 SERVERSTANDARD` on the
  multi-edition media.

## How the tests were run

The manifests were rendered into the isolated namespace, including the shared image imports
and golden-image DataSources. Creation-time answer files supply the test computer names and passwords.
Guest setup was inspected through VNC and the QEMU guest agent. Where the DVD requested a
key, a key was sent once; this manual boot step is documented in the manifests.

Installer success requires automatic shutdown after Sysprep. Verification VMs clone only
the resulting root PVC into an 80 GiB volume, with matching firmware and disk bus,
and attach a fresh Sysprep volume containing a customized `unattend.xml`.
Checks cover unattended first boot, Windows partition expansion, agent connectivity, network
configuration, and guest-agent operation. Modern clones use agent 110.2.3; legacy clones
use 7.6.2. All clones reached `IMAGE_STATE_COMPLETE`.

The modern examples were installed from fresh disks with the final native-generalization
flow. The legacy runs also used fresh disks with the pinned container, cluster preference,
and modern VirtIO; the Server 2008 SP2 script CDs were refreshed for the rename-only step.
Server 2008 R2 and the later examples use native generalization, which needs no such step.
Their final
clones used `autoattachInputDevice: false`, and all tested VirtIO devices worked.

For Server 2008 R2, one key press was sent at the DVD prompt through the VNC console and
progress was inspected with `virtctl vnc screenshot`; the rest of Setup ran unattended.
The stopped golden disk was inspected offline with libguestfs: the driver log reported the
summary above, the guest-agent MSI returned 0, the cached `Panther\unattend.xml` was renamed
to `unattend.install.xml` and native generalization recreated a processed `Panther\unattend.xml`
marked `wasPassProcessed="true"`, and the `NetKVM` package was present in the driver store.
Its verification clone used the golden-image DataSource with an 80 GiB volume and a Sysprep
Secret. It booted unattended to the desktop as Windows Server 2008 R2 Standard; Initial
Configuration Tasks showed `win2k8r2-clone` and a DHCP-assigned IPv4 address, the single
NTFS partition had expanded to 80G (`virt-filesystems` on the stopped clone disk), and guest
agent 7.6.2 connected.

The 2012 pair was installed in parallel with the same procedure. The stopped golden disks
reported the driver summaries above, agent 110.2.3 installed successfully, and the registry
confirmed `Windows Server 2012 Standard Evaluation` / `Windows Server 2012 R2 Standard
Evaluation` with the full shell (`explorer.exe` present). Native generalization left
`Panther\unattend.install.xml` and a processed `Panther\unattend.xml` marked
`wasPassProcessed="true"`. Their verification clones, also run in parallel, booted to the
Server Manager desktop, applied the supplied computer names, expanded the single NTFS
partition to 80G, obtained 10.0.2.2/24 from the masquerade network, and connected agent
110.2.3.

Temporary verification VMs, their owned disks, answer Secrets, and inspection pods were
removed. The stopped installer VMs retain the verified golden disks, exposed through their
`*-golden-image` DataSources.

For a server-side validation in your intended namespace:

```sh
oc apply --dry-run=server -n default -f vms/
```

Follow each installer manifest's header to apply it. Supply per-VM credentials through the
creation-time answer file. Check VM shutdown and guest state before consuming a golden-image
DataSource; DataSource readiness does not indicate that Windows installation has finished.

## Answer files in the golden disks

Read-only inspection confirmed that the Server 2008 SP2 images retain
`Panther\unattend.install.xml` and have no `Panther\unattend.xml`. Native generalization in
Server 2008 R2, Windows 11, Server 2025, Server 2012, and Server 2012 R2 recreates
`Panther\unattend.xml`, marking its specialize, OOBE, and audit passes with
`wasPassProcessed="true"`. The installation backup also remains. This processed cache
contains no clone-specific computer name, password, or other personalization.

The Server 2008 R2, Windows 11, Server 2025, Server 2012, and Server 2012 R2 clones
consumed a new Secret-backed `unattend.xml`: they applied the supplied computer names and
completed unattended OOBE.
The native flow therefore meets the creation-time customization requirement; it does not
promise that no file named `unattend.xml` exists anywhere in the golden image.

## Legacy limitations

Server 2008 SP2 reports `ACPI\QEMU0002` (QEMU firmware configuration) as an unknown
device. The tested storage, network, serial, balloon, and PS/2 input devices work.
PowerShell 2.0 works interactively, but launching it through this old QEMU agent returned
no output; native `cmd.exe`/WMIC and the console were used for legacy verification.

Server 2008 R2's agent 7.6.2 connects and reports its version and network interfaces, but
`virtctl guestosinfo` returns empty OS fields and `virtctl fslist` reports zero byte counts.
The computer name and the expanded partition were therefore verified offline from the
stopped clone disk (registry hive and `virt-filesystems`).
