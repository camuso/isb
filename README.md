# isb : Intelligent Subsystem Backport

Automated RHEL kernel subsystem backport tool.

`isb` cherry-picks upstream Linux kernel commits for a given
subsystem into a RHEL or CentOS Stream kernel tree. It discovers relevant
scope paths (source directories, headers, documentation, device-tree
bindings), finds the divergence point between upstream and RHEL, filters
out already-applied commits, and applies the remaining ones with
RHEL-compliant commit messages suitable for merge requests.

## Features

- **Automatic scope discovery** : given a subsystem path (e.g.
  `drivers/char/ipmi`), finds related source, headers, documentation,
  and device-tree binding paths.
- **Divergence detection** : determines which upstream commits are already
  present in the RHEL tree and skips them.
- **Conflict resolution** : four policies:
  - `ai` (Cursor AI Rule) : Cursor Opus agent resolves conflicts
    automatically, then presents a review menu (accept / redo in
    editor / skip) — **default** when cursor-sdk and CURSOR_API_KEY
    are available
  - `patchpal` : launch the [patchpal](https://github.com/camuso/patchpal)
    GUI for interactive three-way merge (fallback default when a display
    is available but cursor-sdk is not)
  - `edit` : open conflicted files in your editor
  - `abort` : stop on any in-scope conflict
- **Build testing** : after conflicts (or every commit / never — configurable),
  prepares the host-arch `.config` + `modules_prepare` when needed, then
  `make M=<subsystem>` in the toolbox. Failures are fail-closed so an
  unprepared tree cannot silently “pass.” Kconfig changes use a fuller
  mkmod-style validation.
- **Tree-wide commit handling** : out-of-scope files touched by tree-wide
  commits are automatically excluded.
- **Non-functional residual skip** : if conflict resolution leaves only
  comment/whitespace edits while upstream had real code changes, the
  commit is skipped (already present or unmergeable), not kept as a
  comment-only backport.
- **Fixes: seek** : after KEEP/OMIT classification, isb searches upstream
  for commits that `Fixes:` a KEEP hash (same idea as patchreview).
  Out-of-scope dependents go into `Omitted-fixes` / `Omitted-fix:` in
  the MR description so they are not reported as missing. In-scope
  dependents that Phase 2 missed are added to KEEP.
- **Inapplicable omit** : before resolving conflicts, isb compares
  upstream functional +/- lines to RHEL ours/HEAD (and any clean staged
  merge). If nothing can land (no matching preimage, or additions
  already present), it omits to `Omitted-fixes` without calling AI/edit.
  The same check runs again after apply/resolve and after build for
  residual comment-only diffs. Unrelated AI/build-fix edits outside
  upstream-touched files do not keep the commit.
- **Skip recommendation** : the pre-resolve missing-prereq gate and
  post-conflict AI analysis both recommend SKIP when many prerequisites
  are missing or hunks are kept-as-RHEL / dropped. The heuristic always
  wins over a mistaken AI ACCEPT. Menus highlight **SKIP RECOMMENDED**
  and offer `s` to omit into `Omitted-fixes`.
- **Resolved-by / Assisted-by** : non-interactive isb AI ends
  `Conflicts:` with `  Resolved-by: isb/cursor` or
  `  Resolved-by: isb/cursor/<model>` when an agent ran. Interactive
  patchpal uses `Assisted-by: patchpal` (not Resolved-by). Manual
  in-scope editor resolutions omit both.
- **New Kconfig detection** : Phase 3 parses each commit's Kconfig diff
  for newly added `config FOO` options. If `redhat/configs` has no
  matching `CONFIG_FOO` snippet, the keep entry is annotated and
  cherry-pick prompts to create
  `redhat/configs/common/generic/CONFIG_FOO` (using the Kconfig
  default, or an explicit y/n/m choice). Any commit that touches
  Kconfig (or creates a snippet) forces an mkmod-style validation:
  regenerate dist-configs, install the host-arch `.config`, run
  `prepare`/`modules_prepare`, then `make M=<subsystem>`.
- **Extensible API replacement rules** : upstream-to-RHEL API mappings
  (e.g. `kzalloc_obj` -> `kzalloc(sizeof(...), GFP_KERNEL)`) are used to
  filter commits during classification and auto-fix build errors. Rules
  are loaded from `api-replacements.json` shipped with `isb`; users can
  add their own in `~/.config/isb/api-replacements.json` without editing
  the tool.
- **Session persistence** : per-backport parameters are saved to
  `.backport/session.json` and reused across runs with an interactive
  single-keypress menu.
- **User Config menu (`C`)** : stable preferences live in
  `~/.config/isb/config.json` — conflict/build policy, editor, ailog,
  AI provider, API key path/env (never the raw secret), and model
  preference lists. Open with `C` from Parameters / session, or when
  prompted by `isb new`.
- **Resumable** : interrupted runs save progress and resume automatically
  with `--continue`.
- **Automatic commit-graph** : on first run, isb writes a git commit-graph
  for both the RHEL and upstream repos if one doesn't exist, dramatically
  speeding up ancestry walks (cherry-pick, merge-base, log) on large repos.
- **Single-commit mode** : `isb pick HASH` cherry-picks one upstream commit
  with full conflict resolution and commit message formatting. Detects
  already-backported commits by searching the RHEL log for the upstream
  hash before cherry-picking.
- **AI conflict explanation & build-fix** : `--ailog` (on by default) calls
  Claude via the Cursor SDK for two purposes:
  1. After every conflict resolution, generates a concise explanation of
     *why* the conflict occurred and *what* the resolution changed,
     appended to the commit message Conflicts stanza and the conflict
     report file.
  2. When a build fails and pattern-based auto-fix cannot resolve it,
     spawns a Cursor agent to read the compiler errors, examine the
     failing source, apply minimal fixes, and rebuild : retrying up to
     3 times before falling back to the manual menu.
  Disable with `--no-ailog`.
- **Hunk-first missing prerequisites** : before resolving in-scope
  conflicts, isb parses `<<<<<<<` / `=======` / `>>>>>>>` hunks and
  blames the upstream parent for the **theirs** lines in each hunk
  (not whole-file post-resolve blame). Commits are ranked by how many
  conflict lines they own. Caps: 50 blame lines per hunk,
  30 distinct commits checked against RHEL history. Excess candidates
  are reported as unchecked — never shown as fake “missing” entries.
  If missing commits are found, isb notifies with options (continue
  resolve anyway / skip / menu / quit) instead of hard-exiting. When a
  single missing commit owns most of a file’s conflict lines, the UI
  marks it **likely blocked**. Per-hunk rankings are fed to Cursor AI
  Rule resolve and `--ailog` explanation prompts.
- **Reviewer-style conflict notes** : the Conflicts stanza in commit
  messages produces descriptions reviewers expect:
  - Dropped architecture hunks named explicitly
    (e.g. `dropped loongarch, riscv hunks`)
  - API renames detected from the diff
    (e.g. `rdmsrl->rdmsrq; in kvm_init()`)
  - Missing prerequisites grouped by series when they share keywords
  - Known upstream-only APIs flagged via `api-replacements.json`
  - Function names and change characterization
    (e.g. `added return value capture in sev_init()`)
  - Ends with `  Resolved-by: isb/cursor[/<model>]` for non-interactive
    isb AI; interactive patchpal gets `Assisted-by: patchpal` instead
- **Commit review with omit toggle** : the interactive commit review
  page (`v` hotkey) shows previously omitted commits with a ✘ mark.
  Toggle any entry by number to restore it to the keep list.
- **Commit exclusion** : `--skip HASH ...` excludes specific commits
  by hash prefix. User-skipped commits (from `--skip`, patchpal, or
  the review menu) are persisted across resets, so resetting to zero
  and replaying will still skip them. Restore via the commit review page.
- **Graceful patchpal exit** : closing patchpal without saving presents
  Retry / Skip / Quit instead of killing the session. Skipped commits
  appear in the commit review page for later restoration.
- **Dry-run mode** : `--dry-run` classifies and lists upstream commits
  without applying anything.
- **Built-in help** : `isb help` shows a summary of all commands and
  options; `isb help <topic>` gives detailed help for any command or
  option (e.g. `isb help conflict-policy`).

## Requirements

- Python 3.12+
- Git
- A RHEL or CentOS Stream kernel repository
- An upstream Linux kernel repository
- [toolbox](https://containertoolbx.org/) (required for build testing
  if not building natively, e.g. host OS differs from RHEL target).
  See **Toolbox path rules** below.
- [patchpal](https://github.com/camuso/patchpal) (optional, for GUI
  conflict resolution; requires X11/Wayland display)
- [cursor-sdk](https://pypi.org/project/cursor-sdk/) + `CURSOR_API_KEY`
  (optional, for AI conflict explanation via `--ailog` and Cursor AI
  Rule conflict resolution via `--conflict-policy ai`)

### Toolbox path rules

Toolbox bind-mounts `$HOME` into the container. Host paths outside
`$HOME` are reached via `/run/host<abspath>` (the host root mount).
isb resolves the RHEL repo with `realpath` and syncs `PWD` to that
path before every `toolbox run`, because toolbox follows `PWD`, not
the process cwd.

| Repo location (after realpath) | How isb runs builds |
| --- | --- |
| Under `$HOME` (including entry via a symlink such as `/work` → `~/work`) | `toolbox run` with cwd/`PWD` set to the realpath under `$HOME` |
| Outside `$HOME` (e.g. `/data/...`) | `toolbox run` + `cd /run/host<realpath>` inside the container |

Custom toolboxes without a `/run/host` mount, or paths the container
cannot see, are not supported — bind-mount the tree explicitly or keep
the repo under `$HOME`. Path/mount failures are treated as infrastructure
errors (AI build-fix is skipped).

## Setting up the Cursor API Key

AI-assisted features require `cursor-sdk` and an API key (paid Cursor
subscription). Without them, isb falls back to heuristic conflict notes.

**Easiest path:** in the isb Parameters menu press `C` → `k` → paste the
key. isb writes `~/.config/isb/api-keys/cursor` with mode `0600` and
records only that path in `config.json` (never the secret).

Manual setup:

1. `pip3 install cursor-sdk`
2. Create a User API key at
   [cursor.com/dashboard/api](https://cursor.com/dashboard/api)
   (copy the full secret from the create dialog — shown once)
3. Either paste via Config (`C` → `k`), or:

   ```sh
   mkdir -p ~/.config/isb/api-keys
   read -sp "Cursor API key: " key && printf '%s\n' "$key" > ~/.config/isb/api-keys/cursor
   chmod 600 ~/.config/isb/api-keys/cursor
   # Optional: also export for other tools
   export CURSOR_API_KEY="$(cat ~/.config/isb/api-keys/cursor)"
   ```

Model preference lists (analysis vs fix) are edited in Config (`m` / `f`).
isb still picks the first listed model that `Cursor.models.list()` reports
as available.

## Installation

Copy or symlink the script to a directory on your `$PATH`:

```sh
ln -s "$PWD/isb" ~/bin/isb
```

Install the man page:

```sh
sudo cp isb.1 /usr/local/share/man/man1/
```

## Usage

Run from inside a RHEL kernel git repository. On first invocation, an
interactive menu prompts for session parameters:

```sh
cd ~/kernels/centos-10
isb --dry-run
```

Subsequent runs reuse the saved session:

```sh
isb
```

Pick a single upstream commit:

```sh
isb pick a8aebe93a4938c0ca1941eeaae821738f869be3d
```

Reset to an earlier backport commit and resume from there:

```sh
isb reset abc123def456
```

Override a session value for one run:

```sh
isb --upstream-version v7.2
```

Get help on any command or option:

```sh
isb help                   # full summary
isb help pick              # details on the pick command
isb help conflict-policy   # details on conflict resolution options
isb help ailog             # details on AI conflict explanation
```

See `man isb` or `isb -h` for the full man page.

## License

Copyright (c) Tony Camuso <tcamuso@redhat.com>
