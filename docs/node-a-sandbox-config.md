# Oracle VM VirtualBox Sandbox Architecture Configuration (Node C)

This document outlines the technical configuration for establishing an isolated sandbox environment using Oracle VM VirtualBox for a Windows 11 guest. The configuration balances system stability and visual fluidity for recording purposes, enforces strict execution constraints, manages cryptographic states, and limits networking surfaces. The architecture integrates critical modifications to the Desktop Window Manager pipeline and multi-core synchronization protocols to ensure an interactive analysis state.

## Prerequisite: Hypervisor Session State Cleansing
The hypervisor enforces strict single-writer session locks on virtual machine configuration files. The initialization sequence mandates a preemptive termination of all background daemon processes to guarantee successful parameter injection and prevent invalid object state rejections before executing any architectural changes.

**Implementation Command:**

    killall -9 VirtualBox VBoxSVC VBoxHeadless 2>/dev/null || true
    sleep 2

## 1. Physical Host Kernel Hardening
Securing the execution boundary requires fortifying the physical host operating system against advanced hypervisor escape techniques. The engine utilizes the sysctl daemon to inject restrictive kernel parameters. Blinding the Extended Berkeley Packet Filter Just-In-Time compiler neutralizes heap spraying exploits, while strict pointer restrictions prevent malicious payloads from mapping the host kernel memory layout.

**Implementation Command:**

    cat << 'EOF' | sudo tee /etc/sysctl.d/99-sandbox-hardening.conf
    kernel.kptr_restrict = 2
    net.core.bpf_jit_harden = 2
    kernel.unprivileged_bpf_disabled = 1
    kernel.yama.ptrace_scope = 2
    vm.unprivileged_userfaultfd = 0
    kernel.dmesg_restrict = 1
    EOF
    sudo sysctl --system

**Diagnostic Command:**

    sysctl kernel.kptr_restrict net.core.bpf_jit_harden

**Optimal Output:**

    kernel.kptr_restrict = 2
    net.core.bpf_jit_harden = 2

## 2. Internal Network Assignment and VirtIO Offloading
The network bridging configuration isolates the primary virtual adapter to an internal memory switch, neutralizing uncontrolled egress routing. Utilizing the paravirtualized VirtIO driver circumvents emulation vulnerabilities and leverages a streamlined shared-memory ring buffer mechanism that bypasses hardware emulation entirely. Binding the nullified secondary interfaces to a permanent disabled state mathematically reduces the network stack attack surface.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --nic1=intnet --intnet1="VBOX-NETWORK-SANDBOX" --nic-type1=virtio --nic-promisc1=deny
    VBoxManage modifyvm "NODE-A-VM-NAME" --nic2=none --nic3=none --nic4=none --nic5=none --nic6=none --nic7=none --nic8=none

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "NIC 1"

**Optimal Output:**

    NIC 1:                       MAC: 080027XXXXXX, Attachment: Internal Network 'VBOX-NETWORK-SANDBOX', Cable connected: on, Trace: off (file: none), Type: virtio, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: NODE-A-BANDWIDTH-GROUP

## 3. Network Bandwidth Throttling
Establishing a strict ceiling on outbound network transmission prevents a compromised detonation node from executing denial-of-service floods against synthetic infrastructure or external networks. The token-bucket bandwidth controller enforces a hard limit of ten megabytes per second. This ensures covert lateral floods remain mathematically impossible while preserving guest uptime.

**Implementation Command:**

    VBoxManage bandwidthctl "NODE-A-VM-NAME" add "NODE-A-BANDWIDTH-GROUP" --type network --limit=10M 2>/dev/null || true
    VBoxManage modifyvm "NODE-A-VM-NAME" --nic-bandwidth-group1="NODE-A-BANDWIDTH-GROUP"

**Diagnostic Command:**

    VBoxManage bandwidthctl "NODE-A-VM-NAME" list

