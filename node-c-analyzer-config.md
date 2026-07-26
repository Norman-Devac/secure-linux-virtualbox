**Implementation Command:**

    VBoxManage modifyvm "NODE-C-VM-NAME" --nic1=intnet --intnet1="VBOX-NETWORK-CONTROL" --nic-type1=virtio --nic-promisc1=allow-all && \
    VBoxManage modifyvm "NODE-C-VM-NAME" --cpus=4 --cpu-execution-cap=100 --memory=8192 && \
    VBoxManage modifyvm "NODE-C-VM-NAME" --nested-paging=on --large-pages=on --page-fusion=off && \
    VBoxManage storagectl "NODE-C-VM-NAME" --name "NVMe-Telemetry" --add=pcie --controller=NVMe --portcount=1 --hostiocache=off && \
    VBoxManage storageattach "NODE-C-VM-NAME" --storagectl "NVMe-Telemetry" --port 0 --device 0 --type hdd --medium "NODE-C-VDI-FILENAME" && \
    VBoxManage modifyvm "NODE-C-VM-NAME" --audio-driver=none --audio-controller=none --usb=off --usbehci=off --usbxhci=off && \
    VBoxManage modifyvm "NODE-C-VM-NAME" --clipboard-mode=disabled --drag-and-drop=disabled --vrde=off && \
    VBoxManage modifyvm "NODE-C-VM-NAME" --spec-ctrl=on --l1d-flush-on-vm-entry=on --mds-clear-on-vm-entry=on --ibpb-on-vm-entry=on --ibpb-on-vm-exit=on && \
    VBoxManage modifyvm "NODE-C-VM-NAME" --firmware=efi64 && \
    VBoxManage modifynvram "NODE-C-VM-NAME" inituefivarstore && \
    VBoxManage modifynvram "NODE-C-VM-NAME" enrollorclpk && \
    VBoxManage modifynvram "NODE-C-VM-NAME" secureboot --enable && \
    sed -i '/^af-packet:/,/^  - interface:/{s/^  - interface:.*/  - interface: NODE-C-LINUX-INTERFACE\n    cluster-id: 99\n    cluster-type: cluster_qm\n    defrag: yes\n    use-mmap: yes\n    tpacket-v3: yes\n    ring-size: 200000/}' /etc/suricata/suricata.yaml && \
    systemctl restart suricata.service && \
    sed -i 's/^interface=.*/interface=NODE-C-LINUX-INTERFACE/' /opt/zeek/etc/node.cfg && \
    /opt/zeek/bin/zeekctl deploy && \
    cat << 'EOF' > /opt/arkime/etc/config.ini
    [default]
    interface=NODE-C-LINUX-INTERFACE
    pcapReadMethod=pcap-over-ip-server
    pcapoverip=NODE-B-STATIC-IP:NODE-C-PCAP-PORT
    elasticsearch=NODE-C-ELASTICSEARCH-URL
    EOF
    systemctl restart arkimecapture.service

**Diagnostic Command:**

    VBoxManage showvminfo "NODE-C-VM-NAME" | grep -i -E "(NIC 1|Number of CPUs|Memory size)" ; \
    VBoxManage showvminfo "NODE-C-VM-NAME" | grep -i "NVMe-Telemetry" -A 2 ; \
    VBoxManage showvminfo "NODE-C-VM-NAME" | grep -E -i "(Clipboard Mode|USB:|Audio:|Secure Boot|mds-clear)" ; \
    grep -A 8 "^af-packet:" /etc/suricata/suricata.yaml ; \
    /opt/zeek/bin/zeekctl status ; \
    systemctl status arkimecapture.service | grep -i "pcapoverip"

**Optimal Output:**

    Memory size: 8192MB
    Number of CPUs: 4
    NIC 1: MAC: 080027XXXXZZ, Attachment: Internal Network 'VBOX-NETWORK-CONTROL', Cable connected: on, Trace: off (file: none), Type: virtio, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: allow-all, Bandwidth group: none
    Storage Controller Name (0): NVMe-Telemetry
    Storage Controller Type (0): NVMe
    Storage Controller Instance Number (0): 0
    Clipboard Mode: disabled
    USB: disabled
    Audio: disabled (Driver: None, Controller: None, Codec: Unknown)
    Secure Boot: enabled
    mds-clear-on-vm-entry="on"
    af-packet:
    interface: NODE-C-LINUX-INTERFACE
    cluster-id: 99
    cluster-type: cluster_qm
    defrag: yes
    use-mmap: yes
    tpacket-v3: yes
    ring-size: 200000
    Name Type Host Status Pid Started
    zeek standalone localhost running 11234 System Initialization
    CGroup: /system.slice/arkimecapture.service
    └─12456 /opt/arkime/bin/capture -c /opt/arkime/etc/config.ini --pcapoverip