# Full disk RAID support

## Theory of operation

Users can write FCC sugar that enables redundant boot and/or an encrypted root volume.  There are three cases:

1. The sugar only specifies encrypted root.  FCCT desugars the section into a LUKS volume within the existing root partition, and a root filesystem layered on top.
2. The sugar only specifies redundant boot.  FCCT desugars the section into an Ignition config which completely recreates the partition structure on each specified disk.  The config creates partitions, RAID volumes, and filesystems for the root and boot partitions; partitions and independent filesystems for each ESP replica; and partitions for each BIOS-BOOT replica.
3. The sugar specifies both.  FCCT desugars similarly to the second case, with the addition of a LUKS volume.

If redundant boot is specified, Dracut glue detects the corresponding config stanzas, saves the corresponding on-disk data to RAM before the Ignition disks phase, and restores it after disks phase completes.  The glue also replicates the boot sector when replicating BIOS-BOOT.

Each piece of the replication logic (root partition, boot partition, ESP, and BIOS-BOOT + MBR) functions independently.  An advanced user can create nonstandard partition configurations (for example, RAID-1 boot with RAID-5 root) by manually configuring partitions in their FCC.

Recovery from a disk failure is out of scope for this enhancement.  We will recommend that users recover from a hardware failure by reprovisioning the machine.

## FCC sugar

```yaml
variant: fcos
version: 1.3.0
boot_device:
  # Allows choosing from multiple disk layout templates hardcoded in FCCT.
  # We might use this to support CPU architectures with different partition
  # requirements (e.g. ppc64le) or to change the disk layout in future OS
  # releases.
  # Optional, defaults to x86_64
  layout: x86_64
  # If specified, mirror every default partition
  mirror:
    devices:
      # Two or more devices are required
      - /dev/vda
      - /dev/vdb
  # If specified, encrypt the root partition
  luks:
    # The contents of the existing clevis struct, without the `custom`
    # field (which is too advanced for sugar).  We don't need to support
    # static key files because they don't work for root.
    tang:
      - url: https://example.com/tang.tang.tang
        thumbprint: x
    tpm2: true
    threshold: 2
```

The `boot_device` struct is at the top level, rather than under `storage`, because CT sugar has historically been placed at the top level and because Go struct embedding semantics make it harder to put it under `storage`.

The sugar does not support advanced features such as RAID spares, RAID levels other than 1, or filesystem/mount options.  Users can manually specify individual partitions, RAIDs, and filesystems if desired.

## Rendered Ignition config

### LUKS only

```json
{
  "ignition": {
    "version": "3.2.0"
  },
  "storage": {
    "filesystems": [
      {
        "device": "/dev/disk/by-id/dm-name-root",
        "format": "xfs",
        "label": "root",
        "wipeFilesystem": true
      }
    ],
    "luks": [
      {
        "clevis": {
          "tpm2": true
        },
        "device": "/dev/disk/by-partlabel/root",
        "label": "luks-root",
        "name": "root",
        "wipeVolume": true
      }
    ]
  }
}
```

### RAID only

```json
{
  "ignition": {
    "version": "3.2.0"
  },
  "storage": {
    "disks": [
      {
        "device": "/dev/vda",
        "partitions": [
          {
            "label": "bios-1",
            "sizeMiB": 1,
            "typeGuid": "21686148-6449-6E6F-744E-656564454649"
          },
          {
            "label": "esp-1",
            "sizeMiB": 127,
            "typeGuid": "C12A7328-F81F-11D2-BA4B-00A0C93EC93B"
          },
          {
            "label": "boot-1",
            "sizeMiB": 384
          },
          {
            "label": "root-1"
          }
        ],
        "wipeTable": true
      },
      {
        "device": "/dev/vdb",
        "partitions": [
          {
            "label": "bios-2",
            "sizeMiB": 1,
            "typeGuid": "21686148-6449-6E6F-744E-656564454649"
          },
          {
            "label": "esp-2",
            "sizeMiB": 127,
            "typeGuid": "C12A7328-F81F-11D2-BA4B-00A0C93EC93B"
          },
          {
            "label": "boot-2",
            "sizeMiB": 384
          },
          {
            "label": "root-2"
          }
        ],
        "wipeTable": true
      }
    ],
    "filesystems": [
      {
        "device": "/dev/disk/by-partlabel/esp-1",
        "format": "vfat",
        "label": "esp-1",
        "wipeFilesystem": true
      },
      {
        "device": "/dev/disk/by-partlabel/esp-2",
        "format": "vfat",
        "label": "esp-2",
        "wipeFilesystem": true
      },
      {
        "device": "/dev/md/md-boot",
        "format": "ext4",
        "label": "boot",
        "wipeFilesystem": true
      },
      {
        "device": "/dev/md/md-root",
        "format": "xfs",
        "label": "root",
        "wipeFilesystem": true
      }
    ],
    "raid": [
      {
        "devices": [
          "/dev/disk/by-partlabel/boot-1",
          "/dev/disk/by-partlabel/boot-2"
        ],
        "level": "raid1",
        "name": "md-boot",
        "options": [
          "--metadata=1.0"
        ]
      },
      {
        "devices": [
          "/dev/disk/by-partlabel/root-1",
          "/dev/disk/by-partlabel/root-2"
        ],
        "level": "raid1",
        "name": "md-root"
      }
    ]
  }
}
```

