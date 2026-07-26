# Oracle VM VirtualBox 7.2.14 Synthetic Gateway Configuration (Node B)

This document outlines the technical configuration for the gateway virtual machine operating on a Fedora 44 host. The system functions as an internet simulator designed to provide deceptive network responses, intercept command-and-control traffic, and emulate high-fidelity services without allowing external egress. Hardware vulnerability buffers and secure boot mechanisms have been fully injected to match the strict hypervisor isolation parameters defined in the primary detonation node.

## 1. Dual-Homed Network Provisioning and VirtIO Encapsulation
The network architecture provisions two independent, paravirtualized network interfaces bound to distinct internal host-memory switches. The primary interface attaches to the detonation network, receiving hostile traffic from the Windows 11 sandbox. The secondary interface connects to the isolated control network. The engine utilizes the VirtIO paravirtualized driver to establish a shared-memory ring buffer, bypassing hardware emulation overhead, eliminating buffer overflows, and minimizing latency anomalies that could alert heuristic evasion checks.

**Implementation Command:**

    VBoxManage modifyvm "NODE-B-VM-NAME" --nic1=intnet --intnet1="VBOX-NETWORK-SANDBOX" --nic-type1=virtio --nic-promisc1=allow-all
    VBoxManage modifyvm "NODE-B-VM-NAME" --nic2=intnet --intnet2="VBOX-NETWORK-CONTROL" --nic-type2=virtio --nic-promisc2=deny
    VBoxManage modifyvm "NODE-B-VM-NAME" --nic3=none --nic4=none --nic5=none --nic6=none --nic7=none --nic8=none

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-B-VM-NAME" | grep -i -E "NIC 1|NIC 2"

**Optimal Output:**

    NIC 1: MAC: 080027XXXXXX, Attachment: Internal Network 'VBOX-NETWORK-SANDBOX', Cable connected: on, Trace: off (file: none), Type: virtio, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: allow-all, Bandwidth group: none
    NIC 2: MAC: 080027XXXXYY, Attachment: Internal Network 'VBOX-NETWORK-CONTROL', Cable connected: on, Trace: off (file: none), Type: virtio, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none

## 2. Execution Architecture and Microarchitectural Concealment
The engine allocates two physical processing cores, capped at maximum utilization to ensure real-time packet processing without introducing queue latency. The software aggressively flushes the Level 1 Data Cache and clears Microarchitectural Data Sampling routines upon context switches. Indirect branch predictor barriers are explicitly enforced to isolate the Fedora 44 host processor from speculative execution attacks potentially originating within compromised gateway software modules processing malformed packet payloads.

**Implementation Command:**

    VBoxManage modifyvm "NODE-B-VM-NAME" --cpus=2 --cpu-execution-cap=100
    VBoxManage modifyvm "NODE-B-VM-NAME" --nested-paging=on --large-pages=on --page-fusion=off
    VBoxManage modifyvm "NODE-B-VM-NAME" --spec-ctrl=on --l1d-flush-on-vm-entry=on --mds-clear-on-vm-entry=on --ibpb-on-vm-entry=on --ibpb-on-vm-exit=on

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-B-VM-NAME" --machinereadable | grep -E "(cpus=|nestedpaging|l1d-flush|ibpb)"

**Optimal Output:**

    cpus=2
    nestedpaging="on"
    l1d-flush-on-vm-entry="on"
    ibpb-on-vm-entry="on"

## 3. High-Throughput NVMe Storage Configuration
The architecture implements a virtualized Non-Volatile Memory Express controller operating over a Peripheral Component Interconnect Express bus. Disabling host caching enforces direct memory access, forcing the virtual input/output requests to bypass the host file system cache. This strictly prevents hostile, high-volume network traffic ingested by the gateway from inducing physical memory pressure or starving the host Linux kernel of necessary block caching resources.

**Implementation Command:**

    VBoxManage storagectl "NODE-B-VM-NAME" --name "NVMe-Controller" --add=pcie --controller=NVMe --portcount=1 --hostiocache=off
    VBoxManage storageattach "NODE-B-VM-NAME" --storagectl "NVMe-Controller" --port 0 --device 0 --type hdd --medium "NODE-B-VDI-FILENAME"

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-B-VM-NAME" | grep -i "NVMe-Controller" -A 2

**Optimal Output:**

    Storage Controller Name (0):            NVMe-Controller
    Storage Controller Type (0):            NVMe
    Storage Controller Instance Number (0): 0

## 4. Telemetry Severance, Virtual Device Lockdown, and Secure Boot
The system removes interactive telemetry, clipboard sharing mechanisms, and remote display virtual endpoints, locking down peripheral interaction. The engine disables the emulated audio backend and utilizes configuration flags to permanently nullify universal serial bus controllers. Secure boot enforcement ensures that the gateway Linux operating system cannot be structurally compromised by a persistent bootkit leveraging parsed binary exploits over the detonation network.

