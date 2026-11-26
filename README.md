# kubevirt-example-vms

Unattended Windows guest installers for OpenShift Virtualization / KubeVirt. Each manifest
imports its installation ISO, creates a golden-image DataSource, and installs Windows without
interaction except for a single key press at the DVD prompt. The installers generalize the
image and shut the VM down, ready to be cloned.

| Installer | Template | Guest | Media |
| --- | --- | --- | --- |
| [`vms/windows11-installer.yaml`](vms/windows11-installer.yaml) | [`templates/windows11-vm.yaml`](templates/windows11-vm.yaml) | Windows 11 Pro 25H2, x64 | Microsoft consumer ISO |
| [`vms/windows2k25-installer.yaml`](vms/windows2k25-installer.yaml) | [`templates/windows2k25-vm.yaml`](templates/windows2k25-vm.yaml) | Windows Server 2025, x64 | Microsoft evaluation ISO |
| [`vms/windows2k8sp2-x64-installer.yaml`](vms/windows2k8sp2-x64-installer.yaml) | [`templates/windows2k8sp2-x64-vm.yaml`](templates/windows2k8sp2-x64-vm.yaml) | Windows Server 2008 SP2 Standard, x64 | MSDN ISO |
| [`vms/windows2k8sp2-x86-installer.yaml`](vms/windows2k8sp2-x86-installer.yaml) | [`templates/windows2k8sp2-x86-vm.yaml`](templates/windows2k8sp2-x86-vm.yaml) | Windows Server 2008 SP2 Standard, x86 | MSDN ISO |
| [`vms/windows2k8r2-installer.yaml`](vms/windows2k8r2-installer.yaml) | [`templates/windows2k8r2-vm.yaml`](templates/windows2k8r2-vm.yaml) | Windows Server 2008 R2 Standard, x64 | MSDN ISO |
| [`vms/windows2k12-installer.yaml`](vms/windows2k12-installer.yaml) | [`templates/windows2k12-vm.yaml`](templates/windows2k12-vm.yaml) | Windows Server 2012 Standard, x64 | Microsoft evaluation ISO |
| [`vms/windows2k12r2-installer.yaml`](vms/windows2k12r2-installer.yaml) | [`templates/windows2k12r2-vm.yaml`](templates/windows2k12r2-vm.yaml) | Windows Server 2012 R2 Standard, x64 | Microsoft evaluation ISO |

## Usage

Apply a manifest from the repository root. It defines objects in two namespaces: the
installation ISO import and DataSources in `openshift-virtualization-os-images`, and the
VM, answer ConfigMap, and golden-image DataSource in the current namespace.

```sh
oc apply -f vms/windows2k12r2-installer.yaml
```

Press a key once in the VNC console when the DVD asks. Wait for the VM to shut down after
generalization before cloning its golden-image DataSource. The example answers in
[`unattend/`](unattend/) are starting points for clones.

## Templates

Each golden image has a native KubeVirt `VirtualMachineTemplate`
(`template.kubevirt.io/v1beta1`) in [`templates/`](templates/), shipped together with a
ConfigMap that holds the example `unattend.xml`. Apply the file, then create and start a VM:

```sh
oc apply -n default -f templates/windows2k12r2-vm.yaml
virtctl template process windows2k12r2-golden-vm -p NAME=myvm --create
virtctl start myvm
```

`NAME` is generated when omitted. The template clones the golden-image DataSource into an
80 GiB root disk and attaches the `<os>-answer` ConfigMap as its Sysprep volume. That
ConfigMap is shared by every VM created from the template, so edit it before creating VMs.
The templates need KubeVirt 1.9 or later (the feature is beta and enabled by default) and a
virtctl from the same release line; older clients warn about the deprecated v1alpha1 API.

## Documentation

- [Cloning a golden image](docs/cloning.md)
- [Cluster validation results](docs/testing.md)
- [Windows Server 2008 and 2008 R2 details](docs/legacy-windows.md)
- [Windows Server 2012 and 2012 R2 details](docs/windows-server-2012.md)

## Requirements

- OpenShift Virtualization (KubeVirt) with CDI
- The `openshift-virtualization-os-images` namespace
- Pull access to `registry.redhat.io` for the VirtIO driver container
- Outbound access from the cluster to the ISO URLs and, for some examples, Microsoft Update
