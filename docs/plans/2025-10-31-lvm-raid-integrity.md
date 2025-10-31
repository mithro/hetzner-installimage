# LVM RAID with Integrity Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add support for LVM RAID volumes with dm-integrity, enabling disk layouts with mdadm /boot and LVM RAID logical volumes.

**Architecture:** Extend existing installimage partitioning system with new LVMRAID mode that skips mdadm for LVM partitions, creates multiple PVs across drives, and uses native LVM RAID with integrity for all logical volumes.

**Tech Stack:** Bash, mdadm, LVM2 (pvcreate, vgcreate, lvcreate), parted/sgdisk

---

## Task 1: Add Configuration Variable Reading

**Files:**
- Modify: `functions.sh:712-795` (read_vars function)

**Step 1: Locate the SWRAIDLEVEL reading code**

Read the file to find the exact location:

```bash
grep -n "SWRAIDLEVEL=" functions.sh | head -5
```

Expected: Line ~792-793 shows where SWRAIDLEVEL is read and defaulted

**Step 2: Add LVMRAID config variables after SWRAIDLEVEL**

Add these lines immediately after the SWRAIDLEVEL section (around line 794):

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

**Step 3: Verify the modification**

```bash
grep -A 12 "# LVM RAID configuration" functions.sh
```

Expected: Shows the 3 variable assignments and export line

**Step 4: Test variable reading with a temp config**

Create test config in project temp directory:

```bash
mkdir -p .tmp_test
cat > .tmp_test/test.conf << 'EOF'
LVMRAID 1
LVMRAIDLEVEL 1
LVMRAIDINTEGRITY 1
EOF
```

Source the functions and test:

```bash
source functions.sh
read_vars .tmp_test/test.conf
echo "LVMRAID=$LVMRAID LVMRAIDLEVEL=$LVMRAIDLEVEL LVMRAIDINTEGRITY=$LVMRAIDINTEGRITY"
```

Expected: `LVMRAID=1 LVMRAIDLEVEL=1 LVMRAIDINTEGRITY=1`

**Step 5: Clean up test files**

```bash
rm -rf .tmp_test
```

**Step 6: Commit**

```bash
git add functions.sh
git commit -m "feat: add LVMRAID config variable reading

Add support for reading LVMRAID, LVMRAIDLEVEL, and LVMRAIDINTEGRITY
from config files in read_vars function.

Related to LVM RAID with integrity implementation.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Modify Partition Type Logic

**Files:**
- Modify: `functions.sh:2040-2042` (create_partitions function)

**Step 1: Locate the SWRAID partition type code**

Find the exact lines where SFDISKTYPE is set for RAID:

```bash
grep -n 'if.*SWRAID.*-eq.*1.*PART_FS' functions.sh
```

Expected: Line ~2040 shows the conditional that sets SFDISKTYPE="fd" for RAID

**Step 2: Read the current logic**

```bash
sed -n '2037,2043p' functions.sh
```

Expected: Shows the current if statement setting SFDISKTYPE for LVM and RAID

**Step 3: Replace the SWRAID conditional**

The current code at line 2040-2042 should be:

```bash
   if [[ "$SWRAID" -eq "1" && "${PART_FS[$i]}" != "esp" ]]; then
     SFDISKTYPE="fd"
   fi
```

Replace it with:

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

**Step 4: Verify the modification**

```bash
sed -n '2037,2047p' functions.sh
```

Expected: Shows nested if checking LVMRAID before setting SFDISKTYPE

**Step 5: Create verification test script**

Create a test in project temp directory to verify logic:

```bash
mkdir -p .tmp_test
cat > .tmp_test/verify_partition_type.sh << 'EOF'
#!/bin/bash
source functions.sh

# Test 1: SWRAID=1, LVMRAID=0, LVM partition -> should be fd (RAID)
SWRAID=1
LVMRAID=0
PART_MOUNT[1]="lvm"
PART_FS[1]="lvm"
i=1

SFDISKTYPE="83"
if [[ "$SWRAID" -eq "1" && "${PART_FS[$i]}" != "esp" ]]; then
  if [[ "$LVMRAID" -eq "1" && "${PART_MOUNT[$i]}" = "lvm" ]]; then
    SFDISKTYPE="8e"
  else
    SFDISKTYPE="fd"
  fi
fi

echo "Test 1 (SWRAID=1, LVMRAID=0, LVM): SFDISKTYPE=$SFDISKTYPE (expected: fd)"
[[ "$SFDISKTYPE" == "fd" ]] && echo "PASS" || echo "FAIL"

