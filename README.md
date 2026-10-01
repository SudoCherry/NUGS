# NUGS
The Only Portable Syncing Software You Will Ever Need.

**0.0.1** — mirror a local folder onto a portable device.

`nugs` is a single small C program. It finds attached devices, works out what
is missing or stale, and copies files across — one direction, and never
anything you did not ask for. Deletions on the device are opt-in.

It is deliberately not tied to audio. With the default filter it syncs music to
an MP3 player; the other filters let it mirror a folder of documents, contacts
or notes onto a handheld instead.

Two kinds of device are supported, and `nugs` picks whichever it finds:

- **MSC** — devices that appear as a normal filesystem (USB mass storage).
- **MTP** — devices that speak Media Transfer Protocol and have no filesystem.

## Contents

- [Building](#building)
- [Getting started](#getting-started)
- [Commands](#commands)
- [How decisions are made](#how-decisions-are-made)
- [Deletions](#deletions)
- [Which files get synced](#which-files-get-synced) — the `filter` setting, and
  the `palm` and `ce` modes for Palm OS and Windows CE / Pocket PC
- [PDAs and other non-audio devices](#pdas-and-other-non-audio-devices)
- [Notes on the two backends](#notes-on-the-two-backends)
- [Testing](#testing)
- [Configuration reference](#configuration-reference)
- [Changelog](#changelog)

## Building

```sh
make
```

That is it on Linux with a compiler. There are no runtime dependencies beyond
libc; `make` needs nothing installed unless you want MTP support, which
additionally needs `libmtp` and `libusb`.

On Debian/Ubuntu, for MTP support:

```sh
sudo apt install build-essential libmtp-dev libusb-dev
make
```

On FreeBSD:

```sh
pkg install libmtp libusb
make
```

`make` detects `libmtp` with `pkg-config` and enables the MTP backend only when
it is really there, so a plain `make` works on machines without it. To force
either way, use `make MTP=1` or `make MTP=0`.

Other targets:

```sh
make test      # run the regression suite
make install   # install to /usr/local (override with PREFIX=)
make clean
```

To run it without installing, add the build directory to your `PATH`:

```sh
ln -sf ~/Documents/NUGS/target/nugs ~/.local/bin/nugs
```

## Getting started

```sh
nugs devices          # what is plugged in?
nugs init             # write ~/.config/nugs/nugsrc
$EDITOR ~/.config/nugs/nugsrc
nugs folders          # what is the device folder called?
nugs status           # what would change?
nugs plan             # the same, itemised
nugs sync             # do it
```

`nugs init` produces something like this, with each line explained in comments:

```
library = ~/Music
subdir = Music
verify = 0
filter = audio
```

`library` is the local root on this computer. Its directory layout is mirrored
onto the device, so `~/Music/Radiohead/Weird Fishes.mp3` becomes
`Music/Radiohead/Weird Fishes.mp3` on the device.

`subdir` is the device folder the local root maps onto. Use `Muziek`, `MP3`,
`Card`, `My Documents`, or whatever your device calls it. If you are not sure,
run `nugs folders` to see the names it actually has, or set it to `""` to use
the whole device.

By default only audio files are considered — `mp3`, `m4a`, `m4b`, `aac`, `alac`,
`flac`, `ogg`, `oga`, `opus`, `wma`, `wav`, `wavpack`, `aif`, `aiff`, `ape`,
`wv`, `mpc`, `mka`, `mp2`, `mpga`, `webm`, `amr`, `mid`, `midi`, `3gp`, `m4r`,
`dsf`, and the playlist formats `m3u`, `m3u8` and `pls`. Playlists are included
so that one on the device keeps pointing at the tracks next to it.

`mp4` and `m4v` are deliberately *not* treated as audio, since they are usually
video; use `filter = custom` if you want them.

See [Which files get synced](#which-files-get-synced) to sync other kinds of
file.

## Commands

| Command | What it does |
| --- | --- |
| `devices` | list attached devices and free space |
| `folders` | list the top-level folders on the device |
| `status` | summarise the difference; never changes anything |
| `plan` | list every copy and deletion `sync` would make |
| `sync` | copy new and changed files to the device |
| `pull <device-dir> <local-dir>` | copy files back off the device (see below) |
| `init` | write a starter config file |

Options:

| Option | Effect |
| --- | --- |
| `-d, --device <name>` | pick a device by substring of its label |
| `-l, --library <dir>` | override the local root |
| `-s, --subdir <dir>` | override the device folder |
| `--config <file>` | use a different config file |
| `--prune` | also delete device files that are not in the local root |
| `--no-delete` | never even propose deletions |
| `--verify` | compare file contents, not just size and timestamp |
| `-f, --filter <mode>` | override the filter: `audio`, `all`, `custom`, `palm` or `ce` |
| `-n, --dry-run` | report what `sync` would do, write nothing |
| `-v, --verbose` | explain what is happening |
| `-h, --help` | print the built-in help |
| `--version` | print the version |

The command may come before or after the options: `nugs plan --dry-run` and
`nugs --dry-run plan` are the same.

### pull

`pull` copies files off the device and back into a real directory on this
computer. Two things about it are easy to get wrong:

**The path is relative to the device, not to `subdir`.** It does not start from
the folder named in the config. If your content is in `Music` or `Card`, you
have to say so:

```sh
nugs sync                                   # put files into Music/
nugs pull Music ~/Recovered                 # ...then take them back out
```

```
Device: /media/usb/CLIP
[1/1] 100%  song.mp3
  + song.mp3

1 pulled into /home/you/Recovered
```

**It obeys the filter.** With the default audio filter, `pull` will not fetch a
`.vcf` or a `.doc`, and will name the filter in its refusal:

```
$ nugs pull Card ~/Contacts
Device: /media/usb/PALM
No matching files under 'Card' (filter = audio).
```

Pass `--filter all`, or set `filter = custom` with your extensions, to take
everything. This matters most when pulling off a handheld, where the files you
want are rarely the audio ones.

## How decisions are made

A file is copied when the device does not have it, or when the two copies
disagree. By default they agree if the size matches and the timestamps are
within two seconds. That two-second window matters because FAT32 stores
timestamps at two-second resolution, and a stricter comparison would re-copy
every file on every run.

`--verify` compares actual file contents instead, and catches anything the
timestamp shortcut misses: a truncated copy, a file edited in place without
its timestamp changing, a device that mangles metadata. It reads every file
on both sides, so on MTP it means reading the device back — much slower than a
normal sync. Worth using when you suspect something is wrong, not every run.

The trade-off to know about: because of that two-second window, a change made
to a file within two seconds of its previous timestamp, with no size change,
can be missed by a plain `sync`. Run `sync --verify` when that matters.

## Deletions

Nothing is ever deleted unless you ask. `status` and `plan` will *tell* you
about files on the device that are not in the local root, but `sync` leaves them
alone:

```
$ nugs plan
Device: /media/usb/CLIP (MSC)
delete  4.2 MiB  Old Band/2004/live.mp3

Plan: 0 to copy (0 B), 12 already in sync, 1 to remove
```

Add `--prune` to act on it. `--no-delete` suppresses the suggestion entirely.
There is no trash on these devices, so check `plan` before you prune.

Empty directories left behind by pruning are not removed.

What `--prune` is allowed to propose depends on the
[filter](#which-files-get-synced), since the filter applies to both sides of
the comparison. Under `filter = palm`, a `.xyz` on the device is invisible, so
it is never a deletion candidate.

## Which files get synced

`filter` decides what takes part. It applies to both sides of the comparison,
so a file `nugs` will not copy is also one it will not propose to delete.

| `filter` | Takes | Aliases |
| --- | --- | --- |
| `audio` | known audio extensions — the default | `music` |
| `all` | every file | `any` |
| `custom` | the `include` list, minus `exclude` | — |
| `palm` | Palm OS types: `.prc` `.pdb` `.pqa` `.prg` `.psys`, plus audio | `palmos`, `pda` |
| `ce` | Windows CE / Pocket PC types: `.cab` `.exe` `.dll` `.reg` `.cpl` `.ocx` `.lnk`, plus audio | `wince`, `pocket`, `windowsce` |

`include` and `exclude` are comma-separated extensions. Dots are optional and
case is ignored, so `.MP3`, `mp3` and `MP3` are the same thing. `exclude` wins
in every mode, `all` included.

```ini
filter  = custom
include = mp3, m4a, vcf, pdb, doc, txt
exclude = tmp, bak
```

With `filter = custom` and no `include`, every file with an extension is
synced. `nugs` says so when it starts, so this is hard to do by accident.

`max_size` skips anything larger, with a `K`/`M`/`G`/`T` suffix:

```ini
max_size = 64M
```

Hidden files, and the clutter devices and desktops leave lying around
(`.DS_Store`, `Thumbs.db`, `desktop.ini`, `lost+found`), are never synced in
any mode.

Override the mode for one run without editing the config:

```sh
nugs sync --filter all
```

`--filter custom` uses the `include` and `exclude` from the config file.

### palm and ce

`palm` and `ce` exist because both platforms have file types of their own, and
spelling out a long `include` list for every handheld folder is tiresome.

`filter = palm` takes:

| | |
| --- | --- |
| `.prc` | application or resource bundle |
| `.prg` | the older name for `.prc` |
| `.pdb` | database: Address, Memo, ToDo |
| `.pqa` | Palm Query Application, the pre-`.prc` format |
| `.psys` | system resource / launcher skin |
| plus | `.vcf` `.vcs` `.vnote` `.txt` `.doc` `.rtf` `.htm` `.html` `.pdf`, `.jpg` `.jpeg` `.png` `.gif` `.bmp` `.tif` `.tiff`, `.zip` `.gz` `.ttf` `.ttc` `.snd`, and audio |

`filter = ce` takes the Windows CE and Pocket PC set:

| | |
| --- | --- |
| `.exe` `.dll` `.ocx` `.cpl` `.reg` `.cab` | programs and the plumbing that installs them |
| `.lnk` `.ini` `.xml` `.e32` `.stl` `.cel` `.cpg` | shortcuts, settings, themes, precompiled HTML |
| `.prc` `.pdb` | shared with Palm: both platforms used these |
| `.sql` `.sdf` `.mdb` | databases |
| `.vcf` `.vcs` `.txt` `.doc` `.rtf` `.htm` `.html` `.pdf` `.xls` `.ppt` `.csv` | documents |
| `.jpg` `.jpeg` `.png` `.gif` `.bmp` `.tif` `.tiff` `.ttf` `.ttc`, and audio | media |

Both modes include audio, because a handheld folder usually holds media
alongside its apps and databases rather than instead of them.

`palm`, `palmos` and `pda` all mean `palm`. `ce`, `wince`, `pocket` and
`windowsce` all mean `ce`.

Neither list can be exhaustive — Palm OS and Windows CE both accepted almost any
extension. These cover the well-known types; for anything missed, `custom` is
the answer:

```ini
filter  = custom
include = prc, pdb, pqa, snd
```

### Filters and deletion

Because the filter applies to both sides, `--prune` can only ever propose
deleting a file the filter accepts. A `.xyz` sitting on a device is invisible
under `filter = palm`, so it is neither counted nor deleted:

```
$ nugs plan --prune           # filter = palm
Device: /media/usb/PALM (MSC)
delete  64 B     junk.bmp           # a palm type, not in the local root
copy    64 B     new.prc

Plan: 1 to copy (64 B), 0 already in sync, 1 to remove
```

`junk.xyz` is on the device too, and nothing proposes to delete it.

This is deliberate. Handheld storage is small and full of files that have
nothing to do with your library, and a filter that cannot be argued with is
safer than one that deletes whatever it finds.

## PDAs and other non-audio devices

`nugs` syncs a directory tree onto a device. It does not care what is in the
files, so a folder of documents works as well as a folder of music:

```sh
nugs sync --library ~/Documents/Palm --subdir "My Documents" --filter palm
```

`filter = palm` covers the file types a handheld stores — `.prc`, `.pdb`,
`.pqa` and friends — without needing an `include` list. `filter = ce` does the
same for Windows CE and Pocket PC. See
[Which files get synced](#which-files-get-synced).

### A worked example

Mount the card, copy apps and databases across, and check before doing
anything irreversible:

```sh
# 1. what is on the device?
$ nugs devices
MSC  /media/gabriel/Palm
     free 24 MiB of 128 MiB
     id  /media/gabriel/Palm

# 2. what folders does it have? names vary, so ask
$ nugs folders
/media/gabriel/Palm (MSC) folders:
  Backup
  Card
  My Documents

Sync into one of these with: nugs sync --subdir <name>

# 3. what would happen? plan and dry-run never write anything
$ nugs plan --library ~/palm/apps --subdir Card --filter palm
Device: /media/gabriel/Palm (MSC)
copy    128.0 KiB Address.pdb
copy    16.0 KiB Memo.pdb
copy    64.0 KiB calc.prc

Plan: 3 to copy (208.0 KiB), 0 already in sync
```

`sync --dry-run` does the same check and reports the total:

```
$ nugs sync --library ~/palm/apps --subdir Card --filter palm --dry-run
Device: /media/gabriel/Palm (MSC)

Copying 3 files (208.0 KiB)
(dry run, nothing was written)
```

Keep that in a config and it becomes `nugs status`, `nugs plan`, `nugs sync`:

```ini
library = ~/palm/apps
subdir  = Card
filter  = palm
exclude = tmp
```

### What is supported

- **PDAs that expose their storage as a filesystem.** Palm OS handhelds with an
  SD or CompactFlash card mounted by the host, and Windows CE / Pocket PC
  devices in mass-storage mode. These appear as ordinary MSC volumes, so this is
  the same code path as an MP3 player: no special support is needed, only a
  different `filter`.
- **MP3 players and similar**, on either backend.

### What is not supported

- **Palm HotSync** and **Windows ActiveSync** are proprietary sync protocols,
  not filesystem or MTP. `nugs` cannot speak them. A handheld has to be
  mounted as mass storage instead.
- **macOS.** MSC works; see the note on MTP below.

Nothing in `nugs` has been tested against physical Palm OS or Windows CE
hardware. The MSC code path those devices use is tested continuously against a
fake volume, and the filters are tested directly, but no one has confirmed
their behaviour on real hardware.

## Notes on the two backends

**MSC** works anywhere the device shows up as a filesystem. Detection reads
`/proc/mounts` on Linux and `getmntinfo(3)` on the BSDs and macOS. A mount is
only offered as a device when it is a FAT-family filesystem, or when it sits
under `/media` or `/run/media`, where desktop environments put them. Network
and system filesystems are never offered, so a mounted backup disk is not a
mistake you can make with `--prune`.

On Linux, udisks2 or `udisksctl` is needed to mount the device; `nugs` does not
mount anything itself:

```sh
lsblk -f                        # find the device
udisksctl mount -b /dev/sdb1    # needs no sudo
```

**MTP** uses libmtp. On Linux the device is normally claimed by the kernel's
`mtphptun`/`usb-storage` driver, and libmtp needs to detach it first, which it
can only do as root or via a udev rule. Install `libmtp` and, if transfers are
refused, give yourself permission over the device.

**On macOS, MTP does not work.** The backend compiles, and the tool builds and
runs fine, but macOS claims MTP devices before libusb can see them, so
enumerating one either finds nothing or fails. Use MSC on macOS. This is a
platform limitation, not a configuration problem.

## Testing

```sh
make test
```

The suite exercises the whole MSC backend — discovery, sync, idempotency,
change detection, dry run, prune, `--verify`, pull, config overrides and error
paths — against a temporary directory made to look like a mounted device. It
also covers the file filters directly — every mode, the extension lists,
`max_size`, and the aliases — using non-audio fixtures such as `.prc`, `.pdb`,
`.cab`, `.exe`, `.dll`, `.reg` and `.cpl`. A bug in the filter is invisible to
a suite made only of `.mp3` files, which is exactly why the audio-only tests
would not have caught the filter work. It needs no root, no mounting and no hardware, and runs on Linux, the
BSDs and macOS. Two environment hooks in the MSC backend make that possible:

| Variable | Effect |
| --- | --- |
| `NUGS_TEST_MOUNT` | treat this directory as the one and only device |
| `NUGS_MOUNT_TABLE` | parse this mount table instead of asking the OS |

The second one exercises the Linux discovery rules — octal-escape decoding,
read-only mounts, network and pseudo filesystems — on any platform, so the
Linux paths are tested even when the tests run on macOS. The MTP backend needs
real hardware and is not covered.

### Checking a Linux build from macOS

If you develop on macOS but ship to Linux, you can prove the Linux build
compiles and links without a Linux machine:

```sh
brew install messense/macos-cross-toolchains/x86_64-unknown-linux-gnu
make CC=/opt/homebrew/opt/x86_64-unknown-linux-gnu/bin/x86_64-linux-gnu-gcc \
     MTP=0 TARGET=/tmp/nugs-linux/nugs
```

To include MTP, point `MTP_CFLAGS` and `MTP_LIBS` at a Linux libmtp and libusb
as well — the macOS ones will not link.

The MTP backend is the one part that cannot be verified without a real device.
If something misbehaves there, run with `-v` and report what it says.

## Configuration reference

`~/.config/nugs/nugsrc`, plain `key = value` lines, `#` for comments. `~` is
expanded. Blank lines are ignored.

| Key | Default | Meaning |
| --- | --- | --- |
| `library` | `~/Music` | local root whose layout is mirrored |
| `subdir` | `Music` | device folder the local root maps onto |
| `device` | — | substring of the device label |
| `verify` | `0` | compare contents instead of timestamps |
| `filter` | `audio` | `audio`, `all`, `custom`, `palm` or `ce` |
| `include` | — | extensions for `filter = custom` |
| `exclude` | — | extensions never synced, in any mode |
| `max_size` | `0` | skip anything larger; 0 means no limit |

---

## Roadmap

- iPod Support (most likely just stripping iopenpod and add it into the workflow)
- Palm HotSync and Windows CE/Pocket ActiveSync
- A TUI of some sort
- PSP sync support in standard USB mode (uses MSC)
- NDS and 3DS sync support (Pending...)
- Nokia Symbian device sync support (Uses MSC or MTP)

## Changelog

Nill, First Revision

### 0.0.1

Initial release.

- MSC and MTP backends, one-way sync, opt-in deletions, checksum `--verify`.
- Configurable file filter: `audio` (default), `all` or `custom` with
  `include`/`exclude` extension lists and a `max_size` cap, so the tool can
  mirror a document folder onto a PDA and not just music onto a player.
- `nugs folders` lists the folders a device actually has, for finding the right
  `subdir`.
- `pull` honours the filter and says which one it used.
- `filter = palm` and `filter = ce` cover Palm OS and Windows CE / Pocket PC
  file types, including `.prc`, `.pdb`, `.pqa`, `.cab`, `.exe`, `.dll`, `.reg`
  and `.cpl`.
- `exclude` now applies to `filter = all` as documented, instead of being
  silently ignored.

## Contributing

Gianni Hatzopoulos AKA SudoCherry

## Licence

This project is licensed under the **GNU General Public License, version 3 or
later** (`GPL-3.0-or-later`). 

### What this means for you:
* **Permissions:** You are completely free to download, run, modify, and distribute this software.
* **Conditions:** If you modify or fork this software and distribute it, your modified version **must also be open-sourced under the GPLv3 or later**. You cannot close the source or turn it into proprietary software.
* **Warranty:** This software is provided as-is, without any express or implied warranty.

The full text is in [`LICENSE`](LICENSE), and the canonical version is at
<https://www.gnu.org/licenses/gpl-3.0.html>. Each source file carries the same
notice; the SPDX identifier for this project is `GPL-3.0-or-later`, which
means a fork may be released under GPLv3 or any later version.

## Acknowledgements

Tyler Lee Calender for being the only dude who cared about this. to Gabe Newell, Linus Torvalds and Steve Wozniak who showed me why "free, as in freedom" matters.
