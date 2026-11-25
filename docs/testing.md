# Cluster validation

Validation started on 2026-11-25 on the supplied disposable OpenShift Virtualization cluster:
OpenShift Virtualization 5.0.1, KubeVirt 1.9.0, Kubernetes 1.36.5, and the default `managed-csi`
Azure block storage class. All test resources use namespace `kubevirt-examples-test-20261125`.

**All four examples passed the installer and clone checks below.** A running VMI alone
is not considered a passing test.

| Example | Observed result |
| --- | --- |
| Server 2025 installer | Passed native generalization and shutdown, creation-time answer, 80 GiB GPT clone expansion, DHCP, guest-agent checks, and device checks. |
| Windows 11 installer | Passed offline WinPE installation, native generalization and shutdown, creation-time answer, 80 GiB GPT clone expansion, DHCP, guest-agent checks, and device checks. |
| Server 2008 SP2 x64 | Passed fresh installation with the cluster preference and modern VirtIO, rename-only Sysprep, creation-time answer, 80 GiB expansion, DHCP, agent 7.6.2, PowerShell 2.0, and KMDF 1.9. |
| Server 2008 SP2 x86 | Passed fresh installation with the cluster preference and modern VirtIO, rename-only Sysprep, creation-time answer, 80 GiB expansion, DHCP, agent 7.6.2, PowerShell 2.0, and KMDF 1.9. |

All four installer manifests pass YAML/XML parsing and Kubernetes server-side dry-run validation.
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
- Use `windows.2k8.virtio` for both legacy architectures. Omit `useVirtioTransitional`: the
  pinned drivers support modern PCI IDs, and both clones boot with non-transitional devices.
  Set `autoattachInputDevice: false` because the preference's VirtIO tablet and KubeVirt's
  alternative USB/xHCI tablet have no compatible driver. PS/2 input remains available.
- Use the verified legacy container described in [legacy-windows.md](legacy-windows.md).
  Its 66 relevant driver files match the reference 1.9.12 ISO byte for byte, and its agent
  is the separately compatible version 7.6.2.
- Check MSI results and use Windows Setup's
  [`WillReboot=OnRequest`](https://learn.microsoft.com/en-us/windows-hardware/customize/desktop/unattend/microsoft-windows-deployment-runsynchronous-runsynchronouscommand-willreboot)
  return-code semantics:
  0 continues, 1 reboots before the next command, and 2 reboots and repeats the command.
  The scripts return 3 on failure so Setup stops for inspection instead of generalizing.
- Rename the cached `Panther\unattend.xml` to `unattend.install.xml` before Sysprep. Supply a separate `unattend.xml` when creating
  each clone, as described in [cloning.md](cloning.md).
- Use Windows Setup's native `Generalize` block in both modern examples. The driver step
  handles required reboots before a short command renames the cached installation answer.
- Install only VirtIO drivers and the QEMU guest agent in the modern examples. Windows 11
  loads only `viostor` in WinPE; `ipconfig /all` confirmed no network adapters during
  installation. Its network driver is installed later in audit mode.
- Give the Server 2025 ISO import 10 GiB to accommodate the ISO and filesystem overhead.

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
and modern VirtIO; their script CDs were refreshed for the rename-only step. Their final
clones used `autoattachInputDevice: false`, and all tested VirtIO devices worked.

Temporary verification VMs, their owned disks, answer Secrets, and inspection pods were
removed. Four stopped installer VMs retain the verified golden disks, exposed through their
`*-golden-image` DataSources in the test namespace.

For a server-side validation in your intended namespace:

```sh
oc apply --dry-run=server -n default -f vms/
```

Follow each installer manifest's header to apply it. Supply per-VM credentials through the
creation-time answer file. Check VM shutdown and guest state before consuming a golden-image
DataSource; DataSource readiness does not indicate that Windows installation has finished.

## Answer files in the golden disks

Read-only inspection confirmed that the legacy images retain `Panther\unattend.install.xml`
and have no `Panther\unattend.xml`. Native generalization in Windows 11 and Server 2025
recreates `Panther\unattend.xml`, marking its specialize, OOBE, and audit passes with
`wasPassProcessed="true"`. The installation backup also remains. This processed cache
contains no clone-specific computer name, password, or other personalization.

Both modern clones consumed a new Secret-backed `unattend.xml`: they applied the supplied
computer names, completed unattended OOBE, and reached `IMAGE_STATE_COMPLETE`.
The native flow therefore meets the creation-time customization requirement; it does not
promise that no file named `unattend.xml` exists anywhere in the golden image.

## Legacy limitations

Server 2008 SP2 reports `ACPI\QEMU0002` (QEMU firmware configuration) as an unknown
device. The tested storage, network, serial, balloon, and PS/2 input devices work.
PowerShell 2.0 works interactively, but launching it through this old QEMU agent returned
no output; native `cmd.exe`/WMIC and the console were used for legacy verification.