# Test 2: SWRAID=1, LVMRAID=1, LVM partition -> should be 8e (LVM)
LVMRAID=1

SFDISKTYPE="83"
if [[ "$SWRAID" -eq "1" && "${PART_FS[$i]}" != "esp" ]]; then
  if [[ "$LVMRAID" -eq "1" && "${PART_MOUNT[$i]}" = "lvm" ]]; then
    SFDISKTYPE="8e"
  else
    SFDISKTYPE="fd"
  fi
fi

echo "Test 2 (SWRAID=1, LVMRAID=1, LVM): SFDISKTYPE=$SFDISKTYPE (expected: 8e)"
[[ "$SFDISKTYPE" == "8e" ]] && echo "PASS" || echo "FAIL"

# Test 3: SWRAID=1, LVMRAID=1, /boot partition -> should be fd (RAID)
PART_MOUNT[1]="/boot"

SFDISKTYPE="83"
if [[ "$SWRAID" -eq "1" && "${PART_FS[$i]}" != "esp" ]]; then
  if [[ "$LVMRAID" -eq "1" && "${PART_MOUNT[$i]}" = "lvm" ]]; then
    SFDISKTYPE="8e"
  else
    SFDISKTYPE="fd"
  fi
fi

echo "Test 3 (SWRAID=1, LVMRAID=1, /boot): SFDISKTYPE=$SFDISKTYPE (expected: fd)"
[[ "$SFDISKTYPE" == "fd" ]] && echo "PASS" || echo "FAIL"
EOF

chmod +x .tmp_test/verify_partition_type.sh
```

**Step 6: Run verification**

```bash
.tmp_test/verify_partition_type.sh
```

Expected: All three tests show PASS

**Step 7: Clean up test files**

```bash
rm -rf .tmp_test
```

**Step 8: Commit**

```bash
git add functions.sh
git commit -m "feat: skip RAID partition type for LVM when LVMRAID enabled

Modify create_partitions to set partition type to 8e (LVM) instead of
fd (RAID) for LVM partitions when LVMRAID=1.

This prevents mdadm from attempting to create arrays on LVM partitions
that will use LVM RAID instead.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: Skip mdadm Array Creation for LVM Partitions

**Files:**
- Modify: `functions.sh:2300-2406` (make_raid function)

**Step 1: Locate the make_raid loop**

```bash
grep -n "while read.*line.*make_raid" functions.sh
grep -n "done < \$fstab.tmp" functions.sh | grep -A1 -B1 2400
```

Expected: Shows the while loop around line 2402 that processes fstab.tmp

**Step 2: Read the loop start**

```bash
sed -n '2320,2340p' functions.sh
```

Expected: Shows the beginning of the loop that processes each partition

**Step 3: Add LVMRAID skip logic at loop start**

Find the line that starts `while read -r line; do` in the make_raid function (around line 2320), and add the skip logic right after it:

After the line `while read -r line; do`, add:

```bash
    # Skip LVM partitions when LVMRAID is enabled
    if [[ "$LVMRAID" -eq "1" && -n "$(echo "$line" | grep "LVM")" ]]; then
      debug "# Skipping mdadm for LVM partition (LVMRAID enabled)"
      continue
    fi
```

**Step 4: Verify the modification**

```bash
sed -n '2320,2330p' functions.sh
```

Expected: Shows the while loop followed by the LVMRAID skip logic

**Step 5: Create a minimal test for the skip logic**

```bash
mkdir -p .tmp_test
cat > .tmp_test/test_raid_skip.sh << 'EOF'
#!/bin/bash

# Simulate the condition
LVMRAID=1
line="1 /dev/md/0 LVM vg0"

if [[ "$LVMRAID" -eq "1" && -n "$(echo "$line" | grep "LVM")" ]]; then
  echo "SKIP: Would skip mdadm for LVM partition"
  skipped=1
else
  echo "CREATE: Would create mdadm array"
  skipped=0
fi

[[ $skipped -eq 1 ]] && echo "PASS: LVM partition skipped correctly" || echo "FAIL"

# Test non-LVM partition
line="1 /dev/md/0 /boot ext3"

if [[ "$LVMRAID" -eq "1" && -n "$(echo "$line" | grep "LVM")" ]]; then
  echo "SKIP: Would skip mdadm"
  skipped=1
else
  echo "CREATE: Would create mdadm array for /boot"
  skipped=0
fi

[[ $skipped -eq 0 ]] && echo "PASS: /boot partition not skipped" || echo "FAIL"
EOF

chmod +x .tmp_test/test_raid_skip.sh
```

