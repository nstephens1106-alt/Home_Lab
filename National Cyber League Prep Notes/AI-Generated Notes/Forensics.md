# NCL Forensics Reference

---

## Tool: CyberChef

### What it is
CyberChef is a browser-based data analysis tool. It works by chaining "operations" (called a Recipe) to transform or analyze data step by step.

### Use Case: Identifying corrupted or misidentified files

**Scenario:** You're given a file (e.g. `flag.jpeg`) that won't open or behaves strangely.

**Recipe steps:**
1. `Magic` — detects the actual file type by analyzing the data. Enable "Intensive mode" and "Extensive language support" for deeper analysis.
2. `To Hex` — converts the file to hexadecimal so you can manually inspect the raw bytes.
3. `Strings` — extracts readable text from the binary, useful for spotting acronyms, keywords, or embedded data.

**What to look for in the hex output:**
- The first 8 bytes are the **magic bytes** (file signature). Compare them to known signatures.
- If the magic bytes don't match the file extension, the file has been corrupted or disguised.

**Common magic bytes:**
| File Type | Magic Bytes (Hex) |
|-----------|-------------------|
| PNG | `89 50 4E 47 0D 0A 1A 0A` |
| JPEG | `FF D8 FF` |
| ZIP | `50 4B 03 04` |
| PDF | `25 50 44 46` |
| GZ | `1F 8B` |

### Use Case: Repairing a file signature (magic byte fix)

**Scenario:** A file is disguised as a JPEG but is actually a PNG. The file won't open because the header is wrong.

**Step-by-step process:**
1. Open the file in CyberChef → use `To Hex` to see the raw bytes.
2. Use `Magic` to detect the real file type — look for acronyms or known signatures in the strings output.
3. Replace the first 8 bytes with the correct PNG magic bytes: `89 50 4E 47 0D 0A 1A 0A`
4. If the file still won't open, the chunk length may also be corrupted.
5. Take a screenshot of a known valid PNG, then run: `hexdump screenshot.png -C | head` to get the correct IHDR chunk length.
6. Replace the corrupted length in the original file with the valid one.
7. Open the repaired file to retrieve the flag.

**Key concept:** PNG files use strict chunk formatting. Each chunk has a length field, a type, data, and a CRC checksum. If any of these are corrupted the file won't render — even if the magic bytes are correct.

### Use Case: SVG redaction bypass

**Scenario:** A web page shows a password visually "redacted" with a black box, rendered as an SVG.

**What's actually happening:**  
The developer drew a black rectangle on top of the password text inside the SVG. The password is still present in the file as vector path data — it's just visually covered.

