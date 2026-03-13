# Data Recovery Guide – Western Digital Black SN750 NVMe SSD

## Overview

This guide walks through recovering music files from a Western Digital Black SN750 NVMe M.2 SSD.
The recommended approach is to **image the drive first** and recover from the image, so the original
drive is never modified.

---

## ⚠️ Critical Rules Before You Start

1. **Do NOT let Windows auto-repair or run `chkdsk` on the drive.** If the drive was a Windows
   boot drive, Windows may launch automatic repair on first detection and overwrite recoverable data.
2. **Do NOT write anything to the suspect drive** – not even a tiny file.
3. **Work from a disk image**, not the original drive, whenever possible.
4. **Power cycles matter** – every unnecessary power-on of a degraded SSD increases the chance of
   further data loss. Plan each step before plugging the drive in.

---

## Step 0 – Assess the Situation

Before doing anything, answer these questions to pick the right path:

| Question | Answer leads to |
|---|---|
| Does the drive appear at all in another computer's BIOS/UEFI? | Yes → proceed to Step 1. No → jump to [Professional Recovery](#step-5--professional-recovery-last-resort). |
| Is the drive readable (files visible) when plugged in via the USB enclosure on any OS? | Yes → copy files immediately, then verify. |
| Did Windows start running `chkdsk` or "Repair" automatically? | **Unplug immediately**, use Linux instead. |

### Quick Windows check (only if you accept the risk)

If you want a fast first look without Linux:

1. Plug the USB enclosure into a Windows PC.
2. **Hold Shift** and click "Cancel" immediately if any repair/scan dialog appears.
3. Open File Explorer. If files are visible, **copy them right now** to a separate drive before
   doing anything else.
4. If files are not visible or the drive is unreadable, do not proceed with Windows – switch to the
   Linux approach below.

---

## Step 1 – Boot a Linux Live USB (Recommended Path)

Using Linux is safer because:

- Linux will not auto-run `chkdsk` or filesystem repair.
- You can mount the drive **read-only**, preventing any accidental writes.
- `ddrescue` (the best free imaging tool) is a Linux utility.

### Create a Linux Live USB