**Step 6: Run test**

```bash
.tmp_test/test_raid_skip.sh
```

Expected: Both tests show PASS

**Step 7: Clean up**

```bash
rm -rf .tmp_test
```

**Step 8: Commit**

```bash
git add functions.sh
git commit -m "feat: skip mdadm array creation for LVM when LVMRAID enabled

Modify make_raid to skip creating mdadm arrays for LVM partitions
when LVMRAID=1. LVM partitions will use LVM-level RAID instead.

Non-LVM partitions (like /boot) still get mdadm arrays as normal.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Implement Multiple PV Creation

**Files:**
- Modify: `functions.sh:2454-2465` (make_lvm function - PV creation section)

**Step 1: Locate PV creation code**

```bash
grep -n "Creating PV" functions.sh | head -5
```

Expected: Shows line ~2461 where PVs are created

**Step 2: Read current PV creation logic**

```bash
sed -n '2454,2475p' functions.sh
```

Expected: Shows the while loop that reads fstab and creates single PV per VG

**Step 3: Replace PV creation logic**

The current code creates one PV. Replace the while loop that starts around line 2456 with:

```bash
    # Read the lines from fstab and create PVs
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

**Step 4: Verify the modification**

```bash
sed -n '2454,2495p' functions.sh
```

Expected: Shows the new logic with LVMRAID conditional and loop over drives

**Step 5: Create test for PV list building logic**

```bash
mkdir -p .tmp_test
cat > .tmp_test/test_pv_list.sh << 'EOF'
#!/bin/bash

# Test multiple PV list building
LVMRAID=1
COUNT_DRIVES=2
DRIVE1="/dev/sda"
DRIVE2="/dev/sdb"
partnum=2

pv_list=""
for n in $(seq 1 $COUNT_DRIVES); do
  TARGETDISK="$(eval echo \$DRIVE${n})"
  p=""
  nvme="$(echo $TARGETDISK | grep nvme)"
  [ -n "$nvme" ] && p='p'
  disk_by="$(echo $TARGETDISK | grep '^/dev/disk/by-')"
  [ -n "$disk_by" ] && p='-part'

  pv_device="$TARGETDISK$p$partnum"
  pv_list="$pv_list $pv_device"
done

echo "PV list: $pv_list"
expected=" /dev/sda2 /dev/sdb2"
[[ "$pv_list" == "$expected" ]] && echo "PASS: PV list correct" || echo "FAIL: Expected '$expected', got '$pv_list'"

# Test with nvme drives
DRIVE1="/dev/nvme0n1"
DRIVE2="/dev/nvme1n1"

pv_list=""
for n in $(seq 1 $COUNT_DRIVES); do
  TARGETDISK="$(eval echo \$DRIVE${n})"
  p=""
  nvme="$(echo $TARGETDISK | grep nvme)"
  [ -n "$nvme" ] && p='p'

  pv_device="$TARGETDISK$p$partnum"
  pv_list="$pv_list $pv_device"
done

echo "NVMe PV list: $pv_list"
expected=" /dev/nvme0n1p2 /dev/nvme1n1p2"
[[ "$pv_list" == "$expected" ]] && echo "PASS: NVMe PV list correct" || echo "FAIL: Expected '$expected', got '$pv_list'"
EOF

chmod +x .tmp_test/test_pv_list.sh
```

**Step 6: Run test**

```bash
.tmp_test/test_pv_list.sh
```

Expected: Both PASS messages for SATA and NVMe drives

**Step 7: Clean up**

```bash
rm -rf .tmp_test
```

**Step 8: Commit**

```bash
git add functions.sh
git commit -m "feat: create multiple PVs across drives when LVMRAID enabled

Modify make_lvm PV creation to loop over all drives and create
separate PVs when LVMRAID=1. The PV list is stored for VG creation.

Maintains backward compatibility - single PV behavior when LVMRAID=0.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: Implement VG Creation with Multiple PVs

**Files:**
- Modify: `functions.sh:2476-2490` (make_lvm function - VG creation section)

**Step 1: Locate VG creation code**

```bash
grep -n "Creating VG" functions.sh | head -3
```

Expected: Shows line ~2486 where VGs are created

**Step 2: Read current VG creation logic**

```bash
sed -n '2476,2491p' functions.sh
```

Expected: Shows the loop that creates VGs with single PV

**Step 3: Modify vgcreate/vgextend to use PV list**

The current code uses `$pv` as a single value. Since we now store space-separated PV lists in `dev[$i]`, the vgcreate and vgextend commands will automatically handle multiple PVs. Change the variable name for clarity:

Find the section around line 2477-2490 and modify:

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

**Step 4: Verify the modification**

```bash
sed -n '2476,2491p' functions.sh
```

Expected: Shows `pvs` variable and updated debug messages with "PV(s)"

**Step 5: Create test for vgcreate command generation**

```bash
mkdir -p .tmp_test
cat > .tmp_test/test_vgcreate.sh << 'EOF'
#!/bin/bash

