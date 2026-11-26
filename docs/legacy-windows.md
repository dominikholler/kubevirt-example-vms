# Windows Server 2008 and Windows Server 2008 R2

All three installer examples use the `windows.2k8.virtio` cluster preference, one active MBR
Windows partition that fills the disk, and modern VirtIO devices.
`useVirtioTransitional` is unnecessary with the pinned drivers: every architecture booted
with non-transitional disk, network, serial, and balloon devices. The driver INFs include
both legacy and modern PCI device IDs. The preference
supplies the VirtIO disk/interface models, ACPI/APIC, Windows timers, and Hyper-V features.
The standard VGA driver handles the preference's VirtIO display.
BIOS/q35 and `host-model` come from KubeVirt defaults. The x64 examples use `u1.large`
(2 vCPUs, 8 GiB); the Server 2008 SP2 x86 example uses `u1.small` (1 vCPU, 2 GiB) and
still runs on KubeVirt's `amd64` host architecture. The reference tests' `qemu64` is
excluded from KubeVirt's default schedulable CPU models.

## Windows Server 2008 SP2

The two Server 2008 SP2 examples disable automatic tablet attachment: the pinned media's
`vioinput` driver supports Server 2008 R2 but not this OS. KubeVirt's alternative USB
tablet uses an xHCI controller that this guest also does not support. The built-in PS/2
keyboard and mouse work without extra drivers.

Server 2008 SP2 has no inbox PowerShell, so the bootstrap downloads Microsoft KB968930
(Windows Management Framework Core, providing PowerShell 2.0) before running the driver
script. Its older `pnputil` reports exit code 1, not 259, for the storage and network
drivers that Setup already injected; the script accepts that code for `viostor` and
`NetKVM` only.

## Windows Server 2008 R2

The Server 2008 R2 example installs Standard (Full), SP1, from Microsoft's multi-edition
media; `/IMAGE/NAME` selects `Windows Server 2008 R2 SERVERSTANDARD`. Server 2008 R2 has
inbox PowerShell 2.0, so no
Windows Management Framework download is required, and its `pnputil` returns 259 for the
already-bound `viostor` driver like the Server 2012 family.

The pinned media ships 10 driver packages for this OS. Nine of their catalogs are SHA-1
signed, so a fresh, unpatched guest verifies them without the SHA-2 servicing updates
KB4474419 and KB4490628. `NetKVM` is the exception: its catalog is SHA-256 signed and
chains through `Microsoft Windows Third Party Component CA 2012` to the
`Microsoft Root Certificate Authority 2010`, which this OS does not trust. Windows Setup
has already injected `NetKVM` through offline servicing when the audit-mode script runs,
so the script recognizes the package in the driver store and does not reinstall it.
The SHA-2 servicing updates are only needed once the whole driver media uses SHA-256
catalogs, such as virtio-win 1.9.58; that flow must install KB4490628 and KB4474419,
reboot, and only then run the driver script.

`vioinput` supports Server 2008 R2, so the example keeps the preference's VirtIO tablet
enabled. The PS/2 keyboard remains available during installation, before that driver is
installed.

## Pinned driver container

The examples reference
`registry.redhat.io/cnv-tech-preview/virtio-win:v1.4.0-2` and pin the digest:

```text
sha256:6ee05b1f81263da91eea6da1e93077752ecf4c293b3b255be63f4398578b2028
```

The tag keeps the manifest readable; the digest makes the pull immutable.

No driver ISO upload is needed. The cluster must be able to pull from `registry.redhat.io`;
credentials are supplied by the cluster, not embedded in these public manifests. Windows
installation ISOs are imported from the public URLs in the manifests.

All **66** `.inf`, `.cat`, `.sys`, `.dll`, and `.exe` files under this image's `*/2k8/amd64`
and `*/2k8/x86` directories were compared byte for byte with the compatibility repository's
VirtIO 1.9.12 ISO. Every file matched. Its guest-agent MSIs also match the known 7.6.2
packages in VirtIO 1.9.5 and 1.9.7. The reference 1.9.12 ISO's SHA-256 is
`f43ad098e7b9887ede9a071da3157474b92a3009432d932b581bc3368258ed6b`.

The same comparison for the `*/2k8R2/amd64` directories found all **40** matching
`.inf`, `.cat`, `.sys`, `.dll`, and `.exe` files across the 10 driver packages, again by
name, size, and CRC-32. That covers the legacy and modern PCI IDs of `viostor`
(`0x1001`, `0x1042`) and `NetKVM` (`0x1000`, `0x1041`). Check certificate chains before
assuming a catalog is verifiable: the pinned 1.9.12 set mixes SHA-1 catalogs with the
SHA-256 `NetKVM` catalog described above.

Use these exact media versions:

