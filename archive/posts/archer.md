---
title: "I Just Wanted to Clone My Gaming PC"
date: 2026-02-02
draft: false
tldr: "Started cloning a Windows install to a VM, ended up building a framework that reads 31 filesystems, 12 disk image formats, and runs entirely in the browser."
---

I have a gaming PC. RTX 5090, Windows 11, the works. The problem is I don't actually game that much. Most days it just sits there, an expensive space heater drawing idle power while I work on a Mac.

After building [gunmetal](/posts/heavymetal) (a Groth16 prover on Apple Metal), I wanted to add CUDA support. But developing on Windows is painful, and I needed a proper Linux build target anyway. So the obvious idea formed: run Linux as the base OS, run Windows as a VM. Boot Windows when I want to game, use Linux for development the rest of the time. With GPU passthrough, Windows gets the 5090 and Linux uses the integrated adapter. Two build servers from one machine.

Simple plan. One complication: 800 GB of storage, applications, games. I wasn't about to reinstall everything from scratch. I'm not that eager to waste unnecessary time. I'd clone the existing Windows install and restore it inside a VM.

At this time I had no idea what I had done to myself.

## Cloning is the easy part

[Macrium Reflect X](https://www.macrium.com/) turned out to be a solid cloning tool. Image the drive, get a backup in their `mrimgx` format, restore it somewhere else. Straightforward.

But I needed to test the restore on Linux before wiping the machine. And Macrium Reflect is Windows-only. You can't restore an `mrimgx` image on Linux. There's no tool for it.

I dug around and found that Macrium had released [specs for their file format](https://github.com/macrium/mrimgx_file_layout) along with a Visual Studio C++ project you can build on Windows. That meant either installing VS Code and C++ SDKs on the gaming machine, or using their pre-built `img_to_vhdx` demo executable.

Neither option appealed to me. So I did the reasonable thing and wrote an `mrimgx` parser in Rust.

## One parser leads to another

The Rust parser worked great. I could read the Macrium image on my Mac and convert it to QCOW2 for QEMU. (Yes, yes, QEMU has tools to convert from VHDX too. That's beside the point.)

I extended the parser to understand partition tables so I could open the disk properly. Write to QCOW2, no problems.

Then I tried to boot it.

Blue screen.

Windows was looking for native disk drivers that don't exist in a virtual machine. The fix is to modify the Windows registry to disable those drivers before booting. Specifically, you edit the registry HIVE files on the NTFS volume.

But to do that, you need to read NTFS. The [existing Rust crate for NTFS](https://docs.rs/ntfs/0.4.0/ntfs/) is read-only and wasn't deep enough into MFT parsing to avoid recursive indexing, so I wrote my own. Read and write.

Then I needed to modify the registry. There were existing Rust crates for reading registry HIVE files, but none of them supported writing. So I built a registry HIVE writer too. It's only later I discovered that [peitaosu](https://github.com/peitaosu/) had contributed his own [HIVE editor](https://github.com/peitaosu/regf) that supports read/write. Oh well.

The chain looked like this: unpack the `mrimgx` image, open the partition table, find the NTFS volume, mount it, locate the registry hive, modify it to disable the incompatible driver, write everything back, boot in QEMU.

It worked.

## The rabbit hole has no end

I should have stopped there. I had what I needed. The Windows VM booted, the gaming PC was free to become a Linux machine. Mission accomplished.

But I didn't stop. Before I quite understood what was happening, I had added support for 31 filesystems. With most of their features. Including RAID. All in pure Rust. I called it archer.

Here's what it reads now.

Windows/DOS: NTFS, FAT12, FAT16, FAT32, exFAT, ReFS

Linux: ext2, ext3, ext4, XFS, Btrfs, ZFS, JFS, ReiserFS, Reiser4, F2FS, SquashFS, DwarFS

macOS/Apple: APFS, HFS, HFS+

BSD: UFS, UFS2

DragonFly BSD: HAMMER, HAMMER2

Other: VxFS, bcachefs, ISO 9660, UDF

Every single one parsed from first principles in Rust. No C bindings, no FUSE, no shelling out to system tools.

## "All your disks are belong to us" (iykyk)

If you can read filesystems, you need to read the containers they live in. Archer supports 12 disk image formats:

VMDK, QCOW2, VHD, VHDX, VDI, Parallels, Raw/IMG, DMG, OVA, IPSW, ISO 9660

Backup formats: Veeam (VBK/VIB), Acronis (TIB/TIBX), and Macrium Reflect X (.mrimgx).
## Encryption is not optional

Real disks are encrypted. If you want to open actual disk images, encryption support isn't optional. Archer supports:

BitLocker (with recovery key), LUKS/LUKS2, QCOW2 LUKS, DMG AES-128/AES-256, and Veeam encryption.

It also handles LVM2, Linux RAID (md), and ZFS pools for volume management. Because real-world disks aren't just a single partition with a single filesystem. They're layered, encrypted, striped, mirrored, and nested in ways that make you question the sanity of whoever set them up.

## Obviously, it runs in the browser

I compiled the whole thing to WASM.

![archer running in the browser](/images/hdt-preview.png)

You can open an 880 GB disk image in under a second. Browse the filesystem, search for files, edit the Windows registry, explore partitions. All in the browser. No installation, no downloads, no plugins. Just open the page and point it at a disk image.

This works because the WASM code only reads the bytes it needs. It parses partition tables, filesystem metadata, and directory structures on demand. The 880 GB file isn't loaded into memory. It's accessed through range requests, reading only the sectors that matter for whatever you're looking at right now.

## How we got here

Every step was logical at the time. I needed to clone a disk, so I wrote a parser. The parser needed to understand partitions, so I added that. The VM wouldn't boot, so I needed NTFS, then a registry editor. Once you have NTFS you think "ext4 isn't that different" and then "well, ZFS would be interesting" and then some moments in you're reading specs for filesystems you've never even used.

The scope is absurd. I know that. But the thing works, it's fast, it runs everywhere, and every piece of it exists because at some point I genuinely needed it, maybe. Who cares.

## Now what

I registered a domain for it. I think there's something genuinely useful here for anyone who needs to look inside a disk image without installing platform-specific tools or booting up a specific OS. IT forensics, data recovery, migration work, or just curiosity about what's on an old drive.

Oh, and I still haven't wiped that gaming PC.
