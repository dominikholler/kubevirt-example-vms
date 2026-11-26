# Create a VM from a generalized image

The installer manifests create generalized disks and shut down. They rename `C:\Windows\Panther\unattend.xml`
to `unattend.install.xml` before Sysprep, retaining the installation answer for reference.
Per-VM OOBE settings are supplied only when creating the clone.
Modern Windows can recreate `Panther\unattend.xml` during native generalization, with
its installation passes marked `wasPassProcessed="true"`. This processed cache contains no
clone-specific settings. The Windows 11, Server 2025, Server 2008 R2, Server 2012, and
Server 2012 R2 tests confirmed that a fresh Sysprep volume supplies the clone's settings.
Provide your own `unattend.xml` on a **new Sysprep volume when creating each VM**.

The files in [`unattend/`](../unattend/) are optional starting points. They configure the
Administrator account and expand Windows to fill the cloned disk. Edit the password in both
the account and AutoLogon sections, and customize the computer name, time zone, and other
settings as needed. These files are never attached to the installers.

| Golden-image DataSource | Example creation-time answer |
| --- | --- |
| `windows11-golden-image` | [`windows11.xml`](../unattend/windows11.xml) |
| `windows2k25-golden-image` | [`windows2k25.xml`](../unattend/windows2k25.xml) |
| `windows2k8sp2-x64-golden-image` | [`windows2k8sp2-x64.xml`](../unattend/windows2k8sp2-x64.xml) |
| `windows2k8sp2-x86-golden-image` | [`windows2k8sp2-x86.xml`](../unattend/windows2k8sp2-x86.xml) |
| `windows2k8r2-golden-image` | [`windows2k8r2.xml`](../unattend/windows2k8r2.xml) |
| `windows2k12-golden-image` | [`windows2k12.xml`](../unattend/windows2k12.xml) |
| `windows2k12r2-golden-image` | [`windows2k12r2.xml`](../unattend/windows2k12r2.xml) |

Wait for the installer to shut down successfully before cloning. A ready DataSource alone
does not mean that Windows installation or Sysprep has finished.

## From a template

The fastest path is the matching native `VirtualMachineTemplate` in
[`templates/`](../templates/). It clones the golden-image DataSource into an 80 GiB root disk
and already contains the example answer in a shared `<os>-answer` ConfigMap:

```sh
oc apply -n default -f templates/windows2k12r2-vm.yaml
virtctl template process windows2k12r2-golden-vm -p NAME=myvm --create
virtctl start myvm
```

Both objects land in the namespace you apply them to. Set `GOLDEN_IMAGE_NAMESPACE` when the
golden-image DataSource lives elsewhere, and edit the ConfigMap before creating VMs if you
want different OOBE settings.

## Manually, with your own answer

To keep each VM's answer separate, create a Secret with a customized `unattend.xml` and attach
it as a Sysprep volume. For example, save your customized Windows 11 answer as
`my-unattend.xml`, then create a Secret in the new VM's namespace. Its data key must be
exactly `unattend.xml`:

```sh
oc create secret generic my-windows-answer -n default \
  --from-file=unattend.xml=my-unattend.xml
```

Save the following as `my-windows.yaml` and apply it with `oc apply -n default -f my-windows.yaml`:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: my-windows
spec:
  runStrategy: RerunOnFailure
  instancetype:
    name: u1.large
  preference:
    name: windows.11.virtio
  dataVolumeTemplates:
    - metadata:
        name: my-windows-root
      spec:
        sourceRef:
          kind: DataSource
          name: windows11-golden-image
          namespace: default
        storage:
          resources:
            requests:
              storage: 80Gi
  template:
    spec:
      domain:
        devices:
          disks:
            - name: rootdisk
              bootOrder: 1
              disk:
                bus: virtio
            - name: sysprep
              cdrom:
                bus: sata
          interfaces:
            - name: default
              model: virtio
              masquerade: {}
      networks:
        - name: default
          pod: {}
      volumes:
        - name: rootdisk
          dataVolume:
            name: my-windows-root
        - name: sysprep
          sysprep:
            secret:
              name: my-windows-answer
```

For Server 2025, use its DataSource and the `windows.2k25.virtio` preference. For Server 2012
and Server 2012 R2, use `windows.2k12.virtio` and `u1.large`. For Server 2008
and Server 2008 R2, use `windows.2k8.virtio` and choose `u1.large` for x64 or `u1.small` for
the x86 Server 2008 SP2 example.
`useVirtioTransitional` is unnecessary with the pinned drivers. Also disable the unsupported
tablet under `domain.devices` for Server 2008 SP2; Server 2008 R2 keeps the preference's
VirtIO tablet because its pinned `vioinput` driver supports that OS:

```yaml
autoattachInputDevice: false
```

Keep only the cloned root
disk and the new Sysprep volume; the Windows installation
ISO and VirtIO driver CD are unnecessary. The x86 answer uses `processorArchitecture="x86"`,
while KubeVirt still uses host architecture `amd64`.

Windows consumes the supplied answer during the clone's first boot. Changing it after that
boot does not re-run specialization; create a fresh clone to test a different answer file.
