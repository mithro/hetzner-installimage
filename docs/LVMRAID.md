# LVM RAID with Integrity

## Overview

LVM RAID mode enables creating disk layouts with:
- mdadm RAID for `/boot` (bootloader compatibility)
- Multiple LVM physical volumes across drives (no partition-level RAID)
- LVM RAID volumes with optional dm-integrity

## Configuration

### Variables

- `LVMRAID` - Enable LVM RAID mode (0 or 1, default: 0)
- `LVMRAIDLEVEL` - RAID level (0, 1, 5, 6, 10, default: 1)
- `LVMRAIDINTEGRITY` - Enable dm-integrity (0 or 1, default: 0)

### Example

```bash
DRIVE1 /dev/sda
DRIVE2 /dev/sdb

SWRAID 1          # mdadm for /boot
SWRAIDLEVEL 1

LVMRAID 1         # LVM RAID for LVs
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

When `LVMRAID=1`:

1. **Boot partition**: Creates mdadm RAID1 array `/dev/md/0`
2. **LVM partitions**: Created as type 8e (LVM), NOT fd (RAID)
3. **Physical volumes**: Multiple PVs created across all drives
4. **Volume group**: Spans all PVs (e.g., `/dev/sda2 /dev/sdb2`)
5. **Logical volumes**: Created with `--type raidN -m M --raidintegrity y`

## Resulting Layout

```
/dev/sda1, /dev/sdb1 → /dev/md/0 → /boot (mdadm RAID1)
/dev/sda2 → PV
/dev/sdb2 → PV
           ↓
        VG "space"
           ↓
    ┌──────┴──────┐
    ↓             ↓
 LV root      LV swap
(RAID1+int)  (RAID1+int)
```

## Testing

### Manual Testing (Requires Actual Hardware)

**CAUTION**: This will destroy data on target drives!

1. Boot into rescue system
2. Copy modified installimage to rescue system
3. Run with test config:

```bash
./installimage -c test-lvmraid-debian -a
```

4. Verify during installation:
   - Check partition types: `parted /dev/sda print`
   - Check PVs: `pvs` (should show sda2 and sdb2)
   - Check VG: `vgs` (should show both PVs)
   - Check LVs: `lvs -a` (should show raid1 type)
   - Check integrity: `lvs -o +integritymismatches,integrity_data_sectors`

5. After installation:
   - Boot the system
   - Verify RAID status: `lvs -a -o +raid_sync_action,raid_mismatch_count`
   - Test integrity: Write/read test files

### Validation Commands

```bash
# Partition types (should be 8e for LVM, not fd)
parted /dev/sda print | grep "raid\|lvm"

# PVs (should show both drives)
pvs

# VG (should span both PVs)
vgs -o +pv_name

# LVs (should show raid1 type)
lvs -a -o +segtype,devices

# Integrity status
lvs -o +integritymismatches
```

## Troubleshooting

### LV creation fails

```
Insufficient suitable allocatable extents for logical volume
```

**Solution**: Reduce LV size or check VG free space with `vgs`

### Integrity overhead

dm-integrity adds ~4% storage overhead and write latency.

**Solution**: Set `LVMRAIDINTEGRITY=0` if performance is critical

### Boot fails

GRUB cannot boot from LVM RAID directly.

**Solution**: Ensure `/boot` is on mdadm RAID1 (not LVM)

## Requirements

- Linux kernel 4.12+ (dm-integrity support)
- LVM2 with RAID support
- mdadm for `/boot` RAID

## References

- Design doc: `docs/plans/2025-10-31-lvm-raid-integrity-design.md`
- LVM RAID: https://man7.org/linux/man-pages/man7/lvmraid.7.html
- dm-integrity: https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/dm-integrity.html
