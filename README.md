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
- **Build testing** : runs `dist-configs` inside a toolbox container after
  conflicts (configurable: after every conflict, after every commit, or never).
- **Tree-wide commit handling** : out-of-scope files touched by tree-wide
  commits are automatically excluded.
- **Extensible API replacement rules** : upstream-to-RHEL API mappings
  (e.g. `kzalloc_obj` -> `kzalloc(sizeof(...), GFP_KERNEL)`) are used to
  filter commits during classification and auto-fix build errors. Rules
  are loaded from `api-replacements.json` shipped with `isb`; users can
  add their own in `~/.config/isb/api-replacements.json` without editing
  the tool.
- **Session persistence** : parameters are saved to `.backport/session.json`
  and reused across runs with an interactive single-keypress menu.
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
  if not building natively, e.g. host OS differs from RHEL target)
- [patchpal](https://github.com/camuso/patchpal) (optional, for GUI
  conflict resolution; requires X11/Wayland display)
- [cursor-sdk](https://pypi.org/project/cursor-sdk/) + `CURSOR_API_KEY`
  (optional, for AI conflict explanation via `--ailog` and Cursor AI
  Rule conflict resolution via `--conflict-policy ai`)

## Setting up the Cursor API Key

AI-assisted features (`--ailog`) require the `CURSOR_API_KEY` environment
variable and a Cursor paid subscription. Without them, isb falls back to
heuristic-based conflict descriptions and prints a one-time warning:

    CURSOR_API_KEY not set. Falling back to simpler conflict notes.

isb automatically selects the best available model from the Cursor API
at runtime, so no model name needs to be configured.

1. Install the SDK:

   ```sh
   pip3 install cursor-sdk
   ```

2. Generate an API key at
   [cursor.com/dashboard/api](https://cursor.com/dashboard/api):
   click **New API Key** and copy the full secret immediately
   (it is only shown once; format: `crsr_...`)

3. Save the key to a file and export it from your shell profile:

   ```sh
   # Save the key (paste your key when prompted):
   read -sp "Cursor API key: " key && echo "$key" > ~/.cursor/cursor-worker-api-key && echo

   # Add to ~/.bashrc or ~/.bash_profile:
   [ -f ~/.cursor/cursor-worker-api-key ] && export CURSOR_API_KEY="$(cat ~/.cursor/cursor-worker-api-key)"
   ```

4. Source the profile or open a new terminal:

   ```sh
   source ~/.bashrc
   ```

Verify it works:

```sh
python3 -c "import cursor_sdk; import os; print('OK' if os.environ.get('CURSOR_API_KEY') else 'NOT SET')"
```

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
