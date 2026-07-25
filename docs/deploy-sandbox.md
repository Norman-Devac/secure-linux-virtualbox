# Deployment and Verification Scripts

This document provides the consolidated deployment and verification scripts for the Windows 11 sandbox architecture. Executing these instructions applies all security mitigations, networking constraints, temporal obfuscation, and storage containment rules to the engine in a single run. The script utilizes strict POSIX-compliant line continuations and modern syntax bindings to ensure perfect execution on a Linux host.

## Implementation Command

The following script block aggregates all architecture parameters. It configures the isolated internal network, provisions the necessary cryptographic enclaves, shifts the chronological boot epoch, and secures the NVMe storage controller by properly attaching the disk media.

Target identifiers require assignment to the variables at the top of the script prior to execution. The script assumes the virtual disk image possesses pre-injected Red Hat VirtIO network drivers.

```bash
#!/usr/bin/env bash
set -e

VM_NAME="VM-NAME-HERE"
DISK_PATH="Windows.vdi"
STORAGE_CTRL="NVMe-Controller"
NETWORK_NAME="sandbox-net"

# 1. Establish baseline network bandwidth limits
VBoxManage bandwidthctl "$VM_NAME" add "LimitGroup" --type network --limit 10M 2>/dev/null || true

# 2. Apply monolithic hypervisor configuration
VBoxManage modifyvm "$VM_NAME" \
  --nic1=intnet --intnet1="$NETWORK_NAME" --nic-type1=virtio --nic-promisc1=deny \
  --nic2=none --nic3=none --nic4=none --nic5=none --nic6=none --nic7=none --nic8=none \
  --nic-bandwidth-group1="LimitGroup" \
  --boot1=disk --boot2=none --boot3=none --boot4=none \
  --nested-paging=on --large-pages=on --page-fusion=off --x86-vtx-vpid=on \
  --spec-ctrl=on --l1d-flush-on-vm-entry=on --mds-clear-on-vm-entry=on --ibpb-on-vm-entry=on --ibpb-on-vm-exit=on \
  --hpet=on \
  --cpus=4 --cpu-execution-cap=100 \
  --usb=off --usbehci=off --usbxhci=off \
  --audio=none --uart1=off --lpt1=off \
  --iommu=none \
  --graphicscontroller=vboxsvga --accelerate3d=off --vram=128 \
  --default-frontend=gui \
  --mouse=usbtablet \
  --clipboard-mode=disabled --clipboard-file-transfers=off --drag-and-drop=disabled \
  --recording=off --tracing-enabled=off \
  --firmware=efi64 \
  --bios-boot-menu=disabled --bios-logo-display-time=0 \
  --tpm-type=2.0 --paravirt-provider=hyperv \
  --teleporter=off --vrde=off --guestmemoryballoon=0 \
  --nested-hw-virt=off

# 3. Degrade Time Stamp Counter and sever host temporal synchronization
VBoxManage setextradata "$VM_NAME" "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled" 1
VBoxManage setextradata "$VM_NAME" "VBoxInternal/TM/TSCTiedToExecution" 1

# Shift the boot epoch backwards by 30 days (2,592,000,000 milliseconds)
VBoxManage modifyvm "$VM_NAME" --bios-system-time-offset=-2592000000

# 4. Enforce UEFI Secure Boot chain with Microsoft parameters
VBoxManage modifynvram "$VM_NAME" inituefivarstore
VBoxManage modifynvram "$VM_NAME" enrollorclpk
VBoxManage modifynvram "$VM_NAME" enrollmssignatures
VBoxManage modifynvram "$VM_NAME" secureboot --enable

# 5. Lock storage interactions, provision NVMe, and attach the disk medium
VBoxManage storagectl "$VM_NAME" --name "$STORAGE_CTRL" --add=pcie --controller=NVMe --portcount=1 --hostiocache=off
VBoxManage storageattach "$VM_NAME" --storagectl "$STORAGE_CTRL" --port 0 --device 0 --type hdd --medium "$DISK_PATH"

# 6. Seal the disk state and verify shared folder isolation
if VBoxManage showvminfo "$VM_NAME" | grep -q "host_share"; then
    VBoxManage sharedfolder remove "$VM_NAME" --name "host_share" 2>/dev/null || true
    VBoxManage sharedfolder remove "$VM_NAME" --name "host_share" --transient 2>/dev/null || true
fi
VBoxManage modifymedium "$DISK_PATH" --type immutable --autoreset=on
```

