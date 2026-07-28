**Implementation Command:**

    VBoxManage modifyvm "NODE-B-VM-NAME" --nic1=intnet --intnet1="VBOX-NETWORK-SANDBOX" --nic-type1=virtio --nic-promisc1=allow-all && \
    VBoxManage modifyvm "NODE-B-VM-NAME" --nic2=intnet --intnet2="VBOX-NETWORK-CONTROL" --nic-type2=virtio --nic-promisc2=deny && \
    VBoxManage modifyvm "NODE-B-VM-NAME" --nic3=none --nic4=none --nic5=none --nic6=none --nic7=none --nic8=none && \
    VBoxManage modifyvm "NODE-B-VM-NAME" --cpus=2 --cpu-execution-cap=100 --memory=2048 && \
    VBoxManage modifyvm "NODE-B-VM-NAME" --nested-paging=on --large-pages=on --page-fusion=off && \
    VBoxManage modifyvm "NODE-B-VM-NAME" --spec-ctrl=on --l1d-flush-on-vm-entry=on --mds-clear-on-vm-entry=on --ibpb-on-vm-entry=on --ibpb-on-vm-exit=off && \
    VBoxManage storagectl "NODE-B-VM-NAME" --name "NVMe-Controller" --add=pcie --controller=NVMe --portcount=1 --hostiocache=off && \
    VBoxManage storageattach "NODE-B-VM-NAME" --storagectl "NVMe-Controller" --port 0 --device 0 --type hdd --medium "NODE-B-VDI-FILENAME" && \
    VBoxManage modifyvm "NODE-B-VM-NAME" --clipboard-mode=disabled --clipboard-file-transfers=off --drag-and-drop=disabled && \
    VBoxManage modifyvm "NODE-B-VM-NAME" --teleporter=off --vrde=off && \
    VBoxManage modifyvm "NODE-B-VM-NAME" --usb=off --usbehci=off --usbxhci=off --audio-driver=none --audio-controller=none && \
    VBoxManage modifyvm "NODE-B-VM-NAME" --firmware=efi64 && \
    VBoxManage modifynvram "NODE-B-VM-NAME" inituefivarstore && \
    VBoxManage modifynvram "NODE-B-VM-NAME" enrollorclpk && \
    VBoxManage modifynvram "NODE-B-VM-NAME" secureboot --enable && \
    nft add table ip nat && \
    nft add chain ip nat prerouting '{ type nat hook prerouting priority dstnat; policy accept; }' && \
    nft add rule ip nat prerouting iifname "NODE-B-LINUX-INTERFACE" tcp dport 443 redirect to :10443 && \
    nft add rule ip nat prerouting iifname "NODE-B-LINUX-INTERFACE" tcp dport 80 redirect to :80 && \
    nft add rule ip nat prerouting iifname "NODE-B-LINUX-INTERFACE" udp dport 53 redirect to :53 && \
    nft add table ip filter && \
    nft add chain ip filter forward '{ type filter hook forward priority 0; policy drop; }' && \
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
    systemctl daemon-reload && \
    systemctl enable --now polarproxy.service && \
    sed -i 's/^#dns_default_ip.*/dns_default_ip NODE-B-STATIC-IP/' /etc/inetsim/inetsim.conf && \
    sed -i 's/^#start_service dns/start_service dns/' /etc/inetsim/inetsim.conf && \
    sed -i 's/^#start_service http/start_service http/' /etc/inetsim/inetsim.conf && \
    sed -i 's/^start_service https/#start_service https/' /etc/inetsim/inetsim.conf && \
    systemctl restart inetsim.service

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-B-VM-NAME" | grep -i -E "NIC 1|NIC 2" ; \
    VBoxManage showvminfo "NODE-B-VM-NAME" --machinereadable | grep -E "(cpus=|memory=|nestedpaging|l1d-flush|ibpb)" ; \
    VBoxManage showvminfo "NODE-B-VM-NAME" | grep -i "NVMe-Controller" -A 2 ; \
    VBoxManage showvminfo "NODE-B-VM-NAME" | grep -E -i "(Clipboard Mode|Teleporter|USB:|Audio:|Secure Boot)" ; \
    nft list table ip nat ; nft list table ip filter ; \
    systemctl status polarproxy.service | grep -i "Active:" ; \
    grep -E "^(dns_default_ip|start_service)" /etc/inetsim/inetsim.conf

**Optimal Output:**

    NIC 1: MAC: 080027XXXXXX, Attachment: Internal Network 'VBOX-NETWORK-SANDBOX', Cable connected: on, Trace: off (file: none), Type: virtio, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: allow-all, Bandwidth group: none
    NIC 2: MAC: 080027XXXXYY, Attachment: Internal Network 'VBOX-NETWORK-CONTROL', Cable connected: on, Trace: off (file: none), Type: virtio, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none
    cpus=2
    memory=2048
    nestedpaging="on"
    l1d-flush-on-vm-entry="on"
    ibpb-on-vm-entry="on"
    ibpb-on-vm-exit="off"
    Storage Controller Name (0): NVMe-Controller
    Storage Controller Type (0): NVMe
    Storage Controller Instance Number (0): 0
    Clipboard Mode: disabled
    USB: disabled
    Audio: disabled (Driver: None, Controller: None, Codec: Unknown)
    Teleporter Enabled: disabled
    Secure Boot: enabled
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
    Active: active (running)
    start_service dns
    start_service http
    dns_default_ip NODE-B-STATIC-IP