**Optimal Output:**

    Name: 'NODE-A-BANDWIDTH-GROUP', Type: network, Limit: 10 MBytes/sec

## 4. Network Boot Deactivation
Restricting the boot sequence strictly to the local non-volatile storage eliminates a highly complex exploitation vector. The architecture forcibly nullifies all boot device slots except the primary disk. Severing the network initialization sequence removes the theoretical attack surface against the virtual firmware entirely before the operating system kernel is loaded into memory.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --boot1=disk --boot2=none --boot3=none --boot4=none

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "Boot Device"

**Optimal Output:**

    Boot Device 1:               HardDisk
    Boot Device 2:               Not Assigned
    Boot Device 3:               Not Assigned
    Boot Device 4:               Not Assigned

## 5. Hardware-Assisted Paging and TLB Optimization
Activating nested paging allows the physical processor to manage memory translation directly. Explicitly disabling large pages prevents physical memory fragmentation, prioritizing system viability and ensuring rapid guest boot times. Modifying page fusion is omitted to prevent architecture detection faults on 64-bit host kernels.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --nested-paging=on --large-pages=off
    VBoxManage modifyvm "NODE-A-VM-NAME" --x86-vtx-vpid=on

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Nested Paging|Large Pages|VPID)"

**Optimal Output:**

    Nested Paging:               enabled
    Large Pages:                 disabled
    VT-x VPID:                   enabled

## 6. Microarchitectural Buffer Optimization
Mitigating speculative execution vulnerabilities requires hardware buffering. Instructing the hypervisor to flush the Level 1 Data Cache strictly upon virtual machine entry balances the requirement to prevent transient execution memory leaks with the necessity of maintaining fluid execution dynamics. Disabling the indirect branch predictor barrier on virtual machine exit avoids destructive instruction latency and prevents Deferred Procedure Call watchdog timeouts.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --spec-ctrl=on --l1d-flush-on-vm-entry=on --mds-clear-on-vm-entry=on --ibpb-on-vm-entry=on --ibpb-on-vm-exit=off

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" --machinereadable | grep -E "(spec-ctrl|l1d-flush|mds-clear|ibpb)"

**Optimal Output:**

    ibpb-on-vm-exit="off"
    ibpb-on-vm-entry="on"
    spec-ctrl="on"
    l1d-flush-on-vm-entry="on"
    mds-clear-on-vm-entry="on"

## 7. Execution Architecture and Core Allocation
The architecture assigns four physical processing cores with a strict ninety percent execution cap to preserve the thermal design power limits of the physical host processor. The baseline physical memory allocation of 8192 Megabytes guarantees that Virtualization-Based Security frameworks possess the necessary contiguous address space for flawless initialization, completely eradicating STATUS_NO_MEMORY exceptions.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --cpus=4 --cpu-execution-cap=90 --memory=8192

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Number of CPUs|CPU exec cap|Memory size)"

**Optimal Output:**

    Memory size:                 8192MB
    Number of CPUs:              4
    CPU exec cap:                90%

## 8. Legacy Hardware and USB Disconnection
Removing legacy hardware protocols achieves a dramatic reduction in the attack surface. Standard syntax nullifies the universal serial bus controllers and eliminates the Extensible Host Controller Interface. The emulated audio backend is deactivated by removing the driver binding entirely. Restricting pointing inputs strictly to the legacy PS/2 protocol permanently closes communication vectors utilized for out-of-bounds reads against the hypervisor boundaries.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --mouse=ps2
    VBoxManage modifyvm "NODE-A-VM-NAME" --usb=off --usbehci=off --usbxhci=off
    VBoxManage modifyvm "NODE-A-VM-NAME" --audio-driver=none
    VBoxManage modifyvm "NODE-A-VM-NAME" --uart1=off --lpt1=off

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Pointing Device|USB|EHCI|XHCI|Audio:|UART|LPT)"

