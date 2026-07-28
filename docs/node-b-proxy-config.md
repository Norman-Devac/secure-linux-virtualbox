# Oracle VM VirtualBox Synthetic Gateway Configuration (Node B)

This document outlines the technical configuration for the dual-homed proxy node. The architectural mandate requires the seamless ingestion of hostile traffic from the detonation environment without introducing processing latency. By synchronizing the microarchitectural isolation parameters with the primary sandbox and integrating precise transport layer interception routing, the configuration sustains high-throughput interception dynamically.

## 1. Dual-Homed Network Provisioning and VirtIO Encapsulation
Connecting the primary interface to the sandbox switch in promiscuous mode forces the proxy to ingest all Ethernet frames. The secondary interface connects to the sterile control network. The targeted modification utilizes the VirtIO paravirtualized driver to establish an efficient memory-mapped ring buffer, bypassing legacy hardware emulation and queue exhaustion.

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
The architecture utilizes two physical processing cores capped at maximal capacity to handle cryptographic workloads, supported by an explicit memory allocation of 2048 Megabytes. Modifying the Level 1 Data Cache mitigation to execute solely upon virtual machine entry ensures the proxy virtual machine survives the immense interrupt volume associated with real-time packet processing. The indirect branch predictor barrier on exit is explicitly disabled to preserve operational fluidity.

**Implementation Command:**

    VBoxManage modifyvm "NODE-B-VM-NAME" --cpus=2 --cpu-execution-cap=100 --memory=2048
    VBoxManage modifyvm "NODE-B-VM-NAME" --nested-paging=on --large-pages=on --page-fusion=off
    VBoxManage modifyvm "NODE-B-VM-NAME" --spec-ctrl=on --l1d-flush-on-vm-entry=on --mds-clear-on-vm-entry=on --ibpb-on-vm-entry=on --ibpb-on-vm-exit=off

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-B-VM-NAME" --machinereadable | grep -E "(cpus=|memory=|nestedpaging|l1d-flush|ibpb)"

**Optimal Output:**

    cpus=2
    memory=2048
    nestedpaging="on"
    l1d-flush-on-vm-entry="on"
    ibpb-on-vm-entry="on"
    ibpb-on-vm-exit="off"

## 3. High-Throughput NVMe Storage Configuration
The virtualized Non-Volatile Memory Express controller utilizes direct memory access. Forcefully disabling host-level caching ensures that high-frequency log writes bypass the host file system buffers, protecting physical host memory from entering a starved state during volumetric network assaults.

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
Clipboard transfers and audio mapping are explicitly disabled to seal structural vulnerabilities. Initiating Secure Boot strictly prevents persistent exploitation of the proxy itself, ensuring that parsed binary exploits cannot embed persistent rootkits into the startup sequence.

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
Executing localized deception requires detouring external routing logic seamlessly. The architecture resolves translation latency by compiling translation rules directly into the kernel's prerouting hooks using nftables. Defining a strict drop policy on the forward chain guarantees that outbound traffic targeting non-standard ports cannot bypass the NAT interception and reach the control plane.

**Implementation Command:**

    nft add table ip nat
    nft add chain ip nat prerouting '{ type nat hook prerouting priority dstnat; policy accept; }'
    nft add rule ip nat prerouting iifname "NODE-B-LINUX-INTERFACE" tcp dport 443 redirect to :10443
    nft add rule ip nat prerouting iifname "NODE-B-LINUX-INTERFACE" tcp dport 80 redirect to :80
    nft add rule ip nat prerouting iifname "NODE-B-LINUX-INTERFACE" udp dport 53 redirect to :53
    nft add table ip filter
    nft add chain ip filter forward '{ type filter hook forward priority 0; policy drop; }'

**Diagnostic Command:**

    nft list table ip nat ; nft list table ip filter

**Optimal Output:**

    table ip nat {
    chain prerouting {
    type nat hook prerouting priority dstnat; policy accept;
    iifname "NODE-B-LINUX-INTERFACE" tcp dport 443 redirect to :10443
    iifname "NODE-B-LINUX-INTERFACE" tcp dport 80 redirect to :80
    iifname "NODE-B-LINUX-INTERFACE" udp dport 53 redirect to :53
    }
    }
    table ip filter {
    chain forward {
    type filter hook forward priority 0; policy drop;
    }
    }

## 6. Dynamic Cryptographic Interception via PolarProxy
PolarProxy executes transparent transport layer security interception. The engine utilizes the nosni flag to intercept malformed requests lacking valid server name indications. Bypassing massive disk input/output bottlenecks, the system streams the decrypted plaintext traffic over a PCAP-over-IP socket (port 57012) directly to the analytical control plane.

**Implementation Command:**

    cat << 'EOF' > /etc/systemd/system/polarproxy.service
    [Unit]
    Description=PolarProxy TLS Interception Service
    After=network.target

    [Service]
    Type=simple
    User=NODE-B-LINUX-USER
    ExecStart=/opt/PolarProxy/PolarProxy -v -p 10443,80,443 -x /var/log/PolarProxy/polarproxy.cer -f /var/log/PolarProxy/proxyflows.log -o /var/log/PolarProxy/ --terminate --nosni NODE-B-TARGET-DOMAIN --pcapoverip 57012
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
INetSim structurally simulates the required daemons, delivering positive HTTP status codes and forging dynamic DNS resolutions. Disabling the HTTPS module allows PolarProxy absolute control over port 443 termination, resolving port-binding collisions while presenting a perfectly deceptive execution environment.

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