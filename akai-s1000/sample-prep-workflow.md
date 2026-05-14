# Loading Samples onto an Akai S1000 / S1100 via ZuluSCSI

End-to-end guide for prepping a WAV → loading it as an Akai sample → building a drum-style program. Tested on macOS, but every step uses cross-platform tools (`ffmpeg`, `akaiutil`, `python3`).

---

## 0. What you need

| Tool | Purpose | Install (Mac) |
|---|---|---|
| `ffmpeg` | Audio conversion | `brew install ffmpeg` |
| `akaiutil` | Read/write Akai disk images | Build from source: <https://sourceforge.net/projects/akaiutil/> |
| `python3` | Small hex-patch helpers | Already on macOS |

Plus:
- A **ZuluSCSI** (or SCSI2SD) connected to your Akai S1100
- The SD card formatted **FAT32** (works on cards up to 32 GB). For larger cards on Mac: `diskutil eraseDisk MS-DOS "ZULU" MBR /dev/diskN`.

---

## 1. SD card layout

Inside the ZuluSCSI SD card you'll have a few `HDx.img` files plus a `zuluscsi.ini` and a `readMe.txt`:

```
/Volumes/AKAI S1000/
  HD0.img    HD1.img    HD2.img    HD3.img    HD4.img    HD5.img    HD6.img
  zuluscsi.ini   readMe.txt
```

Each `.img` is a 512 MB Akai-formatted disk and is presented to the S1100 as a separate SCSI drive. Each disk is split into nine partitions (A–I); each partition is up to 60 MB (the S1000 OS cap); each partition contains **volumes**; each volume contains files. File types:

| Extension | Meaning |
|---|---|
| `.S1` | S1000-format sample |
| `.P1` | S1000-format program |
| `.D` | Drum-input routing |

---

## 2. Prep the audio

Every WAV destined for the S1100 needs four transforms:

1. Convert to **Mono**
2. Insert **0.10 seconds of silence** at the end (kills click artefacts at sample-end)
3. Resample to **44.1 kHz**
4. Save as **16-bit PCM WAV**

One-liner per file:

```bash
ffmpeg -y -i "INPUT.wav" -af "apad=pad_dur=0.1" -ac 1 -ar 44100 -c:a pcm_s16le "OUTPUT.wav"
```

Batch (zsh):

```bash
SRC="/path/to/source/wavs"
OUT="/path/to/source/wavs/_s1100"
mkdir -p "$OUT"
cd "$SRC"
for f in *.wav; do
  ffmpeg -y -i "$f" -af "apad=pad_dur=0.1" -ac 1 -ar 44100 -c:a pcm_s16le "$OUT/$f"
done
```

---

## 3. Rename for the Akai filesystem

Akai sample names are stored in a 6-bit encoding with a **12-character limit**, restricted to:

- `A`–`Z` (uppercase only)
- `0`–`9`
- `' '` (space)

**Caveat:** `akaiutil`'s command-line parser splits arguments on whitespace, so filenames with spaces can't be created or referenced by name on the CLI. Easiest workaround: **use no-space names for the WAV files before importing**. Once they're inside the Akai image, hex-patching can add spaces back if you really want them (see step 7), but no-space names are simpler.

Pick short, descriptive names. Example mapping:

| Original | Renamed (≤ 12 chars, no spaces) |
|---|---|
| `GG_Bonita Tiger Combo 10.wav` | `TIGERC10.wav` |
| `GG_Bonita Tiger Combo Project 170.wav` | `TIGERPROJ.wav` |
| `GG_Classic 2000's Roller 1 168.wav` | `ROLLER1.wav` |

---

## 4. Inspect the SD card with `akaiutil`

`akaiutil` operates on the `.img` file directly. It has an interactive shell, but you can also pipe a list of commands in.

```bash
# Show partitions + free space on HD5.img
printf 'df\nexit\n' | akaiutil -r "/Volumes/AKAI S1000/HD5.img"

# List volumes inside partition B
printf 'cd /disk0/B\nls\nexit\n' | akaiutil -r "/Volumes/AKAI S1000/HD5.img"

# Recursive dump of everything on a disk
printf 'cd /disk0\nlsrec\nexit\n' | akaiutil -r "/Volumes/AKAI S1000/HD5.img"
```

`-r` opens read-only — always use it for inspection. Drop the flag when you actually want to write.