**Optimal Output:**

    Pointing Device:             PS/2 Mouse
    USB:                         disabled
    EHCI:                        disabled
    XHCI:                        disabled
    Audio:                       disabled (Driver: None, Codec: Unknown)
    UART 1:                      disabled
    LPT 1:                       disabled

## 9. IOMMU and Hardware Translation Lockdown
The configuration utilizes automatic provisioning for the Input-Output Memory Management Unit and integrates the ICH9 virtual chipset. This permits the guest to execute internal direct memory access remapping successfully via Message Signaled Interrupts, ensuring the payload environment supports virtualization-based security frameworks without needlessly sacrificing system viability or inducing Secure Kernel initialization failures.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --iommu=automatic --chipset=ich9

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(IOMMU|Chipset)"

**Optimal Output:**

    IOMMU:                       Automatic
    Chipset:                     ich9

## 10. Graphics Controller and Hardware Acceleration
Enabling 3D hardware acceleration and assigning exactly 128 Megabytes of video memory provides the Windows kernel with sufficient resources to maintain the Desktop Window Manager compositor pipeline. This precise boundary prevents Peripheral Component Interconnect memory-mapped input/output overlaps that trigger persistent black screen loops.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --graphicscontroller=vboxsvga --accelerate-3d=on --vram=128
    VBoxManage modifyvm "NODE-A-VM-NAME" --default-frontend=gui

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Graphics Controller|VRAM size|3D Acceleration|Default Frontend)"

**Optimal Output:**

    Graphics Controller:         VBoxSVGA
    VRAM size:                   128MB
    3D Acceleration:             enabled
    Default Frontend:            gui

## 11. Interaction Processes and Telemetry Disconnection
The sandbox operates as an isolated containment vessel. Disabling shared clipboards, file transfers, and graphical drag-and-drop mechanisms removes bidirectional socket communication. Closing recording buffers restricts the memory-mapped avenues available for out-of-bounds manipulation.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --clipboard-mode=disabled --clipboard-file-transfers=disabled --drag-and-drop=disabled
    VBoxManage modifyvm "NODE-A-VM-NAME" --recording=off --tracing-enabled=off

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Clipboard Mode|Drag'n'drop Mode|Recording|Tracing)"

**Optimal Output:**

    Clipboard Mode:              disabled
    Drag'n'drop Mode:            disabled
    Recording:                   disabled
    Tracing Enabled:             disabled

## 12. Firmware Integrity and UEFI Secure Boot
Injecting digital certificates directly into the non-volatile memory of the virtual motherboard forces the Extensible Firmware Interface to cryptographically verify the operating system initialization chain. Disabling the basic input/output system menu visually accelerates the detonation process while sealing the pre-boot environment.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --firmware=efi64
    VBoxManage modifynvram "NODE-A-VM-NAME" inituefivarstore
    VBoxManage modifynvram "NODE-A-VM-NAME" enrollorclpk
    VBoxManage modifynvram "NODE-A-VM-NAME" enrollmssignatures
    VBoxManage modifynvram "NODE-A-VM-NAME" secureboot --enable
    VBoxManage modifyvm "NODE-A-VM-NAME" --bios-boot-menu=disabled --bios-logo-display-time=0

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Firmware|Secure Boot)"

**Optimal Output:**

    Firmware:                    EFI64
    Secure Boot:                 enabled

## 13. TPM Provisioning and Paravirtualization Stabilization
Generating a software-based Trusted Platform Module natively within the hypervisor fulfills hardware anchor requirements. Configuring the paravirtualization provider strictly to the Hyper-V standard orchestrates accurate interrupt routing between the guest kernel and the virtual processor cores, guaranteeing sustained system responsiveness.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --tpm-type=2.0
    VBoxManage modifyvm "NODE-A-VM-NAME" --paravirt-provider=hyperv
    VBoxManage modifyvm "NODE-A-VM-NAME" --hpet=on
    VBoxManage setextradata "NODE-A-VM-NAME" "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled" 0

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(TPM|Paravirt|HPET)"

