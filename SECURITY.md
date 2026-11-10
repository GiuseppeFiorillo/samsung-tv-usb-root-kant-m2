# Security / responsible use

This repository contains a working kernel exploit for one Samsung TV firmware
family. Use it only on hardware you own or are explicitly authorized to test.

The port was hardware-tested only on QE49Q6FNA with T-KTM2DEUC 1400.2. The
exFAT bug is present across the Kant-M2 platform, but the kernel addresses are
specific to the kernel image. A mismatch can hang or reboot the TV. Static
compatibility is not permission to use an image on another set.

Before testing:

* keep the official firmware recovery image available;
* verify the generated image checksum and the destination block device;
* keep the USB stick inserted for every cold boot, as described in
  [docs/USB-STICK.md](docs/USB-STICK.md);
* understand that a failed p1 trigger can require a power cycle.

Do not submit device-private certificates, keys, serial numbers, live
inventory, or proprietary firmware/modules to this repository.

For security-sensitive contact, open a minimal GitHub issue requesting a
private channel; do not attach secrets or affected third-party data.
