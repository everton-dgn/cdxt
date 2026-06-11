# Troubleshooting

Common problems when installing or running `cdxt`, with diagnosis steps and fixes.

## `command not found: cdxt`

The shell cannot find the command. Check, in order:

1. **Did you open a new terminal?** Aliases added to `~/.zshrc` / `~/.bashrc` only take effect in new shells (or after `source ~/.zshrc`).
2. **Is the alias in the right rc file?** Run `echo $SHELL`. If it ends in `zsh`, the alias belongs in `~/.zshrc`; if it ends in `bash`, in `~/.bashrc`. A common mistake is adding the alias to `~/.zshrc` while actually using bash.
3. **Does the target exist and is it executable?**

   ```bash
   ls -l ~/.codex/scripts/cdxt
   chmod +x ~/.codex/scripts/cdxt   # if the x bit is missing
   ```

4. **Check what the shell resolves:**

   ```bash
   type cdxt
   ```

   This tells you whether `cdxt` is an alias, a file on the PATH, or unknown.

## The plain text menu shows up instead of the fzf UI

This is expected in two situations:

- **fzf is not installed**, or
- **fzf is older than 0.71.** The visual menu uses `--id-nth` (added in fzf 0.71.0) to keep the cursor in place when a toggle refreshes the list. The script detects older versions and falls back to the text menu instead of failing.

Check your version:

```bash
fzf --version
```

Distro packages are frequently outdated (Ubuntu's `apt` ships a version far below 0.71). Install a recent fzf with `brew install fzf` (macOS) or download the binary from the [fzf releases page](https://github.com/junegunn/fzf/releases) (Linux/WSL).

The text menu is fully functional — fzf only adds the nicer interface.

## `Config not found: ...`

The script could not find the Codex config file. Possible causes:

- **Codex CLI is not installed or has never been run.** The config is created by Codex itself. Run `codex` once first.
- **Your config lives somewhere else.** Point the script at it:

  ```bash
  CODEX_CONFIG="/path/to/config.toml" cdxt
  ```

  Or export `CODEX_HOME` if your whole Codex directory is elsewhere.

## `No MCP servers or plugins found in ...`

The config exists but contains no sections the script recognizes. It looks for exactly these two shapes:

```toml
[mcp_servers.context7]
command = "npx"
args = ["-y", "@upstash/context7-mcp"]

[plugins."coderabbit@marketplace"]
enabled = true
```

If your MCP servers are defined in a different file (for example, a project-level config), point `CODEX_CONFIG` at that file.

## An item shows `???` instead of ON/OFF

The section's `enabled` key has a value other than a bare `true` or `false` (for example, a quoted string like `enabled = "true"`). The script refuses to toggle these to avoid corrupting your config. Fix the line manually in `config.toml` so it reads `enabled = true` or `enabled = false`, then run `cdxt` again.

## I toggled something but Codex still shows the old tools

This is how Codex works, not a bug in the script: a running Codex session loads its tool list at startup and does not watch the config file. Restart Codex or open a new session after toggling.

## How do I undo a change?

Every write is preceded by a timestamped backup. List them newest-first and copy one back:

```bash
ls -t ~/.codex/config.toml.backups/
cp ~/.codex/config.toml.backups/config.toml.<timestamp> ~/.codex/config.toml
```

## Old backups keep piling up

Pruning (keeping only the newest 20 by default) requires the `trash` command — the script never deletes files permanently. Install it (`brew install trash` on macOS, `trash-cli` package on most Linux distros) or clear the backup directory manually. You can also raise or lower the retention with `CODEX_CONFIG_BACKUP_KEEP`.

## Windows / WSL issues

### "Can I run it in PowerShell or Git Bash?"

No. The script is zsh-only, and neither PowerShell, cmd, nor Git Bash provides zsh. Use WSL (Windows Subsystem for Linux). Inside WSL:

```bash
sudo apt install zsh
```

zsh does **not** need to be your login shell — the script's shebang (`#!/usr/bin/env zsh`) takes care of running under zsh.

### Editing the config of a native-Windows Codex from WSL

If Codex runs on Windows itself (not inside WSL), its config is at `C:\Users\<you>\.codex\config.toml`, which WSL sees as `/mnt/c/Users/<you>/.codex/config.toml`:

```bash
CODEX_CONFIG="/mnt/c/Users/<you>/.codex/config.toml" cdxt
```

### Sections are not detected on a Windows-side config (CRLF)

The parser matches section headers as exact whole lines. If the file uses Windows line endings (CRLF), every line carries a trailing `\r` and nothing matches — you will see `No MCP servers or plugins found` even though the sections are there.

Detect it:

```bash
file /mnt/c/Users/<you>/.codex/config.toml
# "... with CRLF line terminators" -> that is the problem
```

Fix it by converting the file to LF (Codex on Windows reads LF files fine):

```bash
sudo apt install dos2unix
dos2unix /mnt/c/Users/<you>/.codex/config.toml
```

## `Failed to update ...`

The script could not complete a write. The write path is: create the backup directory, copy the config into it, rewrite the config to a temporary file, then move the temporary file over the original. Your original config is only replaced by that final move — if the script dies earlier, the file is untouched (and a backup copy may already exist in the backup directory).

Check, in order:

1. **Permissions** — you need write access to both the config file and the backup directory:

   ```bash
   ls -ld ~/.codex ~/.codex/config.toml ~/.codex/config.toml.backups
   ```

2. **Disk space** — `df -h ~`
3. **A custom `CODEX_CONFIG_BACKUP_DIR`** — if you overrode it, make sure the path is valid and creatable (`mkdir -p "$CODEX_CONFIG_BACKUP_DIR"`).

## `Could not find enabled=true/false in [...]`

Same root cause as the `???` status above, hit from the non-interactive path (`cdxt toggle N` / `cdxt set N STATE`): the section's current `enabled` value is not a bare boolean, so the script refuses to write. Fix the value manually once, and toggling will work from then on.

## Something else?

Open an issue with:

- the exact command you ran and the full output
- `zsh --version`, `fzf --version` (if installed), and your OS
- the relevant section of your `config.toml` (redact anything sensitive)
