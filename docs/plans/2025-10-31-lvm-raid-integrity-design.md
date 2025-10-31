# LVM RAID with Integrity Support Design

**Date:** 2025-10-31
**Status:** Approved

## Overview

Add support for LVM RAID volumes with dm-integrity to installimage, enabling users to configure disk layouts where:
- `/boot` uses mdadm RAID1 for bootloader compatibility
- LVM physical volumes span multiple drives without partition-level mdadm RAID
- LVM logical volumes use native LVM RAID1 with integrity checking

## Requirements

1. Create small 10GB `/boot` partition using mdadm RAID1 with metadata 0.90
2. Remaining space added to LVM using multiple PVs (no mdadm RAID at partition level)
3. Root filesystem (and all LVs) created as LVM RAID1 with integrity enabled
4. New mode coexists with existing SWRAID functionality but operates exclusively when enabled

## Configuration Format

### New Configuration Variables

- `LVMRAID` - Enable LVM RAID mode (0 or 1, default: 0)
- `LVMRAIDLEVEL` - RAID level for LVM logical volumes (0, 1, 5, 6, 10, default: 1)
- `LVMRAIDINTEGRITY` - Enable dm-integrity for LVM RAID (0 or 1, default: 0)

### Example Configuration

```bash
DRIVE1 /dev/sda
DRIVE2 /dev/sdb

SWRAID 1
SWRAIDLEVEL 1

LVMRAID 1
LVMRAIDLEVEL 1
LVMRAIDINTEGRITY 1

BOOTLOADER grub
HOSTNAME myhost

PART /boot ext3 10G
PART lvm space all

LV space root / ext4 256G
LV space swap swap swap 16G
```

## Behavior

### When LVMRAID=1

1. **Boot partition** (`PART /boot`):
   - Creates partitions on all drives (sda1, sdb1)
   - Creates mdadm RAID1 array: `/dev/md/0`
   - Uses existing metadata logic (0.90 for legacy boot compatibility)

2. **LVM partitions** (`PART lvm`):
   - Creates regular type 8e partitions (sda2, sdb2)
   - **NO mdadm RAID array created** (even with SWRAID=1)
   - Each partition becomes a separate physical volume

3. **Volume group creation**:
   - Multiple PVs created: `pvcreate /dev/sda2 /dev/sdb2`
   - VG spans all PVs: `vgcreate space /dev/sda2 /dev/sdb2`

4. **Logical volume creation**:
   - Each LV uses LVM RAID: `lvcreate --type raid1 -m 1 --raidintegrity y --size 256G -n root space`
   - RAID level determined by `LVMRAIDLEVEL`
   - Integrity enabled when `LVMRAIDINTEGRITY=1`

### Interaction with SWRAID

- `SWRAID=1` still creates mdadm arrays for non-LVM partitions (like `/boot`)
- `LVMRAID=1` prevents mdadm array creation for `PART lvm` entries
- Both can be enabled simultaneously - they control different partition types

## Implementation

### Code Modifications

#### 1. functions.sh - read_vars() (after line ~793)

Add configuration variable parsing:

```bash
# LVM RAID configuration
LVMRAID="$(grep -m1 -e ^LVMRAID "$1" | awk '{print $2}')"
[ "$LVMRAID" = "" ] && LVMRAID="0"

LVMRAIDLEVEL="$(grep -m1 -e ^LVMRAIDLEVEL "$1" | awk '{print $2}')"
[ "$LVMRAIDLEVEL" = "" ] && LVMRAIDLEVEL="1"

LVMRAIDINTEGRITY="$(grep -m1 -e ^LVMRAIDINTEGRITY "$1" | awk '{print $2}')"
[ "$LVMRAIDINTEGRITY" = "" ] && LVMRAIDINTEGRITY="0"

# Export for use in other functions
export LVMRAID LVMRAIDLEVEL LVMRAIDINTEGRITY
```

#### 2. functions.sh - create_partitions() (around line 2040)

Modify partition type assignment to skip RAID type for LVM partitions when LVMRAID enabled:

```bash
if [[ "$SWRAID" -eq "1" && "${PART_FS[$i]}" != "esp" ]]; then
  # Skip setting raid type for LVM partitions when LVMRAID is enabled
  if [[ "$LVMRAID" -eq "1" && "${PART_MOUNT[$i]}" = "lvm" ]]; then
    SFDISKTYPE="8e"  # Regular LVM type, not RAID
  else
    SFDISKTYPE="fd"  # RAID type for other partitions
  fi
fi
```

#### 3. functions.sh - make_raid() (around line 2350)

Skip mdadm array creation for LVM partitions when LVMRAID enabled:

```bash
while read -r line; do
  # Skip LVM partitions when LVMRAID is enabled
  if [[ "$LVMRAID" -eq "1" && -n "$(echo "$line" | grep "LVM")" ]]; then
    debug "# Skipping mdadm for LVM partition (LVMRAID enabled)"
    continue
  fi

  # ... existing mdadm array creation logic ...
done < $fstab.tmp
```

#### 4. functions.sh - make_lvm() (around line 2409)

Major modifications for multiple PV support and RAID LV creation:

**a) PV Creation (around line 2461):**

