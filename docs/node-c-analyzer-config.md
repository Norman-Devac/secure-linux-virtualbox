# Oracle VM VirtualBox Analyst Control Plane Configuration (Node C)

This document dictates the configuration of the analytical control plane responsible for out-of-band deep packet inspection and intelligence indexing. This node must absorb an extreme volume of decrypted data without collapsing under computational pressure, applying a strict defensive envelope while implementing highly threaded memory mapping mechanisms.

## 1. Control Plane Network and Hardware Allocation
The interface attaches directly to the closed control switch in an unrestricted promiscuous mode, allowing the virtual network interface card to accept all mirrored packets and TCP telemetry streams. Allocating four processing cores paired with eight gigabytes of physical RAM accommodates the multi-threaded architecture. The paravirtualized VirtIO driver translates memory rings efficiently to neutralize packet queue depletion.

**Implementation Command:**

    VBoxManage modifyvm "NODE-C-VM-NAME" --nic1=intnet --intnet1="VBOX-NETWORK-CONTROL" --nic-type1=virtio --nic-promisc1=allow-all
    VBoxManage modifyvm "NODE-C-VM-NAME" --cpus=4 --cpu-execution-cap=100 --memory=8192
    VBoxManage modifyvm "NODE-C-VM-NAME" --nested-paging=on --large-pages=on --page-fusion=off

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-C-VM-NAME" | grep -i -E "(NIC 1|Number of CPUs|Memory size)"

**Optimal Output:**

    Memory size: 8192MB
    Number of CPUs: 4
    NIC 1: MAC: 080027XXXXZZ, Attachment: Internal Network 'VBOX-NETWORK-CONTROL', Cable connected: on, Trace: off (file: none), Type: virtio, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: allow-all, Bandwidth group: none

## 2. High-Throughput Forensic Storage
Assigning a dedicated PCIe NVMe storage controller with host input/output caching disabled forces the virtual machine to manage its block storage independently. Evading the physical host cache ensures rapid commits of metadata and prevents kernel ring buffers from overflowing during periods of extreme saturation.

**Implementation Command:**

    VBoxManage storagectl "NODE-C-VM-NAME" --name "NVMe-Telemetry" --add=pcie --controller=NVMe --portcount=1 --hostiocache=off
    VBoxManage storageattach "NODE-C-VM-NAME" --storagectl "NVMe-Telemetry" --port 0 --device 0 --type hdd --medium "NODE-C-VDI-FILENAME"

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-C-VM-NAME" | grep -i "NVMe-Telemetry" -A 2

**Optimal Output:**

    Storage Controller Name (0):            NVMe-Telemetry
    Storage Controller Type (0):            NVMe
    Storage Controller Instance Number (0): 0

## 3. Attack Surface Reduction, Firmware Integrity, and Microarchitectural Mitigation
The hypervisor nullifies peripheral endpoints to close auxiliary communication vectors. The mitigation adjustment replaces strict caching protocols with the entry-state flushing routine, isolating the host processor from speculative execution exploitation while preserving multi-core indexing performance.

**Implementation Command:**

    VBoxManage modifyvm "NODE-C-VM-NAME" --audio-driver=none --audio-controller=none --usb=off --usbehci=off --usbxhci=off
    VBoxManage modifyvm "NODE-C-VM-NAME" --clipboard-mode=disabled --drag-and-drop=disabled --vrde=off
    VBoxManage modifyvm "NODE-C-VM-NAME" --spec-ctrl=on --l1d-flush-on-vm-entry=on --mds-clear-on-vm-entry=on --ibpb-on-vm-entry=on --ibpb-on-vm-exit=off
    VBoxManage modifyvm "NODE-C-VM-NAME" --firmware=efi64
    VBoxManage modifynvram "NODE-C-VM-NAME" inituefivarstore
    VBoxManage modifynvram "NODE-C-VM-NAME" enrollorclpk
    VBoxManage modifynvram "NODE-C-VM-NAME" secureboot --enable

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-C-VM-NAME" | grep -E -i "(Clipboard Mode|USB:|Audio:|Secure Boot|mds-clear|l1d-flush|ibpb)"

**Optimal Output:**

    Clipboard Mode: disabled
    USB: disabled
    Audio: disabled (Driver: None, Controller: None, Codec: Unknown)
    Secure Boot: enabled
    mds-clear-on-vm-entry="on"
    l1d-flush-on-vm-entry="on"
    ibpb-on-vm-exit="off"

## 4. Zero-Copy Network Intrusion Detection via Suricata
The configuration integrates native af-packet sockets utilizing TPACKET_V3 for signature-based structural intrusion detection. By enforcing symmetric hashing and an expansive ring block size, the engine deterministically allocates incoming connection flows to specific processing cores to eradicate race conditions.

**Implementation Command:**

    sed -i '/^af-packet:/,/^  - interface:/{s/^  - interface:.*/  - interface: NODE-C-LINUX-INTERFACE\n    cluster-id: 99\n    cluster-type: cluster_qm\n    defrag: yes\n    use-mmap: yes\n    tpacket-v3: yes\n    ring-size: 200000/}' /etc/suricata/suricata.yaml
    systemctl restart suricata.service

**Diagnostic Command:**

    grep -A 8 "^af-packet:" /etc/suricata/suricata.yaml

**Optimal Output:**

    af-packet:
    interface: NODE-C-LINUX-INTERFACE
    cluster-id: 99
    cluster-type: cluster_qm
    defrag: yes
    use-mmap: yes
    tpacket-v3: yes
    ring-size: 200000

## 5. Protocol-Agnostic Session Parsing via Zeek
Zeek dynamically parses decrypted application-layer traffic independent of predefined rules, structurally identifying anomalous behavior. Pinning the interface strictly to the internal virtual adapter restricts the observation boundary to valid execution traffic exclusively, isolating the engine from extraneous host signals.

**Implementation Command:**

    sed -i 's/^interface=.*/interface=NODE-C-LINUX-INTERFACE/' /opt/zeek/etc/node.cfg
    /opt/zeek/bin/zeekctl deploy

**Diagnostic Command:**

    /opt/zeek/bin/zeekctl status

**Optimal Output:**

    Name         Type    Host             Status    Pid    Started
    zeek         standalone localhost        running   11234  System Initialization

## 6. PCAP-over-IP Ingestion and Arkime Integration
Arkime is configured to ingest directly from PolarProxy's real-time network stream, bypassing disk-read latency completely. Using the pcap-over-ip-server reading method, the system connects to the socket on port 57012, parsing payloads in memory and committing telemetry directly to the backend matrix to establish an unalterable execution timeline.

**Implementation Command:**

    cat << 'EOF' > /opt/arkime/etc/config.ini
    [default]
    interface=NODE-C-LINUX-INTERFACE
    pcapReadMethod=pcap-over-ip-server
    pcapoverip=NODE-B-STATIC-IP:57012
    elasticsearch=NODE-C-ELASTICSEARCH-URL
    EOF
    systemctl restart arkimecapture.service

**Diagnostic Command:**

    systemctl status arkimecapture.service | grep -i "pcapoverip"

**Optimal Output:**

    CGroup: /system.slice/arkimecapture.service
    └─12456 /opt/arkime/bin/capture -c /opt/arkime/etc/config.ini --pcapoverip