1. Download [Ubuntu Desktop](https://ubuntu.com/download/desktop) (any recent LTS release) or
   [Fedora Workstation](https://fedoraproject.org/workstation/).
2. Flash it to a USB stick with [Rufus](https://rufus.ie/) (Windows) or
   [balenaEtcher](https://etcher.balena.io/) (Windows/macOS/Linux).
3. Boot the target computer from that USB stick (usually F12 or F2 at POST to select boot device).
4. Choose **"Try Ubuntu"** (or equivalent) – do NOT install.

---

## Step 2 – Identify the Drive

Open a terminal in the Linux live session and run:

```bash
sudo fdisk -l
```

or

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINT
```

The SN750 connected via USB enclosure will typically appear as `/dev/sdb` or `/dev/sdc`
(not `/dev/sda`, which is usually the live USB itself). Confirm by checking the size matches
the SN750 (e.g., 500 GB, 1 TB).

> **Note the device name** (e.g., `/dev/sdb`) – you'll use it in the next steps.

---

## Step 3 – Create a Disk Image with `ddrescue` *(Most Important Step)*

`gnu-ddrescue` creates a byte-for-byte copy of the drive, skipping unreadable sectors and logging
which sectors were successfully read so it can retry. **This is safer than `dd`.**

### Install ddrescue

```bash
sudo apt-get install gddrescue   # Ubuntu/Debian: package is 'gddrescue', binary is 'ddrescue'
# or
sudo dnf install ddrescue        # Fedora: package and binary are both 'ddrescue'
```

### Connect a destination drive

You need a destination disk at least as large as the SN750. Connect a second external drive
(USB 3.0 or another M.2 enclosure).

Identify the destination drive with `lsblk` and note its device path (e.g., `/dev/sdc`).
Make sure it has enough free space or create a file on it.

### Option A – Image to a file (recommended if destination has enough space)

```bash
# Replace /dev/sdb with the SN750 device name
# Replace /mnt/backup with the mount point of your destination drive

sudo ddrescue -d -r3 --log-rates /dev/sdb /mnt/backup/davids_ssd.img /mnt/backup/davids_ssd.log
```

- `-d` – direct disc access (bypasses kernel cache)
- `-r3` – retry bad sectors up to 3 times
- `--log-rates` – shows transfer rates in the terminal

> Leave this running. It may take 30 minutes to several hours depending on drive health.
> If sectors are failing, `ddrescue` will keep going and note which areas failed in the log file.

### Option B – Clone directly to another drive (same size or larger)

```bash
sudo ddrescue -d -r3 /dev/sdb /dev/sdc /mnt/backup/davids_ssd.log
```

### Verify the image (after ddrescue completes)

```bash
sudo ddrescue --status /mnt/backup/davids_ssd.log
```

A healthy image will show `0 errors`. Some errors are still workable – files in healthy sectors
can still be recovered even if other areas failed.

---

## Step 4 – Recover Files from the Image

Now that you have an image, work from the image file – never the original drive.

### Option A – Mount the image read-only and browse files

If the filesystem is intact (drive was healthy, maybe just needed a check):

```bash
# Find the partition offset inside the image
sudo fdisk -l /mnt/backup/davids_ssd.img

# Mount the relevant partition read-only using the offset from fdisk output.
# Replace 2048 below with the actual 'Start' sector value shown by fdisk for the partition
# you want to mount (the sector size is 512 bytes, so offset = Start * 512).
sudo mount -o ro,loop,offset=$((2048*512)) /mnt/backup/davids_ssd.img /mnt/recovery

# Browse and copy files
ls /mnt/recovery
cp -r /mnt/recovery/Users/David/Music /mnt/backup/recovered_music/
```

### Option B – Use `TestDisk` to repair partition table / recover lost partitions

Install TestDisk:

```bash
sudo apt-get install testdisk
```

Run it against the image:

```bash
sudo testdisk /mnt/backup/davids_ssd.img
```

Follow the interactive menu:
1. Select the image file.
2. Choose the partition table type (usually **Intel/PC** for Windows drives).
3. Select **Analyse** → **Quick Search**.
4. If the original partition shows up, select **Write** to restore the partition table.
5. Then mount the image as shown in Option A above.

### Option C – Use `PhotoRec` for deep file carving (best for music files)

PhotoRec recovers files by their internal signatures regardless of filesystem damage. It works
even when the partition table and directory structure are completely gone.

```bash
sudo photorec /mnt/backup/davids_ssd.img
```

In the menu:
1. Select the image.
2. Choose the partition to scan (or scan the whole image).
3. Choose **File Opt** and enable audio formats: WAV, AIFF, MP3, FLAC, OGG, M4A, etc. Also
   enable project file formats for any DAW David uses (`.ptx` for Pro Tools, `.als` for Ableton,
   `.flp` for FL Studio, `.rpp` for REAPER, `.logic` for Logic Pro).
4. Choose a destination folder on the backup drive.
5. Press **Search**.

> PhotoRec will recover files **without their original filenames or folder structure**. You will get
> numbered files sorted by type. Use a DAW to open and identify them.

---

## Step 5 – Professional Recovery (Last Resort)

If the drive is not detected at all in BIOS or produces no output with `ddrescue`, the NAND flash
or controller may be physically damaged. At this point, **stop** – further attempts may cause more
damage.

Professional labs that handle NVMe SSDs:

- **DriveSavers** – [drivesaversdatarecovery.com](https://www.drivesaversdatarecovery.com) (US)
- **Ontrack** – [ontrack.com](https://www.ontrack.com) (worldwide)
- **Salvagedata** – [salvagedata.com](https://www.salvagedata.com) (US)

Costs typically range from $300–$2000+ depending on damage. Many offer free evaluations.

---

## Summary Decision Tree

```
Drive plugged in via USB enclosure
        │
        ▼
Is the drive detected in BIOS / lsblk?
  NO ──────────────────────────────────► Professional recovery lab
  │
  YES
  │
  ▼
Boot Linux Live USB
Run ddrescue to image the drive
        │
        ▼
Did ddrescue complete with 0 or few errors?
  YES ─► Mount image read-only, copy files directly
  │
  ▼
Try TestDisk to recover partition table
        │
        ▼
Still not accessible? Run PhotoRec for deep file carving
        │
        ▼
Still missing files? ──────────────────► Professional recovery lab
```

---

## Tools Reference

| Tool | Platform | Purpose | Link |
|---|---|---|---|
| `ddrescue` | Linux | Disk imaging with error handling | `apt install gddrescue` |
| `TestDisk` | Linux/Win/Mac | Partition table repair | `apt install testdisk` |
| `PhotoRec` | Linux/Win/Mac | Deep file carving | Included with TestDisk |
| Recuva | Windows | File recovery (easy UI) | [piriform.com/recuva](https://www.piriform.com/recuva) |
| R-Studio | Windows/Mac/Linux | Advanced recovery | [r-studio.com](https://www.r-studio.com) |
| Rufus | Windows | Create Linux live USB | [rufus.ie](https://rufus.ie) |
| balenaEtcher | Win/Mac/Linux | Create Linux live USB | [etcher.balena.io](https://etcher.balena.io) |

---

## Notes on the WD Black SN750

- The SN750 is an **NVMe M.2 (PCIe 3.0 x4)** drive. The M.2 USB enclosure must support NVMe
  (not just SATA M.2). Verify the enclosure packaging says "NVMe" support.
- SSDs use wear-leveling and TRIM, which can reduce recoverability over time compared to HDDs.
  Act quickly to create the disk image, but **minimize unnecessary power cycles** while planning
  your approach – every extra power-on of a degraded SSD risks further loss.
- If the drive was an **OS boot drive**, Windows will likely want to run startup repair when it
  detects the drive. Prevent this by using a Linux live environment exclusively.
- NVMe SSD encryption: The SN750 supports hardware encryption. If BitLocker was enabled on the
  Windows partition, you will need the BitLocker recovery key (stored in Microsoft account or
  printed/saved separately) to decrypt the recovered files.

---

*Good luck – we hope David gets his music back!* 🎵