```bash
# Read the lines from fstab and create PVs on all drives
inc_dev=1
while read -r line; do
  if [ -n "$(echo "$line" | grep "LVM")" ]; then
    vg_name="$(echo "$line" | grep "LVM" | awk '{print $2}')"
    partnum="$(echo "$line" | grep "LVM" | awk '{print $1}' | rev | cut -c1)"

    # When LVMRAID is enabled, create PVs on all drives
    if [ "$LVMRAID" -eq "1" ]; then
      pv_list=""
      for n in $(seq 1 $COUNT_DRIVES); do
        TARGETDISK="$(eval echo \$DRIVE${n})"
        local p; p=""
        local nvme; nvme="$(echo $TARGETDISK | grep nvme)"
        [ -n "$nvme" ] && p='p'
        local disk_by; disk_by="$(echo $TARGETDISK | grep '^/dev/disk/by-')"
        [ -n "$disk_by" ] && p='-part'

        pv_device="$TARGETDISK$p$partnum"
        debug "# Creating PV $pv_device for VG $vg_name"
        wipefs -af $pv_device |& debugoutput
        pvcreate -ff $pv_device 2>&1 | debugoutput
        pv_list="$pv_list $pv_device"
      done
      dev[$inc_dev]="$pv_list"
    else
      # Original single PV behavior
      pv="$(echo "$line" | grep "LVM" | awk '{print $2}')"
      dev[$inc_dev]="$pv"
      debug "# Creating PV $pv"
      wipefs -af $pv |& debugoutput
      pvcreate -ff $pv 2>&1 | debugoutput
    fi
    inc_dev=$(( ${inc_dev} + 1 ))
  fi
done < $fstab
```

**b) VG Creation (around line 2477):**

```bash
# create VGs
for i in $(seq 1 $LVM_VG_COUNT) ; do
  vg=${LVM_VG_NAME[$i]}
  pvs=${dev[${i}]}  # May be space-separated list when LVMRAID=1

  # extend the VG if a VG with the same name already exists
  if [ "$(vgs --noheadings 2>/dev/null | grep "$vg")" ]; then
    debug "# Extending VG $vg with PV(s) $pvs"
    vgextend $vg $pvs 2>&1 | debugoutput
  else
    debug "# Creating VG $vg with PV(s) $pvs"
    [ "$vg" ] && rm -rf "/dev/$vg" 2>&1 | debugoutput
    vgcreate $vg $pvs 2>&1 | debugoutput
  fi
done
```

**c) LV Creation (around line 2493):**

```bash
# create LVs
for i in $(seq 1 $LVM_LV_COUNT) ; do
  lv=${LVM_LV_NAME[$i]}
  vg=${LVM_LV_VG[$i]}
  size=${LVM_LV_SIZE[$i]}
  vg_last_lv=''
  free=''

  # ... existing size calculation logic ...

  # Create LV with or without RAID
  if [ "$LVMRAID" -eq "1" ]; then
    local mirrors=$((COUNT_DRIVES - 1))
    local raid_opts="--type raid${LVMRAIDLEVEL} -m ${mirrors}"

    if [ "$LVMRAIDINTEGRITY" -eq "1" ]; then
      raid_opts="$raid_opts --raidintegrity y"
    fi

    debug "# Creating LV $vg/$lv ($size MiB) with RAID options: $raid_opts"
    lvcreate --yes $raid_opts --name $lv --size $size $vg 2>&1 | debugoutput
  else
    debug "# Creating LV $vg/$lv ($size MiB)"
    lvcreate --yes --name $lv --size $size $vg 2>&1 | debugoutput
  fi

  test $? -eq 0 || return 1
done
```

## Testing Strategy

### Test Configuration Files

Create test configs in `configs/` directory:

1. **configs/test-lvmraid-basic** - Basic LVM RAID1 with integrity
2. **configs/test-lvmraid-nointegrity** - LVM RAID1 without integrity
3. **configs/test-lvmraid-mixed** - Mix of mdadm /boot and LVM RAID root

### Validation Steps

1. Verify partition types are correct (8e for LVM, not fd when LVMRAID=1)
2. Verify mdadm arrays only created for non-LVM partitions
3. Verify multiple PVs created correctly
4. Verify VG contains all PVs
5. Verify LV uses RAID type with correct parameters
6. Verify integrity is enabled when requested
7. Verify system boots correctly from mdadm /boot and LVM RAID root

## Risks and Limitations

1. **LVM RAID Performance**: Initial sync of RAID volumes may take time; consider `--nosync` for testing
2. **Integrity Overhead**: dm-integrity adds write latency and storage overhead (~4GB per 100GB)
3. **Bootloader Compatibility**: GRUB cannot boot from LVM RAID, hence separate mdadm /boot required
4. **Recovery Complexity**: LVM RAID recovery is more complex than mdadm RAID recovery
5. **Kernel Support**: Requires kernel with dm-integrity support (Linux 4.12+)

## Future Enhancements

1. Per-LV RAID configuration (extend LV syntax)
2. Support for RAID5/6/10 at LVM level
3. Configurable integrity block size
4. Support for existing VG preservation with LVMRAID

## References

- LVM RAID: https://man7.org/linux/man-pages/man7/lvmraid.7.html
- dm-integrity: https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/dm-integrity.html
- mdadm metadata formats: https://raid.wiki.kernel.org/index.php/RAID_superblock_formats