To enter an existing volume whose name contains a space, use the **index** form:

```bash
printf 'cd /disk0/B\ncdi 5\nls\nexit\n' | akaiutil -r "/Volumes/AKAI S1000/HD5.img"
```

`cdi 5` = "change directory to volume index 5", bypassing the space-in-name issue.

---

## 5. Create a new volume

`mkvol1` creates an S1000-format volume. With a no-space name it's one line:

```bash
SD="/Volumes/AKAI S1000/HD5.img"
printf 'cd /disk0/B\nmkvol1 NEWVOLUME\nexit\n' | akaiutil "$SD"
```

### 5a. Creating a volume name *with* a space

`akaiutil` won't accept spaces in `mkvol1`'s name argument. To match the factory naming style (e.g. `VOLUME 003`), create it with a placeholder name and then hex-patch the 12 bytes that hold the name.

Akai 6-bit character encoding (used for **all** names — volumes, samples, programs):

| Char | Hex | | Char | Hex |
|---|---|---|---|---|
| `0`–`9` | `0x00`–`0x09` | | `A` | `0x0b` |
| `' '` (space) | `0x0a` | | `B` | `0x0c` |
| | | | ... | ... |
| | | | `Z` | `0x24` |

Names are padded to 12 bytes with `0x0a` (space). Example: `VOLUME 003` encodes to `20 19 16 1f 17 0f 0a 00 00 03 0a 0a`.

Python helper:

```python
def akai_encode(s, length=12):
    out = bytearray()
    for c in s.upper():
        if '0' <= c <= '9':    out.append(ord(c) - ord('0'))
        elif c == ' ':          out.append(0x0a)
        elif 'A' <= c <= 'Z':   out.append(0x0b + ord(c) - ord('A'))
    while len(out) < length:    out.append(0x0a)
    return bytes(out[:length])
```

Full create-and-patch:

```bash
SD="/Volumes/AKAI S1000/HD5.img"

# 1. Create with placeholder name
printf 'cd /disk0/B\nmkvol1 PATCHME\nexit\n' | akaiutil "$SD"

# 2. Hex-patch placeholder bytes to "VOLUME 003"
python3 <<'PYEOF'
def encode(s):
    out = bytearray()
    for c in s.upper():
        if '0' <= c <= '9':    out.append(ord(c) - ord('0'))
        elif c == ' ':          out.append(0x0a)
        elif 'A' <= c <= 'Z':   out.append(0x0b + ord(c) - ord('A'))
    while len(out) < 12: out.append(0x0a)
    return bytes(out[:12])

placeholder = encode("PATCHME")
target      = encode("VOLUME 003")
import os
path = "/Volumes/AKAI S1000/HD5.img"
with open(path, 'r+b') as f:
    data = f.read()
    idx = data.find(placeholder)
    assert idx != -1, "placeholder not found"
    f.seek(idx); f.write(target); f.flush(); os.fsync(f.fileno())
    print(f"patched at offset 0x{idx:x}")
PYEOF

# 3. Verify
printf 'cd /disk0/B\nls\nexit\n' | akaiutil -r "$SD"
```

---

## 6. Import WAVs as `.S1` samples

`wav2sample1` converts WAV → S1000 sample and stores it in the current volume. The "1" suffix means S1000 format (`wav2sample3` = S3000, `wav2sample9` = S900).

```bash
SD="/Volumes/AKAI S1000/HD5.img"
OUT="/path/to/processed/wavs"   # No-space filenames

cat > /tmp/akai_cmds.txt <<EOF
lcd $OUT
cd /disk0/B
cdi 5
wav2sample1 TIGERC10.wav
wav2sample1 TIGERPROJ.wav
wav2sample1 ROLLER1.wav
ls
exit
EOF
akaiutil "$SD" < /tmp/akai_cmds.txt
```

`lcd` sets the local (external) directory. `wav2sample1` reads from there and writes the resulting `.S1` into the current Akai volume.

---

## 7. Build a `.P1` drum program

