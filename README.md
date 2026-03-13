# Save-Davids-Music

David's music recording SSD is failing. Can we help him save it?

## Drive Details

- **Drive:** Western Digital Black SN750 (NVMe M.2)
- **Contents:** Irreplaceable music recordings and creative projects
- **Status:** Removed from computer; condition unknown

## Available Hardware

- Multiple computers to work from
- M.2 external USB enclosure (for non-destructive access)

## Recovery Plan

See **[DATA_RECOVERY.md](DATA_RECOVERY.md)** for the full step-by-step guide.

### TL;DR – Recommended Order of Operations

1. **Do NOT plug the drive into Windows without reading the guide first.** Windows can silently overwrite data.
2. Boot a Linux live USB (Ubuntu or Fedora).
3. Connect the SN750 via the M.2 USB enclosure.
4. Create a full disk image with `ddrescue` before attempting any recovery.
5. Run `TestDisk` / `PhotoRec` on the image to recover files.
6. If the image approach fails, try Windows recovery software (Recuva, R-Studio) as a fallback.
7. If all else fails, contact a professional recovery lab.

> **Golden Rule:** Never work directly on the original drive until you have a verified clone or image.
