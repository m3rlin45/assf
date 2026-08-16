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
assf check   profile.assf                     # stale page dirs / index collisions
assf clean   profile.assf [--dry-run]         # drop unregistered page trees
assf latest  [DIR]                            # newest version per profile (what RS3 loads)
```

Inner paths accept a unique substring; ambiguous matches list the candidates.

For `--oper`, copy the code from an existing entry that computes the same
statistic (`assf report-channels` shows them; the values look like min/max
variants but the enum is AIM's and undocumented).

## How Race Studio 3 actually uses these files

Field notes from tracing RS3 v3.83.48's file I/O (things that will bite you):

- **RS3 loads the highest-timestamp `<profile>.<ts>.assf`** in
  `profiles-available` — and it **silently writes new versions during profile
  loads**, not only on explicit saves. The "newest file" moves around while
  the app runs. Always run `assf latest` before editing; the modifying
  commands warn when a newer version of the same profile exists, because an
  edit to an older version has no visible effect.
- **Page registration** is the `<Layouts>` list in `Profile.xml`
  (`<Layout Idx="N" ...>Name</Layout>`); each registered page maps to a
  top-level `N - Name/` directory. A stale directory sharing an index with a
  registered page makes RS3 **silently drop that page at load** — the classic
  "my added page never saves" symptom. `assf check` finds this; `assf clean`
  fixes it.
- **Plotted-channel visibility** (which channels are graphed) lives in each
  page's top-level `Layout-<Page>.xml` as `<PlotVisible>true` — *not* in the
  nested panel XMLs, which typically list every channel as false.
- **Channels Report** entries are `<MainChan>` blocks in
  `.../ChannelsReport.xml`; a running RS3 picks up edits to this file, while
  layout/visibility edits need a full app restart.
- **Partial-save bug** (present in 3.83.48): a normal "Save profile" only
  serializes the first page and `Profile.xml`, dragging every other page
  along stale — edits on other pages are silently lost. Adding or removing a
  page forces a full clean rewrite of all pages (also purging stale trees).
- `profiles-current/<N>/` folders are per-analysis working copies. RS3 strips
  key files from them on exit; old slots can preserve historical state (e.g.
  a channel selection lost from every archive) — useful for recovery, via
  `assf put`.

## Safety

- Every modifying command first copies the archive into an `assf-backups/`
  subfolder next to it (disable with `--no-backup`). Backups deliberately
  live outside `profiles-available`'s flat namespace so neither RS3's
  profile scan nor a human can mistake them for live versions.
- Modifying commands warn when the target is not the newest version of its
  profile (RS3 only reads the newest).
- Archives are replaced atomically (temp file + rename).
- Edited XML is checked for well-formedness before repacking; malformed input
  leaves the archive untouched.
- Warns if Race Studio 3 is running (WSL: checked via `tasklist.exe`).
  **Close Race Studio 3 before editing archives it may be using.**

## License

[MIT](LICENSE)

## Caveats

- Confirmed working on Race Studio 3 v3.83.48: Channels Report edits made
  with this tool are picked up by the app — even while it is running.
- The app may still overwrite an externally edited archive if you use its own
  "Save profile" afterwards (its save bug re-serializes only the first page);
  re-check with `assf report-channels` after saving from the app.