**How to reveal it:**
1. View the raw page source (right click → View Page Source or intercept with Burp Suite).
2. Find the SVG block — look for `<path d="M6 8h388v42H6z">` which is the black rectangle.
3. Copy the full SVG into [svgviewer.dev](https://svgviewer.dev).
4. Remove the rectangle path (`M6 8h388v42H6z`) from the SVG.
5. The password text underneath will now be visible.

**Key lesson:** Visual redaction ≠ actual redaction. This same concept applies to:
- PDFs with black boxes drawn over text (the text is still there, just covered)
- CSS `display:none` elements (still in the DOM, just hidden)
- HTML comments containing sensitive data
- Images with text "covered" by a layer in Photoshop exports

---

## Tool: exiftool

### What it is
`exiftool` is a command-line tool for reading and writing metadata in files. Every image, document, and many binary files store hidden metadata about how they were created.

### Basic usage
```bash
exiftool filename        # read all metadata
exiftool -v3 filename    # verbose mode with hex dump of raw bytes
```

### Use Case: Reading file metadata
```bash
exiftool flag.jpeg
```
This returns information like GPS coordinates, camera model, creation date, software used, and sometimes hidden comments — all of which can be flags or clues in NCL challenges.

### Use Case: Detecting hidden/appended data with -v3
```bash
exiftool -v3 filename
```
The `-v3` flag outputs a hex dump of the file's raw bytes. This is useful for spotting **trailing data** — extra bytes appended after the legitimate end of the file.

**What to look for:**
- Every file type has a defined ending (e.g. JPEG ends with `FF D9`).
- If there are additional bytes after that ending marker, a second file may be hidden there.
- The extra bytes will often begin with a recognizable magic byte signature.

**Important terminology correction:**
- "Trailing headers" is a misnomer — headers appear at the *beginning* of a file.
- What you're looking for is **trailing bytes** or **appended data** after the file's end marker.

---

## Tool: binwalk

### What it is
`binwalk` scans a file for embedded files and executable code. It's particularly useful for forensics challenges involving disk images, firmware, or stacked files.

### Basic usage
```bash
binwalk filename              # scan for embedded files
binwalk -e filename           # extract embedded files automatically
binwalk -e disk.img           # extract from a disk image
```

### Use Case: Extracting zlib-compressed data from a disk image
If binwalk reports zlib data but can't extract automatically (missing utilities warning), extract manually using the reported hex offsets:

```bash
# Using dd and zlib-flate
dd if=disk.img bs=1 skip=4260926 | zlib-flate -uncompress > extract1.bin

# Using Python if zlib-flate is unavailable
python3 -c "
import zlib, sys
with open('disk.img', 'rb') as f:
    f.seek(4260926)
    data = f.read()
    print(zlib.decompress(data)[:500])
"
```

### Install missing dependencies if extraction fails
```bash
sudo apt install mtd-utils gzip bzip2 tar arj lhasa p7zip p7zip-full cabextract squashfs-tools sleuthkit
```

---

## Tool: strings

### What it is
`strings` extracts all readable ASCII text from a binary file. Extremely fast first-look tool for any unknown file.

### Basic usage
```bash
strings filename                    # extract readable strings
strings filename | grep -i "flag"   # search for flags
strings filename | grep -i "2024"   # search for years (useful for timestamps)
strings disk.gz | grep -A2 -B2 "scrub"  # show context around a keyword
```

### Use Case: Identifying filesystem type from strings output
ZFS filesystem feature flags appear as readable strings and can fingerprint the filesystem without mounting it:
- `org.zfsonlinux` — ZFS on Linux project identifier
- `com.delphix` — Delphix contributes exclusively to ZFS
- `zpool_checkpoint` — "zpool" is ZFS-specific terminology
- `vdev` — virtual device, a ZFS concept

---

## Tool: zdb

### What it is
`zdb` is a ZFS debugger that can read pool metadata directly from a raw disk image without needing the ZFS kernel module loaded.

### Basic usage
```bash
sudo zdb -l disk.img 2>/dev/null        # read ZFS labels (pool metadata)
sudo zdb -u disk.img 2>/dev/null        # dump uberblock array
```

### Use Case: Reading ZFS pool metadata
```bash
sudo zdb -l disk.img 2>/dev/null
```

Returns pool information including:
- `name` — pool name (e.g. `diskpool123`)
- `pool_guid` — unique pool identifier
- `hostname` — machine that created the pool
- `hostid` — host identifier
- `version` — ZFS pool version
- `vdev_tree` — virtual device info including original disk path and allocated size (`asize`)
- `labels` — number of ZFS labels on the disk

### Key ZFS concepts
- **txg (transaction group):** A counter for how many write transactions have occurred on the pool. Not a timestamp.
- **asize:** Allocated size of the virtual device — this is the actual disk space used, different from the raw file size.
- **Feature flags with `+`:** A `+` after a feature name means it's enabled and active on the pool.
- **Labels 0-3:** ZFS stores 4 copies of pool metadata (labels) for redundancy.

---

## Tool: Python (binary parsing)

### Use Case: Finding timestamps in a raw disk image
When readable string searches fail, timestamps are often stored as 32-bit or 64-bit binary integers. Use Python to scan the raw bytes:

```bash
python3 -c "
import struct, datetime

with open('disk', 'rb') as f:
    data = f.read()
    
    for i in range(0, len(data)-4, 4):
        for fmt in ['<I', '>I']:  # little-endian and big-endian 32-bit
            val = struct.unpack(fmt, data[i:i+4])[0]
            if 1600000000 < val < 1800000000:  # valid Unix timestamp range 2020-2027
                dt = datetime.datetime.fromtimestamp(val, datetime.UTC)
                print(f'Offset {hex(i)} ({fmt}): {val} = {dt}')
" | sort -u | head -50
```

**How to interpret results:**
- Filter out future dates (2026+) — these are likely garbage/noise data
- Cross-reference with known system dates (e.g. kernel build date from strings output sets the earliest possible date)
- ZFS scrub timestamps live in the uberblock region around offset `0x140xxxx` in label 1

---

## Lessons Learned / What Went Wrong

### Filesystem module compatibility
**Problem:** ZFS kernel module wouldn't load on Kali (`modprobe: FATAL: Module zfs not found in directory /lib/modules/6.18.12+kali-amd64`)

**Why it happened:** Kali Linux uses a bleeding-edge kernel (6.18). The ZFS DKMS build failed silently because DKMS hadn't compiled the module for that specific kernel version yet.

**What to remember:** ZFS works natively on Ubuntu. If a challenge involves a ZFS disk image, having an Ubuntu VM available saves a lot of time. `zpool status`, `zpool history`, and `zfs list` all work out of the box on Ubuntu and would have immediately returned the scrub timestamp and file listing.

### Sleuthkit can't read ZFS
**Problem:** `fsstat disk.img` returned "cannot determine file system type"

**Why it happened:** Sleuthkit (TSK) doesn't support ZFS natively — it handles ext4, NTFS, FAT, HFS+, but not ZFS.

**What to remember:** Always run `fsstat` first to identify the filesystem before trying extraction tools. If it fails, the filesystem may be ZFS, BTRFS, or another format that requires its native tools.

### Trailing bytes vs trailing headers
**Problem:** Referred to extra appended data as "trailing headers"

**Correction:** Headers appear at the *beginning* of a file and identify the file type. Extra data after a file's end marker is called **trailing bytes** or **appended data**. The concept is correct — the terminology matters in write-ups and on exams.

### strings output is noisy
**Problem:** Searching for scrub timestamps using `strings` returned hundreds of repetitive lines with no clean timestamp.

**Why it happened:** ZFS stores most of its metadata as binary-encoded structures, not readable text. The history log entries repeat because ZFS keeps multiple redundant label copies across the disk.

**What to remember:** `strings` is a quick first look, not a deep analysis tool. For structured binary data like ZFS metadata, use purpose-built tools (`zdb`) or write a parser (Python struct).

# 🧠 NCL Practice Notes --- Git & Repository Investigation

*National Cyber League (NCL) Preparation Reference*

------------------------------------------------------------------------

# 1️⃣ Git Overview (Core Concepts)

## What Git Is

**Short Description:**\
Git is a distributed version control system used to track file changes
over time.

**In‑Depth Explanation:**\
Unlike traditional version control systems that store differences
between files, Git stores *snapshots* of entire project states. Every
cloned repository contains the full history, which makes Git extremely
useful for forensic analysis. Even if files are deleted later, earlier
snapshots still exist in the repository database.

### Why This Matters in NCL

Git challenges frequently involve: - Recovering deleted files -
Investigating commit history - Extracting hidden flags - Analyzing
developer mistakes - Exploring exposed `.git` directories

------------------------------------------------------------------------

# 2️⃣ Git Repository Structure --- `.git/` Directory

The `.git` folder is the internal database of the repository.

    .git/
    ├── HEAD
    ├── config
    ├── description
    ├── hooks/
    ├── info/
    ├── logs/
    ├── objects/
    ├── refs/
    └── index

------------------------------------------------------------------------

## HEAD

**Short Description:**\
Points to the current branch or commit.

**Explanation:**\
HEAD acts like a pointer telling Git which snapshot you are currently
viewing. Normally it references a branch:

    ref: refs/heads/main

If HEAD contains a commit hash instead, the repo is in a **detached
HEAD** state --- often useful during investigations.

------------------------------------------------------------------------

## config

**Short Description:**\
Stores repository configuration.

**Explanation:**\
Contains: - usernames - emails - remote repository URLs - repository
behavior settings

Useful command:

``` bash
git config --list
```

NCL Tip: Developer emails or usernames sometimes contain clues or flags.

------------------------------------------------------------------------

## index (Staging Area)

**Short Description:**\
Tracks files staged for the next commit.

**Explanation:**\
The index is a binary file representing what will be included in the
next snapshot. It sits between the working directory and commits.

View tracked files:

``` bash
git ls-files
```

------------------------------------------------------------------------

## objects/ (MOST IMPORTANT)

**Short Description:**\
Stores every piece of Git data.

**Explanation:**\
Git stores data as compressed objects identified by SHA‑1 hashes.

Object types: - **blob** → file contents - **tree** → directory
structure - **commit** → metadata + pointers - **tag** → named reference

Structure example:

    .git/objects/aa/bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb

Inspect objects:

``` bash
git cat-file -t <hash>
git cat-file -p <hash>
```

NCL Insight: Flags are often hidden directly inside blob objects.

------------------------------------------------------------------------

## refs/

**Short Description:**\
Stores branch and tag pointers.

**Explanation:**\
Branches are simply files containing commit hashes.

    refs/
    ├── heads/
    ├── tags/
    └── remotes/

------------------------------------------------------------------------

## logs/

**Short Description:**\
Tracks reference history.

**Explanation:**\
Records when branches moved between commits. Even rewritten history can
often be recovered here.

Command:

``` bash
git reflog
```

------------------------------------------------------------------------

## hooks/

**Short Description:**\
Automation scripts triggered by Git actions.

**Explanation:**\
Hooks can run scripts before or after commits/pushes. While usually
disabled, challenges may hide clues here.

------------------------------------------------------------------------

# 3️⃣ Essential Git Commands

------------------------------------------------------------------------

## Repository Discovery

``` bash
ls -a
```

Look for:

    .git

Explanation: Hidden repositories may indicate exposed developer
environments.

------------------------------------------------------------------------

## Clone Repository

``` bash
git clone <repo_url>
```

Used when `.git` directories are publicly accessible on web servers.

------------------------------------------------------------------------

## Repository Status

``` bash
git status
```

Shows: - modified files - staged changes - current branch

Helps understand current repo state before investigation.

------------------------------------------------------------------------

## Commit History

``` bash
git log
```

Better visualization:

``` bash
git log --oneline --all --graph
```

Explanation: Displays timeline of developer activity.

------------------------------------------------------------------------

## Inspect Specific Commit

``` bash
git show <commit_hash>
```

Shows: - file changes - commit message - author info

Common flag location.

------------------------------------------------------------------------

## File History

``` bash
git log <filename>
```

View differences:

``` bash
git diff <commit1> <commit2>
```

------------------------------------------------------------------------

## Recover Deleted Files

``` bash
git checkout <commit_hash> -- file.txt
```

Explanation: Git rarely deletes data permanently.

------------------------------------------------------------------------

## Branch Investigation

``` bash
git branch -a
```

Switch branches:

``` bash
git checkout <branch>
```

Hidden branches often contain flags.

------------------------------------------------------------------------

## Searching Repository

``` bash
git grep "flag"
```

Search commit messages:

``` bash
git log --grep="flag"
```

------------------------------------------------------------------------

# 4️⃣ Git Object Investigation (High‑Value Topic)

## List All Objects

``` bash
git rev-list --all
```

Explanation: Enumerates every commit reachable in history.

------------------------------------------------------------------------

## Identify Object Type

``` bash
git cat-file -t <hash>
```

------------------------------------------------------------------------

## View Object Contents

``` bash
git cat-file -p <hash>
```

Often reveals hidden files or messages.

------------------------------------------------------------------------

## Rebuild Downloaded Repository

``` bash
wget -r http://target/.git/
git checkout .
```

Used when `.git` exposure occurs online.

------------------------------------------------------------------------

# 5️⃣ Investigation Techniques

## View Complete History

``` bash
git log --all --full-history
```

------------------------------------------------------------------------

## Find All Stored Files

``` bash
git rev-list --objects --all
```

Helps locate forgotten files.

------------------------------------------------------------------------

## Recover Lost Commits

``` bash
git reflog
git checkout <hash>
```

Explanation: Reflog tracks HEAD movement even after resets.

------------------------------------------------------------------------

## Remote Information

``` bash
git remote -v
```

Shows origin servers and URLs.

------------------------------------------------------------------------

# 6️⃣ Key Git Concepts

## Commit Hash

Unique SHA‑1 identifier for objects.

Example:

    a3f5c92b1d...

Used for referencing snapshots precisely.

------------------------------------------------------------------------

## Git Workflow

    Working Directory → Staging Area → Commit History

Commands:

``` bash
git add
git commit
```

------------------------------------------------------------------------

## Detached HEAD

Occurs when checking out a specific commit instead of a branch.

Useful for historical analysis.

------------------------------------------------------------------------

## Hidden Data Locations

Always check: - old commits - unused branches - stash entries - reflog -
tags

------------------------------------------------------------------------

# 7️⃣ Additional Commands (Common in NCL)

## Stash

``` bash
git stash list
git stash show -p
```

Developers sometimes hide unfinished work here.

------------------------------------------------------------------------

## Tags

``` bash
git tag
git show <tag>
```

Tags may reference hidden commits.

------------------------------------------------------------------------

## Blame

``` bash
git blame file.txt
```

Shows line‑by‑line authorship.

------------------------------------------------------------------------

## Reset

``` bash
git reset --hard <commit>
```

Rewrites working state.

------------------------------------------------------------------------

## Clean

``` bash
git clean -fd
```

Removes untracked files.

------------------------------------------------------------------------

# 8️⃣ Recommended NCL Investigation Workflow

1.  Identify repository
2.  View history
3.  Inspect commits
4.  Search for keywords
5.  Enumerate objects
6.  Inspect blobs
7.  Check branches/tags/stash/reflog

------------------------------------------------------------------------

# 9️⃣ High‑Value NCL Tips

✅ Git rarely deletes data permanently\
✅ Developers often leak secrets in history\
✅ `.git/config` may expose identities\
✅ Metadata can contain flags\
✅ Enumerate before analyzing deeply

------------------------------------------------------------------------

# 🔟 Quick Command Cheat Sheet

``` bash
git log --oneline --all
git show <hash>
git branch -a
git reflog
git grep flag
git rev-list --all
git cat-file -p <hash>
git stash list
git tag
git blame file
```

------------------------------------------------------------------------

# 🏁 Investigator Mindset

Think like a digital forensic analyst: - History is evidence - Snapshots
preserve mistakes - Metadata matters - Deleted ≠ gone

------------------------------------------------------------------------

**End of NCL Git Notes**