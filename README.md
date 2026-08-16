# assf

A small command-line editor for AIM Race Studio 3 **`.assf`** analysis-profile
archives.

## What is an `.assf` file?

Race Studio 3 stores each saved analysis profile as a timestamped `.assf` file
under `C:\AIM_SPORT\RaceStudio3\user\profiles-available\`. An `.assf` is a
plain zip archive of the profile's page tree:

```
0 - Time-Distance/...
1 - Split Times Report/...
2 - Channels Report/0 - Container/0 - Channels Report/ChannelsReport.xml
...
Profile.xml            (profile-wide settings: colors, scales, channel styles)
<timestamp>.txt        (version marker)
```

This tool lets you inspect and edit those XML files directly and repacks the
archive in a way Race Studio 3 accepts (entry order, compression, and
timestamps of untouched files are preserved byte-for-byte).

## Why

As of Race Studio 3 v3.83.48 (and builds since roughly June 2026), **"Save
profile" only serializes the first analysis page and `Profile.xml`** — edits
made on any other page (Channels Report, Suspension Analysis, Data-Movies, …)
are silently discarded. Verified by tracing the app's file writes during
controlled save experiments. Until AIM fixes it, this tool is a way to get
edits into the pages the app refuses to save.

## Install

Single-file Python 3 script, stdlib only:

```bash
ln -s "$(pwd)/assf" ~/bin/assf   # or copy anywhere on your PATH
```

## Usage

```bash
assf list    profile.assf                     # contents grouped by page
assf cat     profile.assf ChannelsReport      # print one inner file
assf extract profile.assf -d out/             # unzip everything
assf edit    profile.assf ChannelsReport      # open in $EDITOR, repack
assf put     profile.assf ChannelsReport local.xml   # replace with a local file
assf report-channels profile.assf             # show Channels Report channels
assf add-report-channel profile.assf "GPS Speed" --oper 2
assf remove-report-channel profile.assf "GPS Speed" --oper 2   # omit --oper to drop all entries
```

Inner paths accept a unique substring; ambiguous matches list the candidates.

For `--oper`, copy the code from an existing entry that computes the same
statistic (`assf report-channels` shows them; the values look like min/max
variants but the enum is AIM's and undocumented).

## Safety

- Every modifying command first copies the archive to
  `<name>.bak-YYYYmmdd_HHMMSS` (disable with `--no-backup`).
- Archives are replaced atomically (temp file + rename).
- Edited XML is checked for well-formedness before repacking; malformed input
  leaves the archive untouched.
- Warns if Race Studio 3 is running (WSL: checked via `tasklist.exe`).
  **Close Race Studio 3 before editing archives it may be using.**

## License

[MIT](LICENSE)

## Caveats

- Race Studio 3's exact load behavior for non-first pages is still being
  mapped out; keep the automatic backups until you've confirmed the app picks
  up your edit.
- Tested on Race Studio 3 v3.83.48 profile archives.