## Diagnostic Command

The diagnostic script parses the engine state and extracts all modified flags. This provides verification that the architecture applied successfully without syntax rejection.

```bash
#!/usr/bin/env bash

VM_NAME="VM-NAME-HERE"
DISK_PATH="Windows.vdi"

echo "=== INITIATING HYPERVISOR STATE AUDIT ==="

echo -e "\n[+] Network, Firmware, and I/O Execution States:"
VBoxManage showvminfo "$VM_NAME" | grep -E -i "(NIC 1|Boot Device|Nested Paging|Large Pages|Page Fusion|VPID|HPET|Number of CPUs|CPU exec cap|USB|EHCI|XHCI|Audio|UART|LPT|IOMMU|Graphics Controller|VRAM size|Acceleration|Default Frontend|Pointing Device|Clipboard Mode|Drag'n'drop Mode|Recording|Tracing|Firmware|Secure Boot|TPM|Paravirt|Teleporter|VRDE|balloon|Nested VT-x/AMD-V|Host I/O Cache|Shared folders)"

echo -e "\n[+] Microarchitectural Side-Channel Mitigations (Raw Arrays):"
VBoxManage showvminfo "$VM_NAME" --machinereadable | grep -E "(spec-ctrl|l1d-flush|mds-clear|ibpb|biossystemtimeoffset)"

echo -e "\n[+] Quality of Service and Bandwidth Topology:"
VBoxManage bandwidthctl "$VM_NAME" list

echo -e "\n[+] Temporal Epoch and Desynchronization Overrides:"
VBoxManage getextradata "$VM_NAME" enumerate | grep -E "(GetHostTimeDisabled|TSCTiedToExecution)"

echo -e "\n[+] Cryptographic Storage Medium Verification:"
VBoxManage showmediuminfo "$DISK_PATH" | grep -E -i "(Type|Auto-Reset)"
```

## Optimal Output

When the diagnostic script executes against the properly formatted and securely locked environment, the engine will return the following verified state metrics.

```text
=== INITIATING HYPERVISOR STATE AUDIT ===

[+] Network, Firmware, and I/O Execution States:
NIC 1:                       MAC: 080027XXXXXX, Attachment: Internal Network 'sandbox-net', Cable connected: on, Trace: off (file: none), Type: virtio, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: LimitGroup
Boot Device 1:               HardDisk
Boot Device 2:               Not Assigned
Boot Device 3:               Not Assigned
Boot Device 4:               Not Assigned
Nested Paging:               enabled
Large Pages:                 enabled
VT-x VPID:                   enabled
Page Fusion:                 disabled
HPET:                        enabled
Number of CPUs:              4
CPU exec cap:                100%
Pointing Device:             USB Tablet
USB:                         disabled
EHCI:                        disabled
XHCI:                        disabled
Audio:                       disabled (Driver: Unknown, Controller: Unknown, Codec: Unknown)
UART 1:                      disabled
LPT 1:                       disabled
IOMMU:                       None
Graphics Controller:         VBoxSVGA
VRAM size:                   128MB
3D Acceleration:             disabled
2D Video Acceleration:       disabled
Default Frontend:            gui
Clipboard Mode:              disabled
Drag'n'drop Mode:            disabled
Recording:                   disabled
Tracing Enabled:             disabled
Firmware:                    EFI64
Secure Boot:                 enabled
TPM Type:                    2.0
Paravirt. Provider:          Hyper-V
Effective Paravirt. Prov.:   Hyper-V
VRDE:                        disabled
Teleporter Enabled:          disabled
Guest memory balloon size:   0 Megabytes
Nested VT-x/AMD-V:           disabled
Host I/O Cache:              off
Shared folders:              <none>

[+] Microarchitectural Side-Channel Mitigations (Raw Arrays):
ibpb-on-vm-exit="on"
ibpb-on-vm-entry="on"
spec-ctrl="on"
l1d-flush-on-vm-entry="on"
mds-clear-on-vm-entry="on"
biossystemtimeoffset="-2592000000"

[+] Quality of Service and Bandwidth Topology:
Name: 'LimitGroup', Type: network, Limit: 10 MBytes/sec

[+] Temporal Epoch and Desynchronization Overrides:
Key: VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled, Value: 1
Key: VBoxInternal/TM/TSCTiedToExecution, Value: 1

[+] Cryptographic Storage Medium Verification:
Type:                        immutable
Auto-Reset:                  on
```