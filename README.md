
> This repository has been archived - find the latest version at https://github.com/pktpls/notes/tree/main/openwrt-firecracker

---

# OpenWrt as a Firecracker MicroVM

- Boots in less than 4-6 seconds. There are a number of large sleeps in the OpenWrt boot process that can probably be avoided in a VM (no actual hardware to wait for).
- Works with OpenWrt snapshots and not with OpenWrt 21.02, because the kernel needs to have `CONFIG_VIRTIO_MMIO_CMDLINE_DEVICES` enabled, which was added in October 2021.
- Reboot and shutdown are wonky in Firecracker VMs. `poweroff` will halt the guest system but not exit the VM itself, `reboot` will exit the VM as you'd expect from `poweroff`. From the host's side, you can simply kill the Firecracker process for a non-graceful shutdown.

## Basic usage

OpenWrt automatically uses `eth0` for the `lan` bridge and `eth1` for `wan`/`wan6`. If the latter doesn't exist, it will simply skipped and only the `lan` bridge created.

Get the kernel and rootfs images.
```sh
wget -O extract-vmlinux.sh https://github.com/torvalds/linux/raw/master/scripts/extract-vmlinux
chmod +x extract-vmlinux.sh

wget -O bzImage https://downloads.openwrt.org/snapshots/targets/x86/64/openwrt-x86-64-generic-kernel.bin
./extract-vmlinux.sh bzImage > vmlinux.elf

wget -O rootfs.img.gz https://downloads.openwrt.org/snapshots/targets/x86/64/openwrt-x86-64-generic-ext4-rootfs.img.gz
gunzip rootfs.img.gz
```

Create host-side interfaces for the VM's eth0/lan and eth1/wan interfaces. The interface names must match what's specified in `vmconfig.json`.
```sh
sudo ip tuntap add dev tap0 mode tap
sudo ip addr add 192.168.1.2/24 dev tap0 # could alternatively obtain DHCP lease
sudo ip link set dev tap0 up

sudo ip tuntap add dev tap1 mode tap
sudo ip addr add 192.168.123.1/24 dev tap1
sudo ip link set dev tap1 up
```

In a separate shell, start the VM. Note that changes to the ext4 rootfs partition are persistent. Other rootfs options (layered or ephemeral) haven't been evaluated yet.

```sh
sudo `which firecracker` --no-api --config-file ./vmconfig.json
```

Now the host can already ping the `lan` bridge.
```sh
ping 192.168.1.1
ssh root@192.168.1.1 ping 192.168.1.2
```

To stop the VM, you use `reboot` inside or `kill` outside.
```sh
sudo killall firecracker
```

## DHCP for eth1 / wan

To get the VM's `wan` interface up, again in a separate shell, start the host's DHCP server for the guest. The `-d` option enables debug mode, so dnsmasq prints the incoming DHCP requests. If you want the VM to have Internet access, you'd also need to enable forwarding and possibly NAT/masquerading separately.
```sh
sudo dnsmasq -d --bind-dynamic --listen-address=192.168.123.1 --dhcp-range=192.168.123.10,192.168.123.100
```

## Multiple VMs

Each VM needs its own TAP interfaces and `vmconfig.json` file.

If you want to run multiple VMs with a bridge connecting their TAP interfaces, you'd need to enable ARP proxying so the VMs can see each other via the host.
```sh
sudo sysctl net.ipv4.conf.tap0.proxy_arp=1 # optional
```

## VLAN

Because TAP interfaces carry Ethernet frames (while TUN interfaces carry IP packets), many convenient things just work, including VLANs.

On the host:
```sh
sudo ip link add link tap0 name tap0.foo type vlan id 42
sudo ip addr add 10.42.0.2/24 dev tap0.foo
sudo ip link set tap0.foo up
```

In the VM:
```sh
ip link add link eth0 name eth0.foo type vlan id 42
ip addr add 10.42.0.1/24 dev eth0.foo
ip link set eth0.foo up
```

Et voila:
```sh
ping 10.42.0.1
ssh root@10.42.0.1 ping 10.42.0.2
```