**Implementation Command:**

    VBoxManage modifyvm "NODE-B-VM-NAME" --clipboard-mode=disabled --clipboard-file-transfers=off --drag-and-drop=disabled
    VBoxManage modifyvm "NODE-B-VM-NAME" --teleporter=off --vrde=off
    VBoxManage modifyvm "NODE-B-VM-NAME" --usb=off --usbehci=off --usbxhci=off --audio-driver=none --audio-controller=none
    VBoxManage modifyvm "NODE-B-VM-NAME" --firmware=efi64
    VBoxManage modifynvram "NODE-B-VM-NAME" inituefivarstore
    VBoxManage modifynvram "NODE-B-VM-NAME" enrollorclpk
    VBoxManage modifynvram "NODE-B-VM-NAME" secureboot --enable

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-B-VM-NAME" | grep -E -i "(Clipboard Mode|Teleporter|USB:|Audio:|Secure Boot)"

**Optimal Output:**

    Clipboard Mode: disabled
    USB: disabled
    Audio: disabled (Driver: None, Controller: None, Codec: Unknown)
    Teleporter Enabled: disabled
    Secure Boot: enabled

## 5. Kernel-Level Packet Redirection via Nftables
The guest operating system utilizes nftables to process network hooks directly in the kernel space, bypassing user-space routing latency. The configuration establishes a prerouting hook within the translation table. The system deterministically forces incoming traffic attempting to reach the external internet on standard web ports into localized listening daemons, trapping the payload within a closed, synthetic networking loop that perfectly simulates external resolution.

**Implementation Command:**

    nft add table ip nat
    nft add chain ip nat prerouting '{ type nat hook prerouting priority dstnat; policy accept; }'
    nft add rule ip nat prerouting iifname "NODE-B-LINUX-INTERFACE" tcp dport 443 redirect to :10443
    nft add rule ip nat prerouting iifname "NODE-B-LINUX-INTERFACE" tcp dport 80 redirect to :80
    nft add rule ip nat prerouting iifname "NODE-B-LINUX-INTERFACE" udp dport 53 redirect to :53

**Diagnostic Command:**

    nft list table ip nat

**Optimal Output:**

    table ip nat {
    chain prerouting {
    type nat hook prerouting priority dstnat; policy accept;
    iifname "NODE-B-LINUX-INTERFACE" tcp dport 443 redirect to :10443
    iifname "NODE-B-LINUX-INTERFACE" tcp dport 80 redirect to :80
    iifname "NODE-B-LINUX-INTERFACE" udp dport 53 redirect to :53
    }
    }

## 6. Dynamic Cryptographic Interception via PolarProxy
The software deploys PolarProxy to execute transparent, dynamic transport layer security interception. The proxy continuously generates forged certificates to seamlessly subvert encrypted handshakes initiated by the malicious payload. The engine aggressively utilizes the nosni flag to intercept malformed requests lacking valid server name indications, forcing the encrypted connection to complete, and silently streaming the highly sensitive, decrypted plaintext traffic over a PCAP-over-IP socket directly to the analytical control plane without disk latency overhead.

**Implementation Command:**

    cat << 'EOF' > /etc/systemd/system/polarproxy.service
    [Unit]
    Description=PolarProxy TLS Interception Service
    After=network.target

    [Service]
    Type=simple
    User=NODE-B-LINUX-USER
    ExecStart=/opt/PolarProxy/PolarProxy -v -p 10443,80,443 -x /var/log/PolarProxy/polarproxy.cer -f /var/log/PolarProxy/proxyflows.log -o /var/log/PolarProxy/ --terminate --nosni NODE-B-TARGET-DOMAIN --pcapoverip NODE-C-PCAP-PORT
    Restart=always
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target
    EOF
    systemctl daemon-reload && systemctl enable --now polarproxy.service

**Diagnostic Command:**

    systemctl status polarproxy.service | grep -i "Active:"

**Optimal Output:**

    Active: active (running)

## 7. Synthetic Service Emulation via INetSim
INetSim operates a suite of synthetic daemons to provide structurally valid responses for decrypted application-layer traffic. The domain name service resolves requested records to the internal gateway address dynamically. The system delivers successful HTTP status codes and synthetic binaries to deceive the execution environment, forcing the payload to reveal its secondary deployment stages believing it has successfully contacted its remote infrastructure.

**Implementation Command:**

    sed -i 's/^#dns_default_ip.*/dns_default_ip NODE-B-STATIC-IP/' /etc/inetsim/inetsim.conf
    sed -i 's/^#start_service dns/start_service dns/' /etc/inetsim/inetsim.conf
    sed -i 's/^#start_service http/start_service http/' /etc/inetsim/inetsim.conf
    sed -i 's/^start_service https/#start_service https/' /etc/inetsim/inetsim.conf
    systemctl restart inetsim.service

**Diagnostic Command:**

    grep -E "^(dns_default_ip|start_service)" /etc/inetsim/inetsim.conf

**Optimal Output:**

    start_service dns
    start_service http
    dns_default_ip NODE-B-STATIC-IP