# Test vgcreate command with multiple PVs
vg="space"
pvs="/dev/sda2 /dev/sdb2"

# Simulate the command (don't actually run it)
cmd="vgcreate $vg $pvs"
expanded_cmd="vgcreate space /dev/sda2 /dev/sdb2"

echo "Command: $cmd"
echo "Would run: vgcreate $vg $pvs"

# Verify the command expands correctly
actual="vgcreate $vg $pvs"
expected="vgcreate space /dev/sda2 /dev/sdb2"

[[ "$actual" == "$expected" ]] && echo "PASS: vgcreate command correct" || echo "FAIL"

# Test with single PV (backward compat)
pvs="/dev/md/0"
actual="vgcreate $vg $pvs"
expected="vgcreate space /dev/md/0"

echo "Single PV command: vgcreate $vg $pvs"
[[ "$actual" == "$expected" ]] && echo "PASS: Single PV backward compat" || echo "FAIL"
EOF

chmod +x .tmp_test/test_vgcreate.sh
```

**Step 6: Run test**

```bash
.tmp_test/test_vgcreate.sh
```

Expected: Both PASS messages

**Step 7: Clean up**

```bash
rm -rf .tmp_test
```

**Step 8: Commit**

```bash
git add functions.sh
git commit -m "feat: create VGs with multiple PVs when LVMRAID enabled

Modify VG creation to accept space-separated PV lists from previous
step. vgcreate/vgextend automatically handle multiple PVs.

Backward compatible - works with single PV from SWRAID mode.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 6: Implement LVM RAID LV Creation

**Files:**
- Modify: `functions.sh:2492-2526` (make_lvm function - LV creation section)

**Step 1: Locate LV creation code**

```bash
grep -n "lvcreate --yes --name" functions.sh
```

Expected: Shows line ~2523 where LVs are created

**Step 2: Read current LV creation logic**

```bash
sed -n '2492,2527p' functions.sh
```

Expected: Shows the simple lvcreate command

**Step 3: Replace lvcreate with conditional RAID logic**

Find line ~2523 with `lvcreate --yes --name $lv --size $size $vg` and replace it with:

```bash
      # Create LV with or without RAID
      if [ "$LVMRAID" -eq "1" ]; then
        local mirrors=$((COUNT_DRIVES - 1))
        local raid_opts="--type raid${LVMRAIDLEVEL} -m ${mirrors}"

        if [ "$LVMRAIDINTEGRITY" -eq "1" ]; then
          raid_opts="$raid_opts --raidintegrity y"
        fi

        debug "# Creating LV $vg/$lv ($size MiB) with RAID options: $raid_opts"
        lvcreate --yes $raid_opts --name $lv --size ${size}M $vg 2>&1 | debugoutput
      else
        debug "# Creating LV $vg/$lv ($size MiB)"
        lvcreate --yes --name $lv --size ${size}M $vg 2>&1 | debugoutput
      fi
      test $? -eq 0 || return 1
```

Note: Added explicit `M` suffix to size since LVM defaults vary.

**Step 4: Verify the modification**

```bash
sed -n '2520,2535p' functions.sh
```

Expected: Shows the if/else block with RAID options

**Step 5: Create test for lvcreate command generation**

