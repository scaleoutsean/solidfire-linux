# solidfire-linux: Notes on Linux with NetApp SolidFire

Notes on Linux clusters with NetApp SolidFire, including (but not limited to) NetApp HCI servers and Mellanox switches.

For additional SolidFire-related information, please refer to [awesome-solidfire](https://github.com/scaleoutsean/awesome-solidfire).

<!-- TOC -->

- [solidfire-linux: Notes on Linux with NetApp SolidFire](#solidfire-linux-notes-on-linux-with-netapp-solidfire)
  - [General Notes](#general-notes)
  - [iSCSI Client Configuration Notes](#iscsi-client-configuration-notes)
    - [NetApp TR-4639](#netapp-tr-4639)
    - [Networking](#networking)
    - [iSCSI](#iscsi)
    - [Multipath I/O](#multipath-io)
    - [udev rules](#udev-rules)
  - [Virtualization](#virtualization)
  - [Containers](#containers)
  - [NetApp HCI Compute Nodes](#netapp-hci-compute-nodes)
    - [NetApp Active IQ OneCollect](#netapp-active-iq-onecollect)
    - [NetApp H410C (and H300E/H500E/H700E)](#netapp-h410c-and-h300eh500eh700e)
      - [Network Adapters and Ports](#network-adapters-and-ports)
    - [NetApp H615C](#netapp-h615c)
    - [Sample Configuration Files](#sample-configuration-files)
      - [netplan](#netplan)
      - [fstab](#fstab)
    - [Linux Driver Updates for NetApp HCI (Compute) Nodes](#linux-driver-updates-for-netapp-hci-compute-nodes)
      - [Installation](#installation)
      - [Update](#update)
  - [Demo Videos](#demo-videos)
  - [Frequently Asked Questions](#frequently-asked-questions)
  - [License and Trademarks](#license-and-trademarks)

<!-- /TOC -->

## General Notes

- Each SolidFire volume is available at a single (iSCSI) IP address. Different iSCSI targets (volumes) may be served over different tagged or untagged iSCSI networks and VLANs when SolidFire is attached to iSCSI networks in switch ports configured in Trunk (or Hybrid) Mode.
  - iSCSI clients login to SolidFire portal - Storage Virtual IP (SVIP) - which redirects each to the SolidFire node which hosts the target (volume) of interest (iSCSI login redirection is described in [RFC-3720](https://tools.ietf.org/html/rfc3720))
  - Volumes are ocassionally rebalanced (the storage node on which they are active changes) and do that transparently to the client (which attempts to access the target and gets redirected to the new location on a different SolidFire storage node)
- Multiple connections from one iSCSI client to single volume (with or without Multipath IO) are rarely needed (NetApp AFF and E-Series are more suitable one or few large workloads)
  - Network adapter teaming (bonding) creates one path to a volume and provides link redundancy, which is enough for 90% of SolidFire use cases
  - There are several ways to create two iSCSI connections to a SolidFire volume. They require Multipath I/O and one of the following (this is not a complete list):
    - Use four NICs to create two teams on the same network, set up one connection from each adapter team's IP address
    - Use two non-teamed NICs on the same network, set up one connection from each interface's IP address
- If you use hundreds or thousands of containers, pay attention to Element (SolidFire) maximums: SolidFire has documented maximums for the number of iSCSI volumes and connections per node (among other things). If you need 1,000 Persistent iSCSI Volumes you should have a cluster of 3-4 SolidFire nodes, for example. Or you could get around that by standing up a NetApp ONTAP Select VM and put hundreds of less busy (or less important, or smaller) volumes on ONTAP NFS v3 or v4 shares backed by SolidFire storage

## iSCSI Client Configuration Notes

### NetApp TR-4639

- NetApp Technical Report 4639 contains detailed information about connecting Linux OS to SolidFire. In great majority of cases it's enough to simply enable jumbo frames

### Networking

- The use of Jumbo Frames is strongly recommended, but not enforced. Possible reasons to not use jumbo frames might be network limitations, client limitations or the need to use SolidFire replication to a remote site when WAN/VPN can't handle Jumbo Frames
- Use enterprise-grade network switches such as Mellanox SN2010 (which you can purchase from NetApp)
- Consider disabling or discarding:
  - IPv6 on ports used on iSCSI network(s), if you don't have other, IPv6-capable iSCSI targets in your environment
  - DHCP service on iSCSI and Live Migration network(s), but if you have to run DHCP on those networks, hand out MTU 9000 through DHCP options
- It appears that light and moderate workloads don't require any tuning on iSCSI clients (even Jumbo Frames, although that is reccommended)
- It is practically mandatory to use Trunk (or Hybrid) Mode on 10/25 GigE because in all likelihood you'll need more than one VLAN for iSCSI, backup and other purposes. Mellanox SN2010 has a variant of it, Hybrid Mode

### iSCSI

- SolidFire supports open-iscsi iSCSI initiator
  - Default options work well (the KISS principle!)
    - If using RHCOS >=4.5 or RHEL or CentOS >=8.2 or other recent Linux distribution releases, ensure that the CHAP authentication algorithm is set to MD5 in /etc/iscsi/iscsid.conf (`sudo sed -i 's/^\(node.session.auth.chap_algs\).*/\1 = MD5/' /etc/iscsi/iscsid.conf`). This applies to as of now latest SolidFire 12.3 and earlier
  - I haven't seen evidence that changing other iSCSI initiator options helps more than it hurts, at least for workloads that commonly run on SolidFire
- Use CHAP, IQNs, VLANs or "all of the above"?
  - CHAP is easier and recommended for NetApp Trident (i.e. container environments)
  - KVM clusters can use IQNs and SolidFire Volume Access Group (VAG) let you group IQNs from multiple systems into goups so when a volume is created and assigned to a VAG, it is accessible to any of the hosts in the IQN group
  - An environment can use multiple approaches (e.g. IQNs from several KVM servers grouped into SolidFire Volume Access Groups and 3 CHAP accounts for 3 Kubernetes clusters in the same environment)
  - Volume access may require both VAG membership and CHAP authentication (see Element documentation for the higher v11 versions or v12.0)
- Configure and set up Linux client as you normally would, based on your Linux distribution's documentation for iSCSI. List of packages required for iSCSI on RPM and DEB based distributions can be found in [NetApp Trident docs](https://netapp-trident.readthedocs.io/en/latest/docker/install/host_config.html), but you should really read generic docs for you distribution. As mentioned earlier, multipathd and device-mapper-multipath are usually not required if you only use SolidFire iSCSI targets.

### Multipath I/O

- If you don't have multiple links to SolidFire or other iSCSI target(s), you probably don't need it (one thing less to install, configure, wait for to start-up, and break-fix). Of course, you'd have VM or container failovers if a switch or cable fails
- There are no recent performance comparisons of various bonding options for SolidFire Linux clients. LACP should give best results in terms of performance, but feel free to experiment with other options
- multipath.conf: try adding a `device` section that looks like this (this should be quite accurate; VMware MPIO settings for SolidFire also balance I/O in batches of 10)

```raw
devices {
  device {
  vendor "SolidFir"
  product "SSD SAN"
  path_grouping_policy multibus
  path_selector "round-robin 0"
  path_checker tur
  hardware_handler "0"
  failback immediate
  rr_weight uniform
  rr_min_io 10
  rr_min_io_rq 10
  features "0"
  no_path_retry 24
  prio const
  skip_kpartx yes
  }
}
```

### udev rules

- There are various udev rule examples in SolidFire Linux-related TRs, but like with other "tuning" suggestions I haven't seen much evidence that one device setting (say, `max_sectors_kb=1024`) is better or worse than any other, any why. Until I know, I'd rather not customize those settings
- `udevadm info` can be used to show you how to compose a SolidFire specific rule
- TODO

## Virtualization

- See the Virtualization section of [awesome-solidfire](https://github.com/scaleoutsean/awesome-solidfire#virtualization)

## Containers

- See the Containers section of [awesome-solidfire](https://github.com/scaleoutsean/awesome-solidfire#kubernetes-and-containers)
- If you ruse CoreOS or Flatcar Container Linux, see [this post](https://scaleoutsean.github.io/2021/12/07/flatcar-linux-with-solidfire-iscsi.html)
- VMware Photon - see [this](https://github.com/scaleoutsean/photon-solidfire) and the blog post linked in the README

## NetApp HCI Compute Nodes

### NetApp Active IQ OneCollect

- [OneCollect](https://mysupport.netapp.com/site/tools/tool-eula/activeiq-onecollect/download) can be set to periodically collect event logs from SolidFire and Windows Server systems collected to it
- It also makes it convenient and easy to submit logs to NetApp Support

### NetApp H410C (and H300E/H500E/H700E)

- Consider disabling IPv6 on interfaces used on iSCSI network(s), if you don't have other, IPv6-capable iSCSI targets in your environment
- Recent Linux distributions come with all drivers required and some (like RHEL) may be more prescriptive and support only specific versions. But, as mentioned above, free feel to download and use any driver version that works for you and that is supported by your distro (if you care about that). 
  - Mellanox ConnectX-4 Lx NIC driver ([Mellanox EN and OFED Drivers](https://www.mellanox.com/products/adapter-ethernet-sw) - mlnx-en-dkms ensures auto-rebuild on kernel change. NOTE: [NIC firmware](https://www.mellanox.com/support/firmware/connectx4lxen) tested with various Mellanox drivers is distributed separately, so best take it easy to rushing with driver updates! A safe approach is to use latest firmware recommended for NetApp HCI (with VMware) and use the latest driver that works with that firmware version.
  - (Optional, for those who use RJ-45 NIC ports) Intel X550 NIC driver ([v5.7.1](https://downloadcenter.intel.com/download/14687/Intel-Network-Adapter-Driver-for-PCIe-Intel-10-Gigabit-Ethernet-Network-Connections-Under-Linux-))
  - (Optional) [ASpeed Linux Driver](https://www.aspeedtech.com/support.php?fPath=24) - to address the cosmetic "/lib/firmware/ast_dp501_fw.bin for module ast" warning
- On NetApp H400 Series compute nodes (6 network ports) it may be more convenient to combine 2 or 4 Mellanox NICs into 1 or 2 LACP Teams and use Trunk (or optionally Hybrid, with Mellanox switches) Mode on network switch ports and VLANs on bridges or vSwitches to segregate workloads and tenants, than having 2 or 3 groups of teamed interfaces. Some users prefer to physically segregate traffic (DMZ, Public, Private) before applying additional isolation (bridges, VLANs).

#### Network Adapters and Ports

- enp24s0f0 and enp24s0f1 are 1/10 GigE Intel X550 (RJ-45)
- The rest are Mellanox Connect-4 Lx with SFP28 (2 dual-ported NICs, SFP28/SFP+)
  - enp25s0f0
  - enp25s0f1
  - enp59s0f0
  - enp59s0f1
- Up to 6 ports that may be used. From left to right we label them A through F (HCI Port column)
- IPMI (RJ-45) port is not shown

```raw
| PCI | Bus | Device | Func | HCI Port | Default OS Name   | Description                             |
|-----|-----|--------|------|----------|-------------------|-----------------------------------------|
| 6   | 24  | 0      | 0    | A        | enp24s0f0         | Intel(R) Ethernet Controller X550       |
| 6   | 24  | 0      | 1    | B        | enp24s0f1         | Intel(R) Ethernet Controller X550       |
| 7   | 25  | 0      | 0    | C        | enp25s0f0         | Mellanox ConnectX-4 Lx Ethernet Adapter |
| 7   | 25  | 0      | 1    | D        | enp25s0f1         | Mellanox ConnectX-4 Lx Ethernet Adapter |
| 1   | 59  | 0      | 1    | E        | enp59s0f0         | Mellanox ConnectX-4 Lx Ethernet Adapter |
| 1   | 59  | 0      | 0    | F        | enp59s0f1         | Mellanox ConnectX-4 Lx Ethernet Adapter |
```

- (TODO: verify port, function and name (enp59s0f0 and enp59s0f1) mapping for HCI Ports E & F)

- NetApp HCI H410C with 6 cables and VMware ESXi uses vSS (switch) and assigns ports as per below. With Linux we may configure them differently so this is just for reference purposes to get you started:

```raw
| HCI Port | Mode   | Purpose                       |
|----------|--------|-------------------------------|
| A        | Access | Management (X550)             |
| B        | Access | Management (X550)             |
| C        | Trunk  | VM Network & Live Migration   |
| D        | Trunk  | iSCSI                         |
| E        | Trunk  | iSCSI                         |
| F        | Trunk  | VM Network & Live Migration   |
```

- Because A and B are RJ-45 ports, most users choose to team ports C-F in one or two (e.g. iSCSI and the rest) Teams, while A and B are normally used for management (if at all), so let's focus on the Mellanox NICs:

```raw
Device (19:00.0):
        19:00.0 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
        Link Width: x8
        PCI Link Speed: 8GT/s

Device (19:00.1):
        19:00.1 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
        Link Width: x8
        PCI Link Speed: 8GT/s

Device (3b:00.0):
        3b:00.0 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
        Link Width: x8
        PCI Link Speed: 8GT/s

Device (3b:00.1):
        3b:00.1 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
        Link Width: x8
        PCI Link Speed: 8GT/s

- The same in `lspci` output:
```shell
19:00.0 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
19:00.1 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
3b:00.0 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
3b:00.1 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]

- The Intel X550 NIC;
```shell
18:00.0 Ethernet controller: Intel Corporation Ethernet Controller 10G X550T (rev 01)
18:00.1 Ethernet controller: Intel Corporation Ethernet Controller 10G X550T (rev 01)
```

### NetApp H615C

- Two Mellanox Connect-4 Lx with SFP28 (1 dual-ported NIC)
  - See Mellanox network driver-related notes under H410C
  - Because there are only two NICs, most would likely team two ports using LACP
  - Adapter names (Ubuntu 18.04 and 20.04): ens5f0 and ens5f1
  - The Mellanox NIC in `lspci` output:

```shell
3b:00.0 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
3b:00.1 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
```

- NVIDIA Tesla T4 driver (selected H615C model): [download](https://www.nvidia.com/download/index.aspx?lang=en-us) a version from nvidia.com supported by your ISV or OS. At the time of writing the current driver NVIDIA Tesla T4 v440.64.00 and CUDA v10.2 worked without issues on H615C with Ubuntu 20.04. Select Product Type Tesla, Product Series T-Series, Product Tesla T4.

### Sample Configuration Files

#### netplan

- Basic Netplan configuration file for H615C: [network_netplan_h615c_2_cable_01-netcfg.yaml](./reference/network_netplan_h615c_2_cable_01-netcfg.yaml)
  - Netplan example is provided because it's less popular (i.e. it's not as easy to find reference configuration files for it)
  - The file can be easily adjusted. Feel free to add bonds, bridges, and VLANs...
  - NetApp HCI NVA for Redhat OpenShift provides a prescriptive guidance for networking layout on a RPM distribution
- Netplan with 3 NICs: bridge on the first NIC (ens160) and DHCP on iSCSI network (ens192 & ens224), from Ubuntu 20.04

```raw
network:
  version: 2
  renderer: networkd
  ethernets:
    ens160:
      dhcp4: no 
      dhcp6: no 
    ens192:
      dhcp4: true
      dhcp6: no 
    ens224:
      dhcp4: true
      dhcp6: no 
  bridges:
  bridges:
    br0:
      interfaces: [ens160]
      addresses: [192.168.1.157/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [192.168.1.5, 192.168.1.4]
      parameters:
        stp: true
        forward-delay: 4
      dhcp4: no
      dhcp6: no
```

#### fstab

- This isn't to say this is "better" than other approaches. You'd need a different path with multipath, but if you keep it simple and have one NIC for iSCSI, this can work without device slippage just like it could with `blkid`'s. Key detail is `_netdev,x-systemd.requires=iscsi.service`, because _netdev is needed for iSCSI. Example from Ubuntu 20.04, but should work with other Linux distributions.

```raw
/dev/disk/by-path/ip-192.168.103.30:3260-iscsi-iqn.2010-01.com.solidfire:mn4y.kvmv01.359-lun-0 /data ext4 _netdev,x-systemd.requires=iscsi.service,noatime,discard 0 1
```

- SolidFire Storage VIP cannot be changed, and neither can unique Cluster ID (`mn4y`, above). Volume Name (`kvm01`) can be changed, but Volume ID (359) cannot. `lun-0` is a partitionless volume (`mkfs.ext4 /dev/sdb`). So if you don't change volume name, device name from the fstab example above won't change.
- For periodic discard, and discard of non-ext4 filesystems, see [Archlinux Wiki](https://wiki.archlinux.org/index.php/Solid_state_drive)

### Linux Driver Updates for NetApp HCI (Compute) Nodes

This are my unofficial notes. Please verify with your NetApp account team if you want to know the official version.
#### Installation

General approach (example for Mellanox NIC(s) on the H410C and H615C):

- Check the latest NetApp HCI drivers (login for support site required) and with that find the latest f/w for your system and NIC. Pay attention - H615C and H410C may each support a different Mellanox Lx4 firmware version, although the both use the same chip
- Now with this firmware version information go to NVIDIA/Mellanox driver downloads and start looking from newer Ethernet drivers for Linux. Driver Release Notes contain the information about recommended and supported NIC f/w
- Do the same for the Intel NIC (if applicable, such as on the H410C)
- Update NIC f/w first and then install a matching Linux NIC driver
- If you install a Mellanox driver that's very new and does not support f/w you have, it may offer to upgrade firmware. If you want to us the NetApp-supplied f/w, decline this offer and install a driver that supports the f/w you want to use

#### Update

You can update compute node firmware (including BIOS, BMC drivers and the rest) from NetApp HCI NDE or manually. If another OS (such as Linux) is already installed, you'd probably want to upgrade f/w without NDE.

When NetApp HCI releases a f/w upgrade in a new NDE version, you could get the f/w from the HCI Compute ISO or download it from the NVIDIA/Mellanox Web site, use the MFT utility to upgrade firmware and finally, and finally install one of Linux drivers that support that f/w. If you don't have any issues, better don't upgrade (considering that Linux OS can still change its routing or NIC names during routine maintenance operations).

In theory, if Linux is installed on the second internal disk, disk #1 (not disk #0, where NetApp HCI Bootstrap OS is installed in the case of NetApp HCI with VMware), you may be able to update BIOS, BMC and other f/w from NetApp NDE, while leaving Linux in place (on disk #1). This is how it works for ESXi, but it's not explicitly supported with Linux. It should work, though.

## Demo Videos

I'm not actively updating this list - please just use a search engine or check my [blog](https://scaleoutsean.github.io/2022/02/22/openstack-solidfire.html).

- [OpenStack](hhttps://scaleoutsean.github.io/2022/02/22/openstack-solidfire.htm) - basic demo of how OpenStack Cinder works with SolidFire iSCSI
- [Rancher k3s & NetApp Trident with SolidFire iSCSI back-end](https://youtu.be/CS5iEPGOfA4)
- [Booting Debian Linux from SolidFire iSCSI Target](https://youtu.be/JVZoMGxte4c) - based on generic Linux documentation
- [Essential KVM Storage management with SolidFire and Cockpit Web UI, virsh and SolidFire Python CLI](https://youtu.be/8cYk2gNnMo8)
- [Backup and Restore of KVM with SolidFire, Duplicati and S3 object storage](https://youtu.be/wP8nAgFo8og)
- [Connect XenServer to SolidFire iSCSI Target](https://youtu.be/kBePozPT_6A) - for XenServer v7; for v8 (aka Citrix Hypervisor) please see the official NetApp HCI - Citrix Hypervisor solution documentation
- [Storage failover and failback with Kubernetes, NetApp Trident and SolidFire](https://youtu.be/f-PJGCtEojQ) - long and slow, but maybe useful to your planning of failover for K8s environments with SolidFire
- [Oracle VirtualBox 6.1 and SolidFire](https://youtu.be/xZJdAZ_FOog) - VirtualBox can directly use SolidFire iSCSI targets
- [Rubrik in a Linux environment with NetApp HCI compute nodes](https://youtu.be/4C4o5DUhmrQ)

## Frequently Asked Questions

Q: Is this what NetApp recommends?

A: No. While some of this may be correct, please refer to the official documentation.

## License and Trademarks

- NetApp, SolidFire and other marks are owned by NetApp, Inc. Other marks may be owned by their respective owners.
- See [LICENSE](LICENSE).
