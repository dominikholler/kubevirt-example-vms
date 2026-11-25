# Windows Server 2008 SP2

Both installer examples use the `windows.2k8.virtio` cluster preference, one active MBR
Windows partition that fills the disk, and modern VirtIO devices.
`useVirtioTransitional` is unnecessary with the pinned drivers: both architectures booted
with non-transitional disk, network, serial, and balloon devices. The driver INFs include
both legacy and modern PCI device IDs. The preference
supplies the VirtIO disk/interface models, ACPI/APIC, Windows timers, and Hyper-V features.
Disable automatic tablet attachment: the preference's VirtIO tablet has no Server 2008 SP2
driver on the pinned media. KubeVirt's USB tablet uses an xHCI controller that is also
unsupported by this guest. The built-in PS/2 keyboard and mouse work without extra drivers.
The standard VGA driver handles the preference's VirtIO display.
BIOS/q35 and `host-model` come from KubeVirt defaults. The x64 example uses `u1.large`
(2 vCPUs, 8 GiB); x86 uses `u1.small` (1 vCPU, 2 GiB). The x86 guest still runs on
KubeVirt's `amd64` host architecture.
The reference test's `qemu64` is excluded from KubeVirt's default schedulable CPU models.

## Pinned driver container

The examples use `registry.redhat.io/cnv-tech-preview/virtio-win:v1.4.0-2`, pinned by digest:

```text
sha256:6ee05b1f81263da91eea6da1e93077752ecf4c293b3b255be63f4398578b2028
```

No driver ISO upload is needed. The cluster must be able to pull from `registry.redhat.io`;
credentials are supplied by the cluster, not embedded in these public manifests. Windows
installation ISOs are imported from the public URLs in the manifests.

All **66** `.inf`, `.cat`, `.sys`, `.dll`, and `.exe` files under this image's `*/2k8/amd64`
and `*/2k8/x86` directories were compared byte for byte with the compatibility repository's
VirtIO 1.9.12 ISO. Every file matched. Its guest-agent MSIs also match the known 7.6.2
packages in VirtIO 1.9.5 and 1.9.7. The reference 1.9.12 ISO's SHA-256 is
`f43ad098e7b9887ede9a071da3157474b92a3009432d932b581bc3368258ed6b`.

Use these exact media versions:

| Purpose | Media | Reason |
| --- | --- | --- |
| VirtIO drivers | Pinned `cnv-tech-preview/virtio-win:v1.4.0-2` | Byte-identical to the tested 1.9.12 `2k8` drivers, including Microsoft-signed catalogs |
| QEMU guest agent | Same container, agent 7.6.2 | Agent 7.6.2 passed both Server 2008 architecture tests |
| PowerShell 2.0 | Microsoft KB968930 for the guest architecture | Server 2008 SP2 has no inbox PowerShell |

The guest agent **101.1.0 bundled with VirtIO 1.9.12 is incompatible** with Server 2008 SP2.
Driver and agent compatibility must be selected independently.

The newer `container-native-virtualization/virtio-win` images checked (`v2.0.0-2`,
`v2.2.0-4`, and `v2.4.0-15`) omit `2k8`, retaining only `2k8R2`. In particular,
`v2.4.0-15` labels its ISO `virtio-win-1.9.12` but lacks the required legacy directories.
An ISO version label alone is therefore insufficient. Public upstream 0.1.160 and 0.1.173
catalogs also failed trust-chain validation on a fresh guest; these are not interchangeable
with the Microsoft-signed drivers tested here.

## Installation flow

Apply either manifest from the repository root:

```sh
oc apply -n default -f vms/windows2k8sp2-x64-installer.yaml
oc apply -n default -f vms/windows2k8sp2-x86-installer.yaml
```

Press a key in VNC if the installation DVD requests it. Setup injects `viostor` in WinPE
and `NetKVM` during offline servicing. In audit mode, the bootstrap downloads KB968930 from
Microsoft, verifies its pinned checksum using the original OS's SHA-1-only `certutil`, and
runs WUSA, which verifies Microsoft's package signature. The package hashes match the
copies used by the compatibility tests.

The driver script then locates the driver CD by its contents, checks catalog signatures,
trusts only already-validated publishers, and runs legacy `pnputil -i -a` to install the
remaining drivers online. This allows the KMDF co-installers to run: offline staging alone
left KMDF 1.7 installed while the serial and balloon drivers required KMDF 1.9.
Server 2008's `pnputil` return code 1 is accepted only for the storage/network drivers already
injected by Setup, following the reference's documented finding. Other install failures stop
the workflow before Sysprep.

After agent installation, Setup reboots before the next `auditUser` command. That command
renames the cached `Panther\unattend.xml` to `unattend.install.xml` and runs Sysprep to
generalize and shut down.
The single active Windows partition is an intentional difference from the compatibility
repository's two-partition installation tests. On NT 6.0, the separate System Reserved
partition could take `C:` after generalization and move Windows to `D:`, causing Setup to
reject the image's registry paths. Hiding that partition avoided the letter conflict but
still failed the clone's component-configuration phase.
The media-discovery command explicitly propagates the script's exit code. A plain
`cmd /c for ...` loop returned zero even when its child requested a reboot with exit code 1,
so Setup immediately launched Sysprep with KMDF 1.7 still installed. The corrected wrapper
uses delayed expansion and `exit /b !errorlevel!`, allowing Setup to reboot and activate
KMDF 1.9 before generalization. Preserving device installations alone did not fix that
missing reboot and is not used.
Only clone the golden-image DataSource after shutdown. The clone needs the root disk
and matching firmware and VirtIO disk/network models, plus a new Sysprep volume with your
`unattend.xml`.
Installation CDs are unnecessary. The example creation-time answers expand the Windows
partition when the clone's disk is larger. See [cloning.md](cloning.md).

Logs are `C:\virtio-driver-install.log`, `C:\Windows\Temp\qemu-ga-install.log`, and the
Windows Setup/Sysprep logs. Set the Administrator password in your creation-time answer file.

These choices follow the `virtio-compatibility` project's Server 2008 answer files,
`install-all-virtio-drivers.ps1`, driver-version inventory, and recorded agent-version sweep.
See [cluster test results](testing.md) for the separate KubeVirt validation.
