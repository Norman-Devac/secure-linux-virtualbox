**Implementation Command:**

    VBoxManage modifyvm "NODE-A-VM-NAME" --nic1=intnet --intnet1="VBOX-NETWORK-SANDBOX" --nic-type1=virtio --nic-promisc1=deny && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --nic2=none --nic3=none --nic4=none --nic5=none --nic6=none --nic7=none --nic8=none && \
    { VBoxManage bandwidthctl "NODE-A-VM-NAME" add "NODE-A-BANDWIDTH-GROUP" --type network --limit=10M 2>/dev/null || true; } && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --nic-bandwidth-group1="NODE-A-BANDWIDTH-GROUP" && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --boot1=disk --boot2=none --boot3=none --boot4=none && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --nested-paging=on --large-pages=on --page-fusion=off && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --x86-vtx-vpid=on && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --spec-ctrl=on --l1d-flush-on-vm-entry=on --mds-clear-on-vm-entry=on --ibpb-on-vm-entry=on --ibpb-on-vm-exit=on && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --hpet=on && \
    VBoxManage setextradata "NODE-A-VM-NAME" "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled" 1 && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --biossystemtimeoffset -2592000000 && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --cpus=4 --cpu-execution-cap=100 && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --mouse=ps2 && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --usb=off --usbehci=off --usbxhci=off && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --audio-driver=none --audio-controller=none && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --uart1=off --lpt1=off && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --iommu=none && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --graphicscontroller=vboxsvga --accelerate-3d=on --vram=128 && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --default-frontend=gui && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --clipboard-mode=disabled --clipboard-file-transfers=off --drag-and-drop=disabled && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --recording=off --tracing-enabled=off && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --firmware=efi64 && \
    VBoxManage modifynvram "NODE-A-VM-NAME" inituefivarstore && \
    VBoxManage modifynvram "NODE-A-VM-NAME" enrollorclpk && \
    VBoxManage modifynvram "NODE-A-VM-NAME" enrollmssignatures && \
    VBoxManage modifynvram "NODE-A-VM-NAME" secureboot --enable && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --bios-boot-menu=disabled --bios-logo-display-time=0 && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --tpm-type=2.0 && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --paravirt-provider=hyperv && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --teleporter=off --vrde=off && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --guest-memory-balloon=0 && \
    VBoxManage modifyvm "NODE-A-VM-NAME" --nested-hw-virt=off && \
    VBoxManage storagectl "NODE-A-VM-NAME" --name "NVMe-Controller" --add=pcie --controller=NVMe --portcount=1 --hostiocache=off && \
    VBoxManage storageattach "NODE-A-VM-NAME" --storagectl "NVMe-Controller" --port 0 --device 0 --type hdd --medium "NODE-A-VDI-FILENAME" && \
    { VBoxManage sharedfolder remove "NODE-A-VM-NAME" --name "NODE-A-SHARED-FOLDER-NAME" 2>/dev/null || true; } && \
    { VBoxManage sharedfolder remove "NODE-A-VM-NAME" --name "NODE-A-SHARED-FOLDER-NAME" --transient 2>/dev/null || true; } && \
    VBoxManage modifymedium "NODE-A-VDI-FILENAME" --type immutable --autoreset=on

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "NIC 1" ; \
    VBoxManage bandwidthctl "NODE-A-VM-NAME" list ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "Boot Device" ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Nested Paging|Large Pages|Page Fusion|VPID)" ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" --machinereadable | grep -E "(spec-ctrl|l1d-flush|mds-clear|ibpb)" ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "HPET" ; \
    VBoxManage getextradata "NODE-A-VM-NAME" enumerate | grep -E "(GetHostTimeDisabled)" ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Number of CPUs|CPU exec cap)" ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Pointing Device|USB|EHCI|XHCI|Audio|UART|LPT)" ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "IOMMU" ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Graphics Controller|VRAM size|3D Acceleration|Default Frontend)" ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Clipboard Mode|Drag'n'drop Mode|Recording|Tracing)" ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Firmware|Secure Boot)" ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(TPM|Paravirt)" ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -E -i "(Teleporter|VRDE|balloon)" ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "Nested VT-x/AMD-V" ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "Host I/O Cache" ; \
    VBoxManage showvminfo "NODE-A-VM-NAME" | grep -i "Shared folders" -A 2 ; \
    VBoxManage showmediuminfo "NODE-A-VDI-FILENAME" | grep -E -i "(Type|Auto-Reset)"

**Optimal Output:**

    NIC 1: MAC: 080027XXXXXX, Attachment: Internal Network 'VBOX-NETWORK-SANDBOX', Cable connected: on, Trace: off (file: none), Type: virtio, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: NODE-A-BANDWIDTH-GROUP
    Name: 'NODE-A-BANDWIDTH-GROUP', Type: network, Limit: 10 MBytes/sec
    Boot Device 1: HardDisk
    Boot Device 2: Not Assigned
    Boot Device 3: Not Assigned
    Boot Device 4: Not Assigned
    Nested Paging: enabled
    Large Pages: enabled
    VT-x VPID: enabled
    Page Fusion: disabled
    ibpb-on-vm-exit="on"
    ibpb-on-vm-entry="on"
    spec-ctrl="on"
    l1d-flush-on-vm-entry="on"
    mds-clear-on-vm-entry="on"
    HPET: enabled
    Key: VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled, Value: 1
    Number of CPUs: 4
    CPU exec cap: 100%
    Pointing Device: PS/2 Mouse
    USB: disabled
    EHCI: disabled
    XHCI: disabled
    Audio: disabled (Driver: None, Controller: None, Codec: Unknown)
    UART 1: disabled
    LPT 1: disabled
    IOMMU: None
    Graphics Controller: VBoxSVGA
    VRAM size: 128MB
    3D Acceleration: enabled
    Default Frontend: gui
    Clipboard Mode: disabled
    Drag'n'drop Mode: disabled
    Recording: disabled
    Tracing Enabled: disabled
    Firmware: EFI64
    Secure Boot: enabled
    TPM Type: 2.0
    Paravirt. Provider: Hyper-V
    Effective Paravirt. Prov.: Hyper-V
    VRDE: disabled
    Teleporter Enabled: disabled
    Guest memory balloon size: 0 Megabytes
    Nested VT-x/AMD-V: disabled
    Host I/O Cache: off
    Shared folders: <none>
    Type: immutable
    Auto-Reset: on