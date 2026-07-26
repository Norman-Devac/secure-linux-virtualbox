# Oracle VM VirtualBox 7.2.14 Sandbox Architecture Configuration

This document outlines the technical configuration for establishing an isolated sandbox environment using Oracle VM VirtualBox 7.2.14 for a Windows 11 guest. The configuration balances system stability and visual fluidity for recording purposes, enforces strict execution constraints, manages cryptographic states, and limits networking surfaces. The architecture integrates critical modifications to the Desktop Window Manager pipeline and multi-core synchronization protocols to ensure an interactive analysis state.

## 1. Internal Network Assignment and VirtIO Offloading
The command links the primary adapter to an isolated switch in host memory, preventing external traffic routing. Using a paravirtualized driver bypasses emulation vulnerabilities, creating an efficient memory channel that communicates directly with the hypervisor boundary. By utilizing the VirtIO standard, the engine eliminates the need to emulate complex hardware states, minimizing the attack surface associated with malicious packet crafting targeting virtual network interface cards. Disabling secondary interfaces blocks alternative escape routes. The engine explicitly marks nullified interfaces as disabled.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --nic1=intnet --intnet1="VBOX-NETWORK-SANDBOX" --nic-type1=virtio --nic-promisc1=deny
    VBoxManage modifyvm "NODE-A-VM-NAME" --nic2=none --nic3=none --nic4=none --nic5=none --nic6=none --nic7=none --nic8=none

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "NIC 1"

**Optimal Output:**

    NIC 1:                       MAC: 080027XXXXXX, Attachment: Internal Network 'VBOX-NETWORK-SANDBOX', Cable connected: on, Trace: off (file: none), Type: virtio, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: NODE-A-BANDWIDTH-GROUP

## 2. Network Bandwidth Throttling
The configuration creates a transmission limit group capped at ten megabytes per second and binds it to the primary network adapter. The token-bucket system silently drops outbound packets that exceed this threshold. By enforcing a hard ceiling on network throughput, the architecture mitigates the risk of the sandbox being utilized as a staging ground for denial-of-service floods or rapid lateral data exfiltration. The limit remains transparent to the guest, preventing the execution of lateral network floods without alerting the payload.

**Implementation Command:**

    VBoxManage bandwidthctl "NODE-A-VM-NAME" add "NODE-A-BANDWIDTH-GROUP" --type network --limit=10M 2>/dev/null || true
    VBoxManage modifyvm "NODE-A-VM-NAME" --nic-bandwidth-group1="NODE-A-BANDWIDTH-GROUP"

**Diagnostic Command:**

    VBoxManage bandwidthctl "NODE-A-VM-NAME" list

**Optimal Output:**

    Name: 'NODE-A-BANDWIDTH-GROUP', Type: network, Limit: 10 MBytes/sec

## 3. Network Boot Deactivation
The configuration restricts the boot sequence strictly to the attached local disk. Setting remaining devices to a null state removes network boot protocols from the initialization phase. The diagnostic output specifically identifies these disconnected slots as not assigned. This prevents firmware from parsing malicious Preboot Execution Environment network configuration packets, effectively severing an initial exploitation vector before the operating system kernel is loaded into memory.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --boot1=disk --boot2=none --boot3=none --boot4=none

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "Boot Device"

**Optimal Output:**

    Boot Device 1:               HardDisk
    Boot Device 2:               Not Assigned
    Boot Device 3:               Not Assigned
    Boot Device 4:               Not Assigned

## 4. Hardware-Assisted Paging and TLB Optimization
Activating nested paging allows the physical processor to manage memory translation directly, while large pages reduce translation overhead. By utilizing Extended Page Tables, the hypervisor prevents the guest from directly manipulating host memory mappings. The architecture-specific parameter enables virtual processor identifiers to tag cache entries, preventing severe latency during context switches. Page fusion is explicitly disabled to prevent memory deduplication side-channel vulnerabilities, ensuring that malicious payloads cannot infer the presence of host-level memory structures by measuring write-access latency.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --nested-paging=on --large-pages=on --page-fusion=off
    VBoxManage modifyvm "NODE-A-VM-NAME" --x86-vtx-vpid=on

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Nested Paging|Large Pages|Page Fusion|VPID)"

**Optimal Output:**

    Nested Paging:               enabled
    Large Pages:                 enabled
    VT-x VPID:                   enabled
    Page Fusion:                 disabled

## 5. Microarchitectural Buffer Optimization
The command forces the hypervisor to scrub processor caches and execute indirect branch predictor barriers during context switches. Activating these hardware defenses isolates the host processor from transient execution manipulation. The mitigations prevent malicious code from exploiting speculative execution vulnerabilities to read privileged memory pages belonging to the underlying Linux host. Flushing the Level 1 Data Cache strictly confines guest operations, ensuring that cryptographic keys or host telemetry routing data remain completely inaccessible to the payload running inside the boundary.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --spec-ctrl=on --l1d-flush-on-vm-entry=on --mds-clear-on-vm-entry=on --ibpb-on-vm-entry=on --ibpb-on-vm-exit=on

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" --machinereadable | grep -E "(spec-ctrl|l1d-flush|mds-clear|ibpb)"

