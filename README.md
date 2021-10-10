
# OpenWrt as a Firecracker MicroVM

Boots in less than 4 seconds.

Noteworthy:
- Reboot and shutdown are wonky in Firecracker VMs. `poweroff` will halt the guest system but not exit the VM, `reboot` will exit the VM as you'd expect from `poweroff`. From the host's side, you can simply kill the Firecracker process for a non-graceful shutdown.
- Kernel needs `CONFIG_VIRTIO_MMIO_CMDLINE_DEVICES=y` -- see https://github.com/openwrt/openwrt/pull/4666 and https://github.com/firecracker-microvm/firecracker/issues/2021#issuecomment-939302986

```sh
# Set up vanilla OpenWrt build environment
git clone https://github.com/openwrt/openwrt
cd openwrt/
./scripts/feeds update -a
./scripts/feeds install -a

# Firecracker needs this
echo CONFIG_VIRTIO_MMIO_CMDLINE_DEVICES=y >> target/linux/x86/64/config-5.10

# Select x86_64 target
echo CONFIG_TARGET_x86=y >> .config
echo CONFIG_TARGET_x86_64=y >> .config
echo CONFIG_TARGET_x86_64_DEVICE_generic=y >> .config

# Skip the unneccessary 2 second wait for failsafe mode
echo CONFIG_IMAGEOPT=y >> .config
echo CONFIG_PREINITOPT=y >> .config
echo CONFIG_TARGET_PREINIT_DISABLE_FAILSAFE=y >> .config

# Fill in the rest of the build config
make defconfig

# Build it
make

# Copy kernel and rootfs
cp -v build_dir/target-x86_64_musl/linux-x86_64/vmlinux.elf ../vmlinux.elf
gunzip -c bin/targets/x86/64/openwrt-x86-64-generic-ext4-rootfs.img.gz > ../rootfs.img

cd ..
sudo `which firecracker` --no-api --config-file ./vmconfig.json
```