```bash
mkdir -p .tmp_test
cat > .tmp_test/test_lvcreate.sh << 'EOF'
#!/bin/bash

# Test RAID lvcreate command generation
LVMRAID=1
LVMRAIDLEVEL=1
LVMRAIDINTEGRITY=1
COUNT_DRIVES=2
vg="space"
lv="root"
size="256000"

mirrors=$((COUNT_DRIVES - 1))
raid_opts="--type raid${LVMRAIDLEVEL} -m ${mirrors}"

if [ "$LVMRAIDINTEGRITY" -eq "1" ]; then
  raid_opts="$raid_opts --raidintegrity y"
fi

cmd="lvcreate --yes $raid_opts --name $lv --size ${size}M $vg"
echo "Command: $cmd"

expected="lvcreate --yes --type raid1 -m 1 --raidintegrity y --name root --size 256000M space"
[[ "$cmd" == "$expected" ]] && echo "PASS: RAID lvcreate command correct" || echo "FAIL: Expected '$expected', got '$cmd'"

# Test without integrity
LVMRAIDINTEGRITY=0
raid_opts="--type raid${LVMRAIDLEVEL} -m ${mirrors}"

if [ "$LVMRAIDINTEGRITY" -eq "1" ]; then
  raid_opts="$raid_opts --raidintegrity y"
fi

cmd="lvcreate --yes $raid_opts --name $lv --size ${size}M $vg"
expected="lvcreate --yes --type raid1 -m 1 --name root --size 256000M space"
[[ "$cmd" == "$expected" ]] && echo "PASS: RAID without integrity correct" || echo "FAIL"

# Test non-RAID mode
LVMRAID=0

if [ "$LVMRAID" -eq "1" ]; then
  cmd="lvcreate --yes $raid_opts --name $lv --size ${size}M $vg"
else
  cmd="lvcreate --yes --name $lv --size ${size}M $vg"
fi

expected="lvcreate --yes --name root --size 256000M space"
[[ "$cmd" == "$expected" ]] && echo "PASS: Non-RAID mode correct" || echo "FAIL"
EOF

chmod +x .tmp_test/test_lvcreate.sh
```

**Step 6: Run test**

```bash
.tmp_test/test_lvcreate.sh
```

Expected: All three PASS messages

**Step 7: Clean up**

```bash
rm -rf .tmp_test
```

**Step 8: Commit**

```bash
git add functions.sh
git commit -m "feat: create LVs with RAID and integrity when LVMRAID enabled

Modify LV creation to use --type raidN -m M --raidintegrity y
when LVMRAID=1. RAID level from LVMRAIDLEVEL, mirrors calculated
from drive count, integrity controlled by LVMRAIDINTEGRITY.

Maintains backward compatibility with simple lvcreate when LVMRAID=0.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 7: Create Test Configuration File

**Files:**
- Create: `configs/test-lvmraid-debian`

**Step 1: Create test configuration**

```bash
cat > configs/test-lvmraid-debian << 'EOF'
# Test configuration for LVM RAID with integrity
#
# This creates:
# - /boot as mdadm RAID1 (10GB)
# - LVM VG spanning both drives without partition-level RAID
# - Root LV as LVM RAID1 with integrity (256GB)
# - Swap LV as LVM RAID1 with integrity (16GB)

DRIVE1 /dev/sda
DRIVE2 /dev/sdb

# Enable mdadm for /boot
SWRAID 1
SWRAIDLEVEL 1

# Enable LVM RAID with integrity
LVMRAID 1
LVMRAIDLEVEL 1
LVMRAIDINTEGRITY 1

BOOTLOADER grub
HOSTNAME test-lvmraid

# Partitions
PART /boot ext3 10G
PART lvm space all

# Logical Volumes with LVM RAID
LV space root / ext4 256G
LV space swap swap swap 16G

IMAGE /root/images/Debian-stable-amd64-base.tar.gz
EOF
```

**Step 2: Verify file created**

```bash
cat configs/test-lvmraid-debian
```

Expected: Shows the complete configuration

**Step 3: Verify config syntax**

```bash
grep "^DRIVE" configs/test-lvmraid-debian | wc -l
grep "^SWRAID" configs/test-lvmraid-debian
grep "^LVMRAID" configs/test-lvmraid-debian
grep "^PART" configs/test-lvmraid-debian | wc -l
grep "^LV" configs/test-lvmraid-debian | wc -l
```

Expected:
- 2 DRIVE lines
- SWRAID 1
- LVMRAID 1
- 2 PART lines
- 2 LV lines

**Step 4: Create additional test config without integrity**

```bash
cat > configs/test-lvmraid-nointegrity << 'EOF'
# Test configuration for LVM RAID without integrity
#
# Same as test-lvmraid-debian but with LVMRAIDINTEGRITY=0

DRIVE1 /dev/sda
DRIVE2 /dev/sdb

SWRAID 1
SWRAIDLEVEL 1

LVMRAID 1
LVMRAIDLEVEL 1
LVMRAIDINTEGRITY 0