A `.P1` program file maps MIDI keys to samples. The format (reverse-engineered from the user's existing programs):

### Header (150 bytes)

| Offset | Bytes | Field |
|---|---|---|
| `0x00` | 1 | Program ID = `0x01` |
| `0x01` | 1 | Unclear (likely checksum-like; leave from template) |
| `0x02` | 1 | Padding |
| `0x03`–`0x0e` | 12 | Program name (6-bit encoded) |
| `0x2a` | 1 | Number of keygroups |

### Keygroups (150 bytes each), starting at offset 150

| Offset (within keygroup) | Field |
|---|---|
| `0x03` | lokey (MIDI note) |
| `0x04` | hikey (MIDI note) |
| `0x22`–`0x2d` | sample name reference (12 bytes, 6-bit encoded) |

Total file size: `150 + N * 150` bytes, where N is the keygroup count.

### Building from a template

The simplest path is to take an existing drum-style program (e.g. `BRK1.P1`) and modify it. First extract a template:

```bash
mkdir -p /tmp/akai_p1
cd /tmp/akai_p1
# Replace indices with whatever fits your card
printf 'cd /disk0/A\ncdi 2\ngeti 11\nexit\n' | akaiutil -r "/Volumes/AKAI S1000/HD5.img"
# This pulls file index 11 from volume 2 of partition A
```

Then build your custom program:

```python
def akai_encode(s, length=12):
    out = bytearray()
    for c in s.upper():
        if '0' <= c <= '9':    out.append(ord(c) - ord('0'))
        elif c == ' ':          out.append(0x0a)
        elif 'A' <= c <= 'Z':   out.append(0x0b + ord(c) - ord('A'))
    while len(out) < length: out.append(0x0a)
    return bytes(out[:length])

# Load template (10-keygroup drum program)
with open('BRK1.P1', 'rb') as f:
    data = bytearray(f.read())

# Header edits
data[0x03:0x0f] = akai_encode("DRUMS")   # program name
data[0x2a] = 3                            # number of keygroups

# Build keygroups by templating the first keygroup of BRK1
kg_template = bytes(data[150:300])
mapping = [
    # (MIDI key, sample name — must match an .S1 you imported)
    (36, "TIGERC10"),
    (37, "TIGERPROJ"),
    (38, "ROLLER1"),
]
new_kgs = bytearray()
for key, sname in mapping:
    kg = bytearray(kg_template)
    kg[0x03] = key                              # lokey
    kg[0x04] = key                              # hikey
    kg[0x22:0x2e] = akai_encode(sname)          # sample reference
    new_kgs += kg

with open('DRUMS.P1', 'wb') as f:
    f.write(data[:150] + bytes(new_kgs))

print(f"wrote DRUMS.P1: {150 + 3 * 150} bytes")
```

Import the resulting `.P1` into the target volume:

```bash
cat > /tmp/akai_cmds.txt <<EOF
lcd /tmp/akai_p1
cd /disk0/B
cdi 5
put DRUMS.P1
ls
exit
EOF
akaiutil "$SD" < /tmp/akai_cmds.txt
```

### Limitations of hand-built `.P1`

- Non-key, non-name parameters (volume, tune, envelope, filter, layer settings) inherit from the template's first keygroup. Usually fine, but you may want to fine-tune on the S1100 itself.
- The byte at offset `0x01` of every keygroup follows a pattern across keygroups (`previous + 150 mod 256`) — likely a chained pointer or checksum. Leaving it as-is from the template usually works. If the S1100 rejects the program, the fastest fix is to re-save the program on the S1100 hardware; the unit will regenerate the offending bytes correctly.

---

## 8. Eject safely

After any write, flush + eject so macOS commits the changes back to the SD card before you pull it.

```bash
diskutil eject "AKAI S1000"
```

Or eject from Finder. **Don't just unplug it** — the writes may not have hit the card yet.

---

## Reference: useful `akaiutil` commands

| Command | Purpose |
|---|---|
| `df` | List disks + free space |
| `cd <path>` | Change directory (no spaces allowed in path) |
| `cdi <index>` | Change directory by volume/file index |
| `ls` | List current directory |
| `lsrec` | Recursive list |
| `geti <index>` | Export Akai file → local disk |
| `put <file>` | Import local file → Akai volume |
| `wav2sample1 <wav>` | Convert WAV → S1000 sample (in current volume) |
| `mkvol1 <name>` | Create new S1000 volume |
| `delvoli <index>` | Delete volume by index |
| `lcd <dir>` | Change local (external) directory |
| `help <cmd>` | Show usage for a command |
| `exit` | Quit |

---

*Generated 2026-05-14*