**Optimal Output:**

    ibpb-on-vm-exit="on"
    ibpb-on-vm-entry="on"
    spec-ctrl="on"
    l1d-flush-on-vm-entry="on"
    mds-clear-on-vm-entry="on"

## 6. Time Desynchronization and Epoch Spoofing
The command severs the virtual machine from the physical host clock, freezing chronological telemetry by disabling host-time synchronization. A negative offset rewinds the chronological epoch by thirty days. This synthetic chronometry subverts malicious payloads relying on time-bomb logic or latency measurements to detect the analysis platform. The multi-core physical execution binding has been omitted, preserving synchronized timing across virtual processors and preventing catastrophic Desktop Window Manager lockups within the guest kernel.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --hpet=on
    VBoxManage setextradata "NODE-A-VM-NAME" "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled" 1
    VBoxManage modifyvm "NODE-A-VM-NAME" --biossystemtimeoffset -2592000000

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "HPET"
    VBoxManage getextradata "NODE-A-VM-NAME" enumerate | grep -E "(GetHostTimeDisabled)"

**Optimal Output:**

    HPET:                        enabled
    Key: VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled, Value: 1

## 7. Execution Architecture and Core Allocation
The command assigns four virtual processing cores to the engine and permits them to utilize their full capacity. Providing sufficient processing power prevents the guest scheduler from dropping threads under heavy analysis loads. By matching the core count of standard commercial consumer hardware, the configuration actively deceives sandbox-evasion routines embedded within malicious payloads, bypassing heuristic checks that refuse execution on single-core or dual-core analysis environments.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --cpus=4 --cpu-execution-cap=100

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Number of CPUs|CPU exec cap)"

**Optimal Output:**

    Number of CPUs:              4
    CPU exec cap:                100%

## 8. Legacy Hardware and USB Disconnection
The system reverts the pointing device to a standard PS/2 mouse to allow the complete detachment of physical universal serial bus protocols. The engine removes legacy communication ports and eliminates the emulated audio backend using standard virtual driver deactivation. Disabling these interfaces removes extensive emulation code, permanently blinding exploit vectors that target virtualized descriptor parsing and hardware translation. Utilizing the modernized parameters ensures operational execution within the updated VirtualBox 7.x syntax framework.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --mouse=ps2
    VBoxManage modifyvm "NODE-A-VM-NAME" --usb=off --usbehci=off --usbxhci=off
    VBoxManage modifyvm "NODE-A-VM-NAME" --audio-driver=none --audio-controller=none
    VBoxManage modifyvm "NODE-A-VM-NAME" --uart1=off --lpt1=off

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Pointing Device|USB|EHCI|XHCI|Audio|UART|LPT)"

**Optimal Output:**

    Pointing Device:             PS/2 Mouse
    USB:                         disabled
    EHCI:                        disabled
    XHCI:                        disabled
    Audio:                       disabled (Driver: None, Controller: None, Codec: Unknown)
    UART 1:                      disabled
    LPT 1:                       disabled

## 9. IOMMU and Hardware Translation Lockdown
The command explicitly disables the emulated hardware translation layer. Because the architecture relies on paravirtualized network drivers instead of passing physical hardware directly into the virtual machine, this translation layer is unnecessary. Keeping it turned off removes complex software from the execution path, eliminating the risk of out-of-bounds memory vulnerabilities where an attacker could theoretically compromise the input/output memory management unit translation tables.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --iommu=none

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "IOMMU"

**Optimal Output:**

    IOMMU:                       None

## 10. Graphics Controller and Hardware Acceleration
The command configures the VirtualBox SVGA graphics controller and strictly enforces three-dimensional hardware acceleration. This modification prevents the Windows 11 Desktop Window Manager from initiating a crash loop by satisfying its Direct3D hardware dependency. To mitigate the expansion of the attack surface toward the host GPU, the video memory is strictly clamped to a minimal operational threshold of 128MB, preventing extensive shader payloads from fully mapping onto the physical rendering engine.

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
The command terminates all communication channels by disabling the shared clipboard and file transfers. It shuts down diagnostic recording and memory tracing systems. Closing data sockets prevents malicious software from interacting with the host clipboard, reading copied telemetry, or exploiting memory-mapped tracing buffers. The diagnostic output relies on specific VirtualBox 7.x phrasing, appending the word mode to the clipboard attributes, neutralizing the potential for guest-to-host data leakage.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --clipboard-mode=disabled --clipboard-file-transfers=off --drag-and-drop=disabled
    VBoxManage modifyvm "NODE-A-VM-NAME" --recording=off --tracing-enabled=off

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Clipboard Mode|Drag'n'drop Mode|Recording|Tracing)"