| Purpose | Media | Reason |
| --- | --- | --- |
| VirtIO drivers | Pinned `cnv-tech-preview/virtio-win:v1.4.0-2` | Byte-identical to the tested 1.9.12 `2k8` and `2k8R2` drivers, including Microsoft-signed catalogs |
| QEMU guest agent | Same container, agent 7.6.2 | Agent 7.6.2 passed all three Server 2008 tests |
| PowerShell 2.0 | Microsoft KB968930 for the guest architecture | Server 2008 SP2 has no inbox PowerShell; Server 2008 R2 does |

The guest agent **101.1.0 bundled with VirtIO 1.9.12 is incompatible** with Server 2008 SP2.
Driver and agent compatibility must be selected independently.

The newer `container-native-virtualization/virtio-win` images checked (`v2.0.0-2`,
`v2.2.0-4`, and `v2.4.0-15`) omit `2k8`, retaining only `2k8R2`. In particular,
`v2.4.0-15` labels its ISO `virtio-win-1.9.12` but lacks the required legacy directories.
An ISO version label alone is therefore insufficient. Public upstream 0.1.160 and 0.1.173
catalogs also failed trust-chain validation on a fresh guest; these are not interchangeable
with the Microsoft-signed drivers tested here.

## Installation flow

Apply the manifests from the repository root:

```sh
oc apply -n default -f vms/windows2k8sp2-x64-installer.yaml
oc apply -n default -f vms/windows2k8sp2-x86-installer.yaml
oc apply -n default -f vms/windows2k8r2-installer.yaml
```

Each manifest also defines the Windows image import and its DataSources in
`openshift-virtualization-os-images`. Apply the file without `-n` if your CLI rejects
objects whose namespace differs from the one on the command line, or split those objects
out; the VM and ConfigMap land in the current namespace either way.

Press a key in VNC if the installation DVD requests it. Setup injects `viostor` in WinPE
and `NetKVM` during offline servicing. The driver script then locates the driver CD by its
contents, checks catalog signatures, trusts only already-validated publishers, and runs
legacy `pnputil -i -a` to install the remaining drivers online. A package that Setup
staged during offline servicing counts as installed when the guest cannot verify its
catalog. This allows the KMDF co-installers to run: offline staging alone left KMDF 1.7
installed on Server 2008 SP2 while the serial and balloon drivers required KMDF 1.9.

On Server 2008 SP2, the bootstrap first downloads KB968930 from Microsoft, verifies its
pinned checksum using the original OS's SHA-1-only `certutil`, and runs WUSA, which verifies
Microsoft's package signature. The package hashes match the copies used by the compatibility
tests. Server 2008 R2 skips this step because PowerShell is inbox.

Server 2008's `pnputil` return code 1 is accepted only for the storage/network drivers already
injected by Setup, following the reference's documented finding; Server 2008 R2 reports the
`ERROR_NO_MORE_ITEMS` code 259 for the same drivers. Other install failures stop the workflow
before Sysprep.

After agent installation, Setup reboots before the next `auditUser` command. On Server 2008
R2 that command renames the cached `Panther\unattend.xml` to `unattend.install.xml`, and the
answer file's native `Generalize` block then generalizes in OOBE mode and shuts the VM down.
No Sysprep script is needed. The two Server 2008 SP2 examples keep their explicit Sysprep
script, the path verified for that OS.
The single active Windows partition is an intentional difference from the compatibility
repository's two-partition installation tests. On NT 6.0, the separate System Reserved
partition could take `C:` after generalization and move Windows to `D:`, causing Setup to
reject the image's registry paths. The Server 2008 R2 example uses the same single-partition
layout for consistency.
The media-discovery command explicitly propagates the script's exit code. A plain
`cmd /c for ...` loop returned zero even when its child requested a reboot with exit code 1,
so Setup immediately launched Sysprep with KMDF 1.7 still installed. The corrected wrapper
uses delayed expansion and `exit /b !errorlevel!`, allowing Setup to reboot and activate
KMDF 1.9 before generalization.
Only clone the golden-image DataSource after shutdown. The clone needs the root disk
and matching firmware and VirtIO disk/network models, plus a new Sysprep volume with your
`unattend.xml`.
Installation CDs are unnecessary. The example creation-time answers expand the Windows
partition when the clone's disk is larger. See [cloning.md](cloning.md).

Logs are `C:\virtio-driver-install.log`, `C:\Windows\Temp\qemu-ga-install.log`, and the
Windows Setup/Sysprep logs. Set the Administrator password in your creation-time answer file.

These choices follow the `virtio-compatibility` project's Server 2008 and Server 2008 R2
answer files, `install-all-virtio-drivers.ps1`, driver-version inventory, and recorded
agent-version sweeps. See [cluster test results](testing.md) for the separate KubeVirt
validation.
