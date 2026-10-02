# nugs

<img width="579" height="579" alt="Screenshot 2026-10-02 at 9 19 05 am" src="https://github.com/user-attachments/assets/585a29f5-10e1-44e9-a13b-364cc63009c2" />

**0.0.2** — mirror a local folder onto a portable device.

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
- [Interrupted copies](#interrupted-copies)
- [Deletions](#deletions)
- [Names a FAT device cannot store](#names-a-fat-device-cannot-store) — why a
  file is skipped instead of copied
- [Which files get synced](#which-files-get-synced) — the `filter` setting, and
  the `palm` and `ce` modes for Palm OS and Windows CE / Pocket PC
- [PDAs and other non-audio devices](#pdas-and-other-non-audio-devices)
- [Notes on the two backends](#notes-on-the-two-backends)
- [Testing](#testing)
- [Configuration reference](#configuration-reference)
- [Versioning](#versioning) — how `--version` and the git tags are kept in step
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
sudo apt install build-essential libmtp-dev libusb-1.0-0-dev
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
ln -sf /path/to/nugs/target/nugs ~/.local/bin/nugs
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
| `--force-prune` | let `--prune` empty the device folder even when the library is empty |
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

A third case keeps a file off the device entirely: when its name is one FAT
cannot hold, it is [skipped with a reason](#names-a-fat-device-cannot-store)
rather than attempted. That is not a disagreement between two copies, and it is
not counted as a failure.

`--verify` compares actual file contents instead, and catches anything the
timestamp shortcut misses: a truncated copy, a file edited in place without
its timestamp changing, a device that mangles metadata. It reads every file
on both sides, so on MTP it means reading the device back — much slower than a
normal sync. Worth using when you suspect something is wrong, not every run.

The trade-off to know about: because of that two-second window, a change made
to a file within two seconds of its previous timestamp, with no size change,
can be missed by a plain `sync`. Run `sync --verify` when that matters.

## Interrupted copies

Every file is written under a scratch name and renamed into place only once it
is complete. A copy that is cut short — cable pulled, card yanked, player
unplugged mid-sync — therefore leaves the destination untouched instead of a
shortened file wearing the right name, which is the failure that matters: it
looks fine, the next `sync` sees a size that nearly matches and skips it, and
the track turns out to be broken much later. `pull` does the same, so a fetch
that dies part way cannot leave a stub in your own library.

The scratch name is unique per run (`track.mp3.nugs4711-0.part`), so two syncs
at once cannot interleave their writes and rename a mixture into place. Nothing
is renamed until its bytes have been pushed to the medium rather than just to
the buffer, which on a USB stick is the difference between surviving an unplug
and not.

Two things are left as they are, deliberately:

- A file that a run was killed mid-copy can still leave its scratch file behind,
  since nothing is there to clean up. The name ends in `.part`, so no filter
  picks it up as a track. `sync --prune` with `filter = all` removes it.
- The timestamp is set after the data is flushed, and FAT players usually ignore
  it anyway.

On MTP the upload goes to a scratch name on the device and is renamed there
afterwards, since a half-transferred object otherwise sits under the name the
library expects. Re-syncing a changed file also removes the copy it replaces;
it used to leave both on the device.

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

### Pruning refuses an empty library

`--prune` deletes every device file the library does not have. If the library
scan comes back with nothing in it, that means *everything* in the folder, and
the player is usually the only other copy of that music. So `sync --prune`
refuses in that case instead of running:

```
$ nugs sync --prune
Device: CLIP [/media/usb/CLIP] (MSC)
nugs: refusing to prune: /home/me/Music holds no files that pass the filter.
      --prune would delete all 128 file(s) from 'Music' on the device.
      Check --library and the filter (currently 'audio'). If you really do want
      the device folder emptied, pass --force-prune.
```

An empty library is nearly always a mistake rather than an intention: the wrong
path, a disk that is not mounted, a home directory that has never synced, or a
filter that matches nothing on disk — a library of `.pdf` files under the
default `audio` filter hits this too, and would otherwise cost you every track
on the player. A library that is missing or unreadable already fails on its own
before the device is touched.

To clear a player on purpose, ask for it:

```sh
nugs sync --prune --force-prune
```

The guard only fires when files would actually be deleted, so an empty library
and an empty device folder still sync and say nothing is to do. A plain `sync`
without `--prune` is never blocked.

It also covers `--dry-run`, which is refused in the same terms rather than
previewing deletions that would then be turned down. A plan is meant to be
truthful about what `sync` will do, and a preview that quietly drops the one
condition that stops it is worse than no preview.

What `--prune` is allowed to propose depends on the
[filter](#which-files-get-synced), since the filter applies to both sides of
the comparison. Under `filter = palm`, a `.xyz` on the device is invisible, so
it is never a deletion candidate.

## Names a FAT device cannot store

The library lives on a normal disk and the player is FAT, which is much stricter.
Your computer will happily hold `What Is It?.mp3`; the device has no way to.
`nugs` checks every path it is about to copy and says which part of the name is
the problem, then skips that file and gets on with the rest:

```
$ nugs sync
Device: CLIP [/media/usb/CLIP] (MSC)

Copying 4 files (38.0 MiB)
  ! Disc 1: Bonus/CD1.wav
      not copied: ':' cannot be stored on a FAT device
[1/4] 100%  Radiohead/OK Computer/Airbag.mp3
  + Radiohead/OK Computer/Airbag.mp3
[2/4] 100%  Radiohead/OK Computer/Lucky.mp3
  + Radiohead/OK Computer/Lucky.mp3
  ! Radiohead/OK Computer/Sub Pop?.mp3
      not copied: '?' cannot be stored on a FAT device
[3/4] 100%  Radiohead/OK Computer/Subway.mp3
  + Radiohead/OK Computer/Subway.mp3

4 copied, 2 skipped
```

They are listed in the position they would have appeared in. A warning also goes
to stderr, so a skipped file is still noticed when the output is piped
somewhere:

```
nugs: warning: 2 file(s) left off the device, starting with:
      Disc 1: Bonus/CD1.wav
      ':' cannot be stored on a FAT device
      rename the file, or drop it from the library. 'nugs plan' lists them all.
```

`nugs plan` shows the same, and `status` counts them.

It is a skip, not a failure: the run still exits 0, because everything it
*could* copy, it copied.

The rules are the ones FAT actually imposes:

- the characters `: ? * " < > |`, plus control characters
- a name ending in a dot or a space, which FAT drops
- a single path component longer than 255 characters

Every component is checked, not just the filename, because one directory called
`Disc 1: Bonus` takes every track under it with it. Spaces, dots in the middle of
a name, and accented or non-Latin letters are all fine, and stay that way.

The check is by name, before any attempt, because these names are simply not
portable and the drivers disagree about what to do with them. Linux and Windows
reject them outright, which makes the copy fail at the very end, after the sync
has already reported every file before it as done. macOS's MS-DOS driver is more
forgiving — it stores `:` and `?` exactly as given, so the file works there — but
it refuses anything over 255 characters all the same. Rather than leave that to
be discovered half-way through a transfer, `nugs` decides before it starts.

Nothing is lost by skipping: the file stays in your library, and it is copied as
soon as you rename it into something the device can hold. Skipped files are
never proposed as deletions, so `--prune` is not affected.

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
MSC  /media/usb/PALM
     free 24 MiB of 128 MiB
     id  /media/usb/PALM

# 2. what folders does it have? names vary, so ask
$ nugs folders
/media/usb/PALM (MSC) folders:
  Backup
  Card
  My Documents

Sync into one of these with: nugs sync --subdir <name>

# 3. what would happen? plan and dry-run never write anything
$ nugs plan --library ~/palm/apps --subdir Card --filter palm
Device: /media/usb/PALM (MSC)
copy    128.0 KiB Address.pdb
copy    16.0 KiB Memo.pdb
copy    64.0 KiB calc.prc

Plan: 3 to copy (208.0 KiB), 0 already in sync
```

`sync --dry-run` does the same check and reports the total:

```
$ nugs sync --library ~/palm/apps --subdir Card --filter palm --dry-run
Device: /media/usb/PALM (MSC)

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

**MTP** uses libmtp. On Linux the kernel may claim the device via the
`usb-storage`/`mtphptun` drivers before libmtp can access it. To allow MTP
transfers, the drivers must be detached so libmtp can claim the device; this
usually requires root privileges or a udev rule to grant permission. Install
`libmtp` and, if transfers are refused, configure device permissions via udev
(or run with sufficient privileges).

**On macOS, MTP does not work.** The backend compiles, and the tool builds and
runs fine, but macOS claims MTP devices before libusb can see them, so
enumerating one either finds nothing or fails. Use MSC on macOS. This is a
platform limitation, not a configuration problem.

## Testing

```sh
make test
```

The suite exercises the whole MSC backend — discovery, sync, idempotency,
change detection, dry run, prune, the empty-library guard, `--verify`, pull,
safe copies, config overrides and error paths — against a temporary directory
made to look like a mounted device.

It also covers the file filters directly — every mode, the extension lists,
`max_size`, and the aliases — using non-audio fixtures such as `.prc`, `.pdb`,
`.cab`, `.exe`, `.dll`, `.reg` and `.cpl`. A bug in the filter is invisible to
a suite made only of `.mp3` files, which is exactly why the audio-only tests
would not have caught the filter work.

And it covers the three ways a run can be a near miss rather than a success:
names a FAT device cannot store, which `tests/fatname.c` also asks the rule
directly about, since APFS and HFS+ refuse a longer component themselves and so
cannot stage the 255 character limit through a real file; unknown config keys,
which must be reported and ignored rather than honoured; and the version, which
must agree with the header, the tags and this file.

It needs no root, no mounting and no hardware, and runs on Linux, the BSDs and
macOS. Two environment hooks in the MSC backend make that possible:

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

A key that is not in that table is reported and ignored, so a typo cannot pass
unnoticed and leave a setting quietly at its default. The likely one is offered:

```
$ nugs status
nugs: warning: ~/.config/nugs/nugsrc:3: unknown key 'fliter', did you mean 'filter'?
nugs: warning: ~/.config/nugs/nugsrc:9: unknown key 'qqqq'.
      known keys are: library, subdir, device, verify, filter, include, exclude, max_size
```

It stays a warning: the rest of the file is read as usual and the exit status is
unaffected. A line with no `=` is reported too.

---

## Roadmap

- iPod Support (most likely just stripping iopenpod and add it into the workflow)
- Palm HotSync and Windows CE/Pocket ActiveSync
- A TUI of some sort
- PSP sync support in standard USB mode (uses MSC)
- NDS and 3DS sync support (Pending...)
- Nokia Symbian device sync support (Uses MSC or MTP)

## Changelog

### v0.0.2

- **Safety fix:** `sync --prune` now refuses to run when the library holds no
  files, instead of reading that as "delete everything in that folder on the
  device". A wrong, unmounted or filter-excluded library can no longer wipe a
  player in one command. `--force-prune` is the deliberate opt-out, for
  clearing a device on purpose. The guard applies to `--dry-run` too, so a
  preview never promises deletions that would be refused.
- Safe copies: every file is written under a scratch name and renamed into
  place once complete, so an interrupted copy can no longer leave a truncated
  file that looks valid. `pull` gains the same treatment, which it lacked
  entirely. Data is flushed to the medium before the rename, and scratch names
  are unique per run.
- Names a FAT device cannot store are detected and skipped, with the reason
  given, instead of being attempted and failing part-way through a sync — after
  every earlier file has already been reported as done. Covers `: ? * " < > |`,
  control characters, a trailing dot or space, and any single path component
  over 255 characters. Every component is checked, so a bad directory name
  catches the tracks under it too. Spaces, dots mid-name and accented letters
  are unaffected. A skipped file is not a failure, is never proposed as a
  deletion, and stays in the library to be copied once renamed.
- An unrecognised key in `nugsrc` is reported by name and line number, with the
  key it was probably meant suggested, instead of the line being read and
  thrown away: `fliter = all` no longer leaves the filter at its default in
  silence. Far enough from every key to guess, it lists them instead. Still a
  warning, not an error.
- `make test` now checks that `--version` agrees with `NUGS_VERSION`, that every
  tag points at a commit that was really that version, that the header is never
  behind an existing tag, and that this README quotes the same number.
- MTP: a re-synced file now replaces the copy already on the device instead of
  being uploaded alongside it.
- Changed dependency reference from `libusb-dev` to `libusb-1.0-0-dev` in
  installation instructions (correct package name on Debian/Ubuntu).

### v0.0.1

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

## Versioning

The version lives in one place that matters: `NUGS_VERSION` in `src/nugs.h`,
which is what `nugs --version` prints. Tags say `vX.Y.Z`; `README.md` says the
same twice, in the first line and as a changelog heading. Four copies is one too
many to keep by hand, so `make test` checks that they agree:

- `--version` reports the value in the header, so a binary built before a bump
  is caught
- every tag points at a commit whose header really was that version
- the header is never behind a tag that exists, which is how a release ends up
  announcing the wrong version
- the README's two mentions match

Cutting a release is therefore: bump `NUGS_VERSION` in `src/nugs.h`, write the
changelog heading, update the first line of the README, commit, then
`git tag vX.Y.Z`. The suite fails if any of those is left out. Nothing derives
the version from git at build time, so a tarball builds and reports the same
number as the repository it came from.

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
