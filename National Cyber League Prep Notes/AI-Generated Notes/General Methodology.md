# NCL General Methodology & Tips

---

## Universal First Steps (Any Challenge)

No matter the category, always do these before diving deep:

1. **Read the question carefully** — the wording often hints at the exact tool or technique needed.
2. **Run `file` on any unknown file** — `file filename` identifies the real type regardless of extension.
3. **Run `strings` on any binary** — quick first look for readable clues, flags, usernames, paths.
4. **Check metadata** — `exiftool filename` on any image or document.
5. **Check the file size** — `ls -lh` for human-readable size, `du -sh` for actual disk usage.
6. **Google anything unfamiliar** — acronyms in strings output, feature flag names, error messages.

---

## Filesystem Identification Cheat Sheet

Run this first on any disk image:
```bash
fsstat disk.img
```

If that fails, use strings to fingerprint:

| String found | Filesystem |
|-------------|-----------|
| `org.zfsonlinux` | ZFS |
| `com.delphix` | ZFS |
| `zpool` / `vdev` | ZFS |
| `ext4` / `ext3` | Linux ext filesystem |
| `NTFS` | Windows NTFS |
| `FAT32` / `FAT16` | FAT filesystem |
| `HFS+` | macOS HFS+ |

**ZFS on Kali:** ZFS does not work natively on Kali's bleeding-edge kernel. Use Ubuntu for ZFS challenges — `zfsutils-linux` installs cleanly and the kernel module loads without issues.

---

## Timestamp Conversion

Unix epoch timestamps appear frequently in forensics challenges.

```bash
# Convert epoch to readable date
date -d @1714974574

# Python alternative
python3 -c "import datetime; print(datetime.datetime.fromtimestamp(1714974574, datetime.UTC))"
```

When searching for timestamps in binary files, scan for 32-bit integers in a valid range:
- 2020 = `1577836800`
- 2027 = `1798761600`

---

## Quick Reference — Key Commands

```bash
# File identification
file filename
exiftool filename
exiftool -v3 filename          # hex dump + verbose metadata

# String extraction
strings filename
strings filename | grep -i "flag"
strings filename | grep -i "password"
strings disk.gz | grep -A2 -B2 "keyword"   # show context around keyword

# Disk images
fsstat disk.img                # identify filesystem
fls -r disk.img                # list files (Sleuthkit)
icat disk.img <inode>          # extract file by inode (Sleuthkit)
binwalk -e disk.img            # extract embedded files
foremost -i disk.img -o ./out  # carve files by signature

# ZFS specific
sudo zdb -l disk.img 2>/dev/null    # read pool label metadata
sudo zdb -u disk.img 2>/dev/null    # read uberblock array
sudo zpool import -f -d $(pwd) poolname
sudo zpool status poolname          # shows scrub info
sudo zpool history poolname         # shows full command history

# Hex inspection
xxd filename | head -5          # first 5 lines of hex dump
hexdump filename -C | head      # canonical hex dump
hexdump screenshot.png -C | head  # inspect a known-good file for comparison

# Network / Web
nikto -h https://target.com
curl -X POST https://target.com/login -H "Content-Type: application/json" -d '{}' -v -L
```

---

## NCL-Specific Notes

- **Flags are usually formatted as:** `NCL-XXXX-XXXX` or similar — grep for `NCL` in strings output.
- **Challenges are connected** — usernames, hostnames, and themes often carry across multiple challenges in the same category (e.g. `liber8`, `secret-server`, `cheddar_holt` appearing in both forensics and web).
- **Easy challenges have simple solutions** — if you've been trying complex techniques for more than 10 minutes on an "easy" challenge, step back and look for something obvious you missed (a button, a page, a comment in source).
- **No brute force** — NCL explicitly prohibits brute forcing. Use wordlists only where allowed and stay within the rules.