FCCT assigns a unique serial number to each disk and appends it to the label assigned to each disk partition.  This ensures each partition can be referenced by a unique path in `/dev/disk/by-partlabel`.

When desugaring, FCCT should be careful to allow explicit settings elsewhere in the config to override settings originating from sugar.  This could be used e.g. to specify filesystem formatting options.

FCCT also hardcodes the partition size for each partition, replicating the existing partition layout.  If we change the default partition sizes in future, we can encode them in a new layout, and we can also change the default layout if we bump the FCC major version.

The original boot disk is not treated specially; it's wiped and rewritten just like any other replica.

Each of root, boot, ESP, and BIOS-BOOT is placed on a separate partition.  MD-RAID volumes are partitionable, so in principle we could put root and boot on the same RAID volume.  However, this is uncommon (neither Anaconda nor Ignition currently allow it) and it complicates GRUB access to the boot filesystem.

### LUKS and RAID

```json
{
  "ignition": {
    "version": "3.2.0"
  },
  "storage": {
    "disks": [
      {
        "device": "/dev/vda",
        "partitions": [
          {
            "label": "bios-1",
            "sizeMiB": 1,
            "typeGuid": "21686148-6449-6E6F-744E-656564454649"
          },
          {
            "label": "esp-1",
            "sizeMiB": 127,
            "typeGuid": "C12A7328-F81F-11D2-BA4B-00A0C93EC93B"
          },
          {
            "label": "boot-1",
            "sizeMiB": 384
          },
          {
            "label": "root-1"
          }
        ],
        "wipeTable": true
      },
      {
        "device": "/dev/vdb",
        "partitions": [
          {
            "label": "bios-2",
            "sizeMiB": 1,
            "typeGuid": "21686148-6449-6E6F-744E-656564454649"
          },
          {
            "label": "esp-2",
            "sizeMiB": 127,
            "typeGuid": "C12A7328-F81F-11D2-BA4B-00A0C93EC93B"
          },
          {
            "label": "boot-2",
            "sizeMiB": 384
          },
          {
            "label": "root-2"
          }
        ],
        "wipeTable": true
      }
    ],
    "filesystems": [
      {
        "device": "/dev/disk/by-partlabel/esp-1",
        "format": "vfat",
        "label": "esp-1",
        "wipeFilesystem": true
      },
      {
        "device": "/dev/disk/by-partlabel/esp-2",
        "format": "vfat",
        "label": "esp-2",
        "wipeFilesystem": true
      },
      {
        "device": "/dev/md/md-boot",
        "format": "ext4",
        "label": "boot",
        "wipeFilesystem": true
      },
      {
        "device": "/dev/disk/by-id/dm-name-root",
        "format": "xfs",
        "label": "root",
        "wipeFilesystem": true
      }
    ],
    "luks": [
      {
        "clevis": {
          "tpm2": true
        },
        "device": "/dev/md/md-root",
        "label": "luks-root",
        "name": "root",
        "wipeVolume": true
      }
    ],
    "raid": [
      {
        "devices": [
          "/dev/disk/by-partlabel/boot-1",
          "/dev/disk/by-partlabel/boot-2"
        ],
        "level": "raid1",
        "name": "md-boot",
        "options": [
          "--metadata=1.0"
        ]
      },
      {
        "devices": [
          "/dev/disk/by-partlabel/root-1",
          "/dev/disk/by-partlabel/root-2"
        ],
        "level": "raid1",
        "name": "md-root"
      }
    ]
  }
}
```

## Root filesystem

The Dracut glue already saves and restores the root filesystem contents whenever the config specifies a filesystem with label `root` and `wipeFilesystem` set to `true`.

## Boot filesystem

