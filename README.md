# Desktop GitHub Sync

Desktop GitHub Sync is a small Linux user service that watches text files under
`~/Desktop` and automatically mirrors them to a GitHub repository.

The current default target is:

```text
https://github.com/Narratify/riyo.git
```

Files are mirrored into the target repository under the `desktop/` directory.
For example:

```text
~/Desktop/memo.md
```

is committed as:

```text
desktop/memo.md
```

## What It Does

- Watches `~/Desktop` recursively.
- Detects text file additions, updates, and deletions.
- Ignores common binary formats such as images, videos, archives, PDF files,
  Office documents, and databases.
- Copies matching files into a local working Git repository.
- Commits changes automatically.
- Pushes changes to GitHub.
- Runs as a `systemd --user` service.
- Restarts automatically if the process exits.
- Can start again after reboot when user lingering is enabled.

## Files

```text
bin/desktop-github-sync
systemd/desktop-github-sync.service
```

`bin/desktop-github-sync` is the Python watcher and Git sync process.

`systemd/desktop-github-sync.service` is the user service definition.

## Runtime Paths

The installed service uses these paths:

```text
~/.local/bin/desktop-github-sync
~/.config/systemd/user/desktop-github-sync.service
~/.local/share/desktop-github-sync/repo
```

The working Git repository is stored at:

```text
~/.local/share/desktop-github-sync/repo
```

That local repository is separate from `~/Desktop`. This prevents unrelated
files from the remote repository from being mixed directly into the desktop.

## How It Detects Changes

This implementation uses polling, not cron and not a systemd timer.

The Python process stays running and scans `~/Desktop` every 5 seconds:

```python
POLL_SECONDS = 5
```

After a change is detected, it waits briefly before committing:

```python
DEBOUNCE_SECONDS = 2
```

If a push fails, it retries later:

```python
PUSH_RETRY_SECONDS = 300
```

This is simple and reliable for small to moderate desktop folders. If the
desktop grows to thousands of files, the polling interval can be increased or
the implementation can be changed to an inotify-based watcher.

## Text File Detection

The script includes common text extensions such as:

```text
.txt .md .json .yaml .yml .toml .csv .tsv .html .css .js .ts .py .sh .sql .log
```

It also checks unknown extensions by reading a small chunk of the file and
accepting files that look like UTF-8 or Shift_JIS text.

Common binary extensions are skipped explicitly.

## Installation

From this repository:

```bash
mkdir -p ~/.local/bin ~/.config/systemd/user
cp bin/desktop-github-sync ~/.local/bin/desktop-github-sync
cp systemd/desktop-github-sync.service ~/.config/systemd/user/desktop-github-sync.service
chmod +x ~/.local/bin/desktop-github-sync
systemctl --user daemon-reload
systemctl --user enable --now desktop-github-sync.service
loginctl enable-linger "$USER"
```

`loginctl enable-linger "$USER"` allows the user service manager to start
services after reboot even before an interactive login session.

## GitHub Authentication

This service uses regular Git commands. GitHub authentication must be available
to `git push`.

One simple option is Git's credential store:

```bash
git config --global credential.helper store
```

With credential store, credentials are saved in:

```text
~/.git-credentials
```

That file stores credentials in plain text. For a more secure setup, use a
desktop keyring, Git Credential Manager, or `gh auth login` if GitHub CLI is
installed.

## Operation

Check status:

```bash
systemctl --user status desktop-github-sync.service
```

Follow logs:

```bash
journalctl --user -u desktop-github-sync.service -f
```

Restart:

```bash
systemctl --user restart desktop-github-sync.service
```

Stop:

```bash
systemctl --user stop desktop-github-sync.service
```

Disable:

```bash
systemctl --user disable --now desktop-github-sync.service
```

## Configuration

The current implementation is configured by constants at the top of
`bin/desktop-github-sync`:

```python
DESKTOP = HOME / "Desktop"
REMOTE_URL = "https://github.com/Narratify/riyo.git"
MIRROR_DIRNAME = "desktop"
POLL_SECONDS = 5
```

Edit those values and restart the service to change behavior.

## Current Deployment

On the original machine, this service is installed and running with:

```text
systemctl --user enable --now desktop-github-sync.service
loginctl enable-linger riyo
```

The GitHub push target is:

```text
https://github.com/Narratify/riyo.git
```