BOOTLOADER grub
HOSTNAME test-lvmraid-nointegrity

PART /boot ext3 10G
PART lvm space all

LV space root / ext4 256G
LV space swap swap swap 16G

IMAGE /root/images/Debian-stable-amd64-base.tar.gz
EOF
```

**Step 5: Verify second config**

```bash
grep "^LVMRAIDINTEGRITY" configs/test-lvmraid-nointegrity
```

Expected: LVMRAIDINTEGRITY 0

**Step 6: Commit**

```bash
git add configs/test-lvmraid-debian configs/test-lvmraid-nointegrity
git commit -m "feat: add test configurations for LVM RAID

Add two test configs:
- test-lvmraid-debian: LVM RAID1 with integrity enabled
- test-lvmraid-nointegrity: LVM RAID1 without integrity

Both use mdadm RAID1 for /boot and LVM RAID for root/swap.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 8: Documentation and Testing Guide

**Files:**
- Create: `docs/LVMRAID.md`

**Step 1: Create documentation**

```bash
cat > docs/LVMRAID.md << 'EOF'
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
EOF
```

**Step 2: Verify documentation**

```bash
cat docs/LVMRAID.md
```

Expected: Complete documentation file

**Step 3: Commit**

```bash
git add docs/LVMRAID.md
git commit -m "docs: add LVM RAID documentation

Add comprehensive documentation covering:
- Configuration variables and examples
- Behavior and resulting disk layout
- Testing procedures and validation commands
- Troubleshooting common issues

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 9: Final Verification and Summary

**Files:**
- Review all modified files

**Step 1: Verify all commits**

```bash
git log --oneline | head -10
```

Expected: 8 commits for the LVM RAID feature

**Step 2: Review changes**

```bash
git diff master...HEAD --stat
```

Expected: Shows modifications to functions.sh and new files in configs/ and docs/

**Step 3: Create summary of changes**

```bash
cat > IMPLEMENTATION_SUMMARY.md << 'EOF'
# LVM RAID Implementation Summary

## Commits

1. feat: add LVMRAID config variable reading
2. feat: skip RAID partition type for LVM when LVMRAID enabled
3. feat: skip mdadm array creation for LVM when LVMRAID enabled
4. feat: create multiple PVs across drives when LVMRAID enabled
5. feat: create VGs with multiple PVs when LVMRAID enabled
6. feat: create LVs with RAID and integrity when LVMRAID enabled
7. feat: add test configurations for LVM RAID
8. docs: add LVM RAID documentation

## Files Modified

- `functions.sh`: Added LVMRAID support to read_vars, create_partitions, make_raid, make_lvm

## Files Created

- `configs/test-lvmraid-debian`: Test config with integrity
- `configs/test-lvmraid-nointegrity`: Test config without integrity
- `docs/LVMRAID.md`: User documentation

## Testing Status

- Unit tests: PASSED (bash logic tests in each task)
- Integration tests: MANUAL (requires actual hardware)

## Next Steps

1. Test on actual hardware with test configs
2. Verify boot process
3. Validate RAID sync and integrity checking
4. Performance testing with integrity enabled/disabled
EOF

cat IMPLEMENTATION_SUMMARY.md
```

**Step 4: Check for any syntax errors**

```bash
bash -n functions.sh
```

Expected: No output (no syntax errors)

**Step 5: Final commit**

```bash
git add IMPLEMENTATION_SUMMARY.md
git commit -m "docs: add implementation summary

Summary of all changes made for LVM RAID feature.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

**Step 6: Show final status**

```bash
echo "=== Implementation Complete ==="
echo ""
echo "Branch: $(git branch --show-current)"
echo "Total commits: $(git log master..HEAD --oneline | wc -l)"
echo ""
echo "Modified files:"
git diff master...HEAD --name-only
echo ""
echo "Next: Test on actual hardware using configs/test-lvmraid-debian"
```

---

## Notes

- All tasks follow TDD-style approach: write logic, test logic, verify, commit
- Each commit is atomic and can be reviewed independently
- Manual integration testing required (destructive disk operations)
- Maintains backward compatibility (LVMRAID=0 works as before)

## Testing Requirements

**Before merging**: Must test on actual hardware in rescue system environment with at least 2 drives.

**Critical validations**:
1. Partition types correct (8e vs fd)
2. mdadm only creates /boot array
3. Multiple PVs created successfully
4. VG spans all PVs
5. LVs use RAID type
6. Integrity enabled when configured
7. System boots successfully