**Optimal Output:**

    HPET:                        enabled
    TPM Type:                    2.0
    Paravirt. Provider:          Hyper-V
    Effective Paravirt. Prov.:   Hyper-V

## 14. Teleportation, VRDE, and Memory Ballooning
Disabling the remote desktop server endpoints eliminates listening transmission control protocol sockets on the host. Constraining the memory balloon to zero megabytes prevents software from interacting with guest integration modules in an attempt to trigger physical RAM exhaustion.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --teleporter=off --vrde=off
    VBoxManage modifyvm "NODE-A-VM-NAME" --guest-memory-balloon=0

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Teleporter|VRDE|balloon)"

**Optimal Output:**

    VRDE:                        disabled
    Teleporter Enabled:          disabled
    Guest memory balloon size:   0 Megabytes

## 15. Nested Hardware Virtualization Constraints
Enabling nested hardware virtualization perfectly aligns the hypervisor envelope with the mandatory architectural demands of modern enterprise security components. This functionality allows the guest kernel to construct isolated micro-visors for structural validation.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --nested-hw-virt=on

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "Nested VT-x/AMD-V"

**Optimal Output:**

    Nested VT-x/AMD-V:           enabled

## 16. SATA Storage, Host I/O Caching, and Disk Attachment
The architecture relies on a virtualized Serial ATA controller operating via the Advanced Host Controller Interface. Maintaining the native storage controller topology prevents early boot storage driver panics inside the guest operating system. Explicitly disabling host-side input/output caching ensures continuous disk writes interact exclusively with the hypervisor boundary rather than polluting the physical memory stack of the host OS.

**Implementation Command:**

    VBoxManage storagectl "NODE-A-VM-NAME" --name "SATA" --add sata --controller IntelAhci --portcount 2 --hostiocache off

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "Host I/O Cache"

**Optimal Output:**

    Host I/O Cache:              off

## 17. Shared Folder Removal
The architecture systematically obliterates all permanent and transient folder allocations. Executing the commands with silenced standard error output safely ignores non-existent directory warnings while destroying primary file-system traversal vectors.

**Implementation Command:**

    VBoxManage sharedfolder remove "NODE-A-VM-NAME" --name "NODE-A-SHARED-FOLDER-NAME" 2>/dev/null || true
    VBoxManage sharedfolder remove "NODE-A-VM-NAME" --name "NODE-A-SHARED-FOLDER-NAME" --transient 2>/dev/null || true

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "Shared folders" -A 2

**Optimal Output:**

    Shared folders:              <none>

## 18. Immutable Disk State Isolation
To maintain absolute forensic purity, the primary virtual disk media is locked into an immutable state. The hypervisor orchestrates a differencing overlay for all system modifications automatically upon initialization. The configuration isolates the globally registered identifier, detaches the medium from the controller to break state dependencies, applies the immutable lock, and maps the secured disk back to the primary storage pipeline.

**Implementation Command:**

    VDI_UUID=$(VBoxManage list hdds | grep -e "^UUID:" -e "^Location:" | grep -B 1 -i "NODE-A-VDI-FILENAME" | grep "^UUID:" | awk '{print $2}' | head -n 1)
    VBoxManage storageattach "NODE-A-VM-NAME" --storagectl "SATA" --port 0 --device 0 --type hdd --medium none 2>/dev/null || true
    VBoxManage modifymedium "$VDI_UUID" --type immutable
    VBoxManage storageattach "NODE-A-VM-NAME" --storagectl "SATA" --port 0 --device 0 --type hdd --medium "$VDI_UUID"

**Diagnostic Command:**

    VDI_UUID=$(VBoxManage list hdds | grep -e "^UUID:" -e "^Location:" | grep -B 1 -i "NODE-A-VDI-FILENAME" | grep "^UUID:" | awk '{print $2}' | head -n 1)
    VBoxManage showmediuminfo "$VDI_UUID" | grep -E -i "(Type)"

**Optimal Output:**

    Type:                        immutable