Similar to the root filesystem, the Dracut glue will save and restore the contents of `/boot` whenever the config specifies a filesystem labeled `boot` with `wipeFilesystem` set to `true`.

GRUB includes a module for reading MD-RAID volumes.  However, the shipped boot sector is hardcoded to set `prefix` to the first boot disk, and we want to replicate the boot sector via a bit-for-bit copy (see below).  Therefore, we'll create the RAID volume with metadata format 1.0, which puts the RAID superblock at the end of the partition.  This allows GRUB to treat individual `/boot` replicas as independent filesystems, provided that the RAID module is _not_ preloaded into the BIOS boot partition.  (It currently is not.)

`/boot` replication prevents the use of the grubenv mechanism.  If GRUB treats the replicas as independent filesystems, grubenv will cause the RAID to desynchronize.  Alternatively, if we were to use the GRUB RAID module, GRUB would [disable grubenv support](https://www.gnu.org/software/grub/manual/grub/grub.html#Environment-block).

## EFI System Partition

The Dracut glue will save the contents of the ESP filesystem whenever the config specifies a partition with the ESP type GUID, and will restore those contents to _every_ partition with that type GUID.

Thus the ESP won't be RAIDed; we'll create multiple independent filesystems with identical directory trees.  Keeping each replica independent avoids RAID desynchronization if the firmware chooses to modify the ESP, and makes sense because CoreOS does not modify the ESP after installation.  Since there will no longer be a single canonical ESP, we'll no longer automount `/boot/efi` in the running system (see [coreos/fedora-coreos-tracker#694](https://github.com/coreos/fedora-coreos-tracker/issues/694)).

When updating the bootloader, bootupd will need to find each partition with an ESP type GUID and update it independently.  This assumes that all ESPs on the system are controlled by CoreOS.

Per the usual Ignition philosophy, the copy logic will fail if the source ESP is missing.  If we decide to [drop the ESP in AWS images](https://github.com/coreos/fedora-coreos-config/pull/407) and want to support mirrored boot disks in AWS, we'll need to provide an alternative `layout` without an ESP.

## BIOS Boot partition

The Dracut glue will save an image of the BIOS boot partition whenever the config specifies a partition with the BIOS boot type GUID, and will restore that image to _every_ such partition.  In addition, the same logic will save the boot sector (the first 440 bytes of the disk) and restore it to every disk where we write a BIOS Boot partition.

The BIOS Boot partition is not updated at runtime, so RAID is not necessary.  When updating the bootloader, bootupd will need to find each BIOS Boot type GUID and update both the partition and the corresponding boot sector.  This assumes that the system is not dual-booting and belongs entirely to CoreOS.

For simplicity, we're copying the BIOS Boot partition and boot sector bit-for-bit, rather than rerunning `grub-install` from the initrd or hand-modifying the boot sector.  Since the boot sector hardcodes the offset of the BIOS Boot partition, the latter must be recreated at the same offset as the original partition.  To avoid hardcoding an awkward constant in FCCT, we'll move the BIOS boot partition from its current offset of 512 MiB to the beginning of the disk (offset 1 MiB) in new images.  The Dracut glue will detect relocation of the BIOS boot partition and fail the boot.

There's no BIOS boot partition in 4Kn images.  We could work around this, but to reduce the test matrix, we'll add an empty BIOS boot partition to the 4Kn image.

## PReP partition

For ppc64le, we'll create independent replicas of the PReP partition, similar to the BIOS boot partition.

## Ignition changes

The Dracut glue can detect the requisite Ignition config stanzas using `jq`, but that's brittle and we're already doing a lot of it in the initrd.  Ideally Ignition would provide command-line arguments for executing relevant (hard-coded) queries against the cached Ignition config.  For example:

```sh
ignition -query wiped-filesystem=root
ignition -query partition-type-guid=c12a7328-f81f-11d2-ba4b-00a0c93ec93b
```

We're currently planning to defer this change to future work.

## Architecture support

ppc64le will need an alternate `layout` definition that includes the PReP partition.  aarch64 will similarly need a `layout` without a BIOS Boot partition.

For s390x, Ignition doesn't yet support DASD partitioning, but could work with FCP disks.  In any event, we'd need provisions for rerunning zipl from the initramfs, if RAID support is even possible at all.  Each DASD is connected via a particular bus identifier, which might need to be encoded into each respective copy of the bootloader.  For now, this functionality is out of scope on s390x for 4.7.

## Configuring boot disk RAID in OCP

For 4.7, we'll document a means for OCP users to use FCCT to render a RAID configuration for their Ignition pointer config.  Better OCP UX for FCCs is left as future work.
