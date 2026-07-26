# Oracle VM VirtualBox 7.2.14 Analyst Control Plane Configuration (Node C)

This document outlines the technical configuration for the out-of-band telemetry node operating on a Fedora 44 host. The system functions as a forensic control plane, ingesting decrypted packet streams, executing deep packet inspection, and extracting behavioral indicators without introducing artifacts into the detonation chamber. Hardware side-channel buffers have been extensively deployed to protect the host architecture from exploitation via the advanced analytical engines parsing hostile data.

## 1. Control Plane Network and Hardware Allocation
The telemetry node requires a single paravirtualized network interface connected exclusively to the telemetry switch. The engine assigns four physical processing cores to manage the heavily multi-threaded load of network security monitoring and signature matching. The virtual adapter operates in an unrestricted promiscuous mode to accurately mirror all Ethernet frames traversing the virtual switch without generating transmission requests.

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
The software provisions a dedicated Non-Volatile Memory Express storage controller to handle the rapid generation of indexed event logs, intrusion detection alerts, and massive packet captures. The direct memory access configuration guarantees that continuous disk writes bypass the hypervisor bottlenecks, ensuring that kernel ring buffers do not overflow and drop essential telemetry during periods of extreme network saturation.

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
The control plane environment limits host interactions by disabling auxiliary emulation modules. Syntax is employed to permanently disable the USB handlers and the audio translation buffers. Critical speculative execution barriers are rigidly enforced to ensure that the ingestion of hostile packet captures through analytical software cannot maliciously exploit the host processor via speculative execution. Secure Boot is activated to lock the initialization chain, defending the forensic toolkit from tampering.

**Implementation Command:**

    VBoxManage modifyvm "NODE-C-VM-NAME" --audio-driver=none --audio-controller=none --usb=off --usbehci=off --usbxhci=off
    VBoxManage modifyvm "NODE-C-VM-NAME" --clipboard-mode=disabled --drag-and-drop=disabled --vrde=off
    VBoxManage modifyvm "NODE-C-VM-NAME" --spec-ctrl=on --l1d-flush-on-vm-entry=on --mds-clear-on-vm-entry=on --ibpb-on-vm-entry=on --ibpb-on-vm-exit=on
    VBoxManage modifyvm "NODE-C-VM-NAME" --firmware=efi64
    VBoxManage modifynvram "NODE-C-VM-NAME" inituefivarstore
    VBoxManage modifynvram "NODE-C-VM-NAME" enrollorclpk
    VBoxManage modifynvram "NODE-C-VM-NAME" secureboot --enable

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-C-VM-NAME" | grep -E -i "(Clipboard Mode|USB:|Audio:|Secure Boot|mds-clear)"

**Optimal Output:**

    Clipboard Mode: disabled
    USB: disabled
    Audio: disabled (Driver: None, Controller: None, Codec: Unknown)
    Secure Boot: enabled
    mds-clear-on-vm-entry="on"

## 4. Zero-Copy Network Intrusion Detection via Suricata
The system utilizes Suricata for signature-based network intrusion detection. To survive intense throughput without dropping frames, the software integrates native kernel sockets utilizing memory mapping combined with TPACKET_V3 architecture and expanded ring boundaries. Symmetric hashing routes packet flows deterministically to eliminate multi-threading race conditions and guarantee high-throughput packet inspection without overwhelming user-space packet ingestion algorithms.

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
Zeek dynamically parses decrypted application-layer traffic independent of predefined rules, relying instead on protocol analysis frameworks. The engine identifies anomalous command structures and custom binary protocols heavily utilized by advanced persistent threats, structurally separating the network data into distinct, queryable telemetry files that allow immediate forensic extraction and event correlation.

**Implementation Command:**

    sed -i 's/^interface=.*/interface=NODE-C-LINUX-INTERFACE/' /opt/zeek/etc/node.cfg
    /opt/zeek/bin/zeekctl deploy

**Diagnostic Command:**

    /opt/zeek/bin/zeekctl status

**Optimal Output:**

    Name         Type    Host             Status    Pid    Started
    zeek         standalone localhost        running   11234  System Initialization

## 6. PCAP-over-IP Ingestion and Arkime Integration
Arkime ingests the continuous stream of decrypted transport layer security packets pulled across the internal network boundary via the dedicated PCAP-over-IP listener operating in conjunction with PolarProxy. Utilizing the pcap-over-ip-server reading method, the system parses the payloads in memory, indexes the metadata into the backend elastic matrix, and writes the decrypted binaries directly to the local NVMe storage hardware, providing an exact, unalterable execution timeline of the detonation phase.

**Implementation Command:**

    cat << 'EOF' > /opt/arkime/etc/config.ini
    [default]
    interface=NODE-C-LINUX-INTERFACE
    pcapReadMethod=pcap-over-ip-server
    pcapoverip=NODE-B-STATIC-IP:NODE-C-PCAP-PORT
    elasticsearch=NODE-C-ELASTICSEARCH-URL
    EOF
    systemctl restart arkimecapture.service

**Diagnostic Command:**

    systemctl status arkimecapture.service | grep -i "pcapoverip"

**Optimal Output:**

    CGroup: /system.slice/arkimecapture.service
    └─12456 /opt/arkime/bin/capture -c /opt/arkime/etc/config.ini --pcapoverip