**Optimal Output:**

    Clipboard Mode:              disabled
    Drag'n'drop Mode:            disabled
    Recording:                   disabled
    Tracing Enabled:             disabled

## 12. Firmware Integrity and UEFI Secure Boot
The command initializes an extensible firmware interface and dynamically injects platform keys and signature databases into the virtual motherboard. Enabling secure boot via the modifynvram subcommand forces the firmware to cryptographically verify the digital signature of the operating system bootloader against the injected certificates. This prevents low-level bootkit malware from modifying the initialization sequence and hiding persistence mechanisms beneath the operating system kernel execution layer.

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
The command generates a software-based trusted platform module within the memory space of the hypervisor, completely severing the guest from the physical cryptographic hardware while satisfying the Windows 11 hardware requirement. Setting the virtualization provider to match hypervisor standards instructs the kernel on proper interrupt routing, ensuring the system remains stable and responsive. This avoids kernel panics associated with unsupported advanced programmable interrupt controller translation requests.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --tpm-type=2.0
    VBoxManage modifyvm "NODE-A-VM-NAME" --paravirt-provider=hyperv

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(TPM|Paravirt)"

**Optimal Output:**

    TPM Type:                    2.0
    Paravirt. Provider:          Hyper-V
    Effective Paravirt. Prov.:   Hyper-V

## 14. Teleportation, VRDE, and Memory Ballooning
The command uses proper syntax to disable remote display endpoints and lock the dynamic memory allocation engine. Disabling the remote server closes interaction vectors where an external connection could manipulate the graphical display state. Condensing the memory balloon parameter into the guest-memory-balloon string prevents syntax rejection, ensuring malicious programs cannot manipulate the guest integration features to force the hypervisor to exhaust physical RAM on the Linux host.

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
The command blocks the guest operating system from accessing underlying processor virtualization instructions. This restriction stops malicious software from attempting to build internal hypervisors inside the sandbox to intercept system calls. Preventing the guest from manipulating control structures ensures the primary containment layer remains intact, so the target operating system cannot hide its internal activities from the external analysis instrumentation.

**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --nested-hw-virt=off

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "Nested VT-x/AMD-V"

**Optimal Output:**

    Nested VT-x/AMD-V:           disabled

## 16. PCIe NVMe Storage, Host I/O Caching, and Disk Attachment
The command adds a storage controller and attaches the virtual disk media. Disabling the host cache forces disk reads and writes to bypass the host Linux kernel cache entirely via direct memory access. This severs the link between guest disk activity and host memory, significantly reducing the impact of storage-based exploits while maximizing input/output operations per second for the rigorous demands of executing payloads.

**Implementation Command:**

    VBoxManage storagectl "NODE-A-VM-NAME" --name "NVMe-Controller" --add=pcie --controller=NVMe --portcount=1 --hostiocache=off
    VBoxManage storageattach "NODE-A-VM-NAME" --storagectl "NVMe-Controller" --port 0 --device 0 --type hdd --medium "NODE-A-VDI-FILENAME"

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "Host I/O Cache"

**Optimal Output:**

    Host I/O Cache:              off

## 17. Shared Folder Removal
The command systematically attempts to delete permanent or temporary shared folders that might exist between the host and the virtual machine. It runs silently, ignoring errors if no folders are found. Removing shared directories is a mandatory security step that destroys direct file-system bridges, preventing malicious software from escaping the execution engine by iterating through mounted volume paths and writing destructive payloads to the host.

**Implementation Command:**

    VBoxManage sharedfolder remove "NODE-A-VM-NAME" --name "NODE-A-SHARED-FOLDER-NAME" 2>/dev/null || true
    VBoxManage sharedfolder remove "NODE-A-VM-NAME" --name "NODE-A-SHARED-FOLDER-NAME" --transient 2>/dev/null || true

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "Shared folders" -A 2

**Optimal Output:**

    Shared folders:              <none>

## 18. Immutable Disk State with Auto-Reset
The command locks the primary virtual disk into a read-only state, generating a differencing disk overlay within a temporary file. Every registry modification, dropped executable, and systemic change is captured exclusively within this transient overlay. Upon shutdown, the engine destroys the overlay completely, purging all forensic artifacts and permanently returning the environment to a pristine, untainted baseline for subsequent detonation cycles.

**Implementation Command:**

    VBoxManage modifymedium "NODE-A-VDI-FILENAME" --type immutable --autoreset=on

**Diagnostic Command:**

    VBoxManage showmediuminfo "NODE-A-VDI-FILENAME" | grep -E -i "(Type|Auto-Reset)"

**Optimal Output:**

    Type:                        immutable
    Auto-Reset:                  on