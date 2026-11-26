# Windows Server 2012 and Windows Server 2012 R2

Both installer examples use the `windows.2k12.virtio` cluster preference, whose display name
covers Server 2012 and Server 2012 R2. They install Standard (Full), x64, from Microsoft's
public evaluation media, use one active MBR Windows partition, and boot with the preference's
modern VirtIO disk, network, serial, balloon, and tablet devices.

The evaluation media list the Server Core images first, so `/IMAGE/INDEX` 1 installs Server
Core. The examples therefore select the full-GUI image by name: `Windows Server 2012
SERVERSTANDARD` and `Windows Server 2012 R2 SERVERSTANDARD`.

## Installation media

| OS | Media | Size in bytes |
| --- | --- | --- |
| Server 2012 | `9200.16384.WIN8_RTM.120725-1247_X64FRE_SERVER_EVAL_EN-US-HRM_SSS_X64FREE_EN-US_DV5.ISO` | 3,694,962,688 |
| Server 2012 R2 | `9600.17050.WINBLUE_REFRESH.140317-1640_X64FRE_SERVER_EVAL_EN-US-IR3_SSS_X64FREE_EN-US_DV9.ISO` | 4,542,291,968 |

The manifests import the ISOs from public archive.org URLs. Evaluation media needs no product
key. Both are BIOS media: they boot-loop under UEFI/OVMF on q35, so the examples rely on
KubeVirt's default BIOS firmware instead of adding a `firmware` block.

## Driver media

Both examples use the same container disk as the Windows 11 and Server 2025 examples:

```text
registry.redhat.io/container-native-virtualization/virtio-win-rhel9:v4.22
```

It ships complete `2k12` and `2k12R2` driver trees. Server 2012 (NT 6.2) and Server 2012 R2
(NT 6.3) verify the media's SHA-256 catalogs natively, so no servicing updates are needed.
Microsoft never published KB4474419 and KB4490628 for these OSes; the SHA-2 updates that
Windows 7 and Server 2008 R2 require do not apply here.

The guest agent is installed from the same container's `guest-agent` MSI. The tested
container ships agent 110.2.3, which connects on both OSes.

## Installation flow

Apply the manifests from the repository root:

```sh
oc apply -n default -f vms/windows2k12-installer.yaml
oc apply -n default -f vms/windows2k12r2-installer.yaml
```

Press a key in VNC if the installation DVD requests it. Setup injects `viostor` in WinPE and
`NetKVM` during offline servicing, so the disk is visible and the network is available in
audit mode. The driver script then locates the driver CD by its contents, checks catalog
signatures, trusts only already-validated publishers, and installs every matching package with
the legacy `pnputil -i -a` syntax, which Server 2012 R2 and earlier require. The QEMU guest
agent MSI follows, and the command returns a reboot request so KMDF and the new drivers are
active before generalization.

After the reboot, a short command renames the cached `Panther\unattend.xml` to
`unattend.install.xml`, and the answer file's native `Generalize` block runs Sysprep in OOBE
mode and shuts the VM down. No separate Sysprep script is involved, matching the Windows 11
and Server 2025 examples.

`pnputil` exit code 259 means no device needed an already-staged package and is counted as
successful, as on the other modern Windows guests. A package that Windows Setup staged during
offline servicing also counts as installed if a catalog cannot be verified.

Logs are `C:\virtio-driver-install.log`, `C:\Windows\Temp\qemu-ga-install.log`, and the
Windows Setup/Sysprep logs. Set the Administrator password in your creation-time answer file.

## Clones

Wait for the installer to shut down, then clone the golden-image DataSource with
`u1.large`, the `windows.2k12.virtio` preference, and a fresh Sysprep volume. Example
answers are [`windows2k12.xml`](../unattend/windows2k12.xml) and
[`windows2k12r2.xml`](../unattend/windows2k12r2.xml); see [cloning.md](cloning.md). The
creation-time answers expand the Windows partition when the clone's disk is larger.
