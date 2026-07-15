# backport-subsystem

Automated RHEL kernel subsystem backport tool.

`backport-subsystem` cherry-picks upstream Linux kernel commits for a given
subsystem into a RHEL or CentOS Stream kernel tree. It discovers relevant
scope paths (source directories, headers, documentation, device-tree
bindings), finds the divergence point between upstream and RHEL, filters
out already-applied commits, and applies the remaining ones with
RHEL-compliant commit messages suitable for merge requests.

## Features

- **Automatic scope discovery** — given a subsystem path (e.g.
  `drivers/char/ipmi`), finds related source, headers, documentation,
  and device-tree binding paths.
- **Divergence detection** — determines which upstream commits are already
  present in the RHEL tree and skips them.
- **Conflict resolution** — three policies:
  - `patchpal` — launch the [patchpal](https://github.com/camuso/patchpal)
    GUI for interactive three-way merge (default when a display is available)
  - `edit` — open conflicted files in your editor
  - `abort` — stop on any in-scope conflict
- **Build testing** — runs `dist-configs` inside a toolbox container after
  conflicts (configurable: after every commit, or never).
- **Tree-wide commit handling** — out-of-scope files touched by tree-wide
  commits are automatically excluded.
- **Session persistence** — parameters are saved to `.backport/session.json`
  and reused across runs with an interactive single-keypress menu.
- **Resumable** — interrupted runs save progress and resume automatically
  with `--continue`.
- **Single-commit mode** — `--pick HASH` cherry-picks one upstream commit
  with full conflict resolution and commit message formatting.
- **Dry-run mode** — `--dry-run` classifies and lists upstream commits
  without applying anything.

## Requirements

- Python 3.6+
- Git
- A RHEL or CentOS Stream kernel repository
- An upstream Linux kernel repository
- [toolbox](https://containertoolbx.org/) (for build testing)
- [patchpal](https://github.com/camuso/patchpal) (optional, for GUI
  conflict resolution)

## Installation

Copy or symlink the script to a directory on your `$PATH`:

```sh
ln -s "$PWD/backport-subsystem" ~/bin/backport-subsystem
```

Install the man page:

```sh
sudo cp backport-subsystem.1 /usr/local/share/man/man1/
```

## Usage

Run from inside a RHEL kernel git repository. On first invocation, an
interactive menu prompts for session parameters:

```sh
cd ~/kernels/centos-10
backport-subsystem --dry-run
```

Subsequent runs reuse the saved session:

```sh
backport-subsystem
```

Pick a single upstream commit:

```sh
backport-subsystem --pick a8aebe93a4938c0ca1941eeaae821738f869be3d
```

Override a session value for one run:

```sh
backport-subsystem --upstream-version v7.2
```

See `man backport-subsystem` or `backport-subsystem -h` for the full
list of options.

## License

Copyright (c) Tony Camuso <tcamuso@redhat.com>
