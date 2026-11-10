# herdr-hop

Hop between [herdr](https://herdr.dev) sessions and open folders as workspaces, from fzf popups.

- **Session switcher**: list, attach, create, stop and delete herdr sessions, from a shell (`hop`) or from a popup inside herdr, without detaching by hand.
- **Workspace opener**: fuzzy-pick a project folder and open it as a new workspace of the current session, with a git preview. Works out of the box, and ranks your most used folders first when zoxide is installed.

![Demo](demo/demo.gif)

## Requirements

- herdr >= 0.9.0
- bash, [fzf](https://github.com/junegunn/fzf) >= 0.45 and [jq](https://jqlang.org)
- [zoxide](https://github.com/ajeetdsouza/zoxide) (optional, recommended) to list your most used folders first in the workspace opener

Tested on macOS and on Linux (Debian 12 and 13, aarch64).

## Install

```sh
herdr plugin install nicosantux/herdr-hop
```

herdr shows the manifest and the commands the plugin runs before installing.

### The `hop` command

herdr cannot be launched from inside herdr, so switching sessions needs a small process that keeps running around the herdr client. That process is `hop`: start herdr with it instead of `herdr`.

Add this function to your `~/.zshrc` or `~/.bashrc`. It resolves the installed plugin on every run, so it keeps working after `herdr plugin update`:

```sh
hop() {
  bash "$(herdr plugin list --json | jq -r '.result.plugins[] | select(.plugin_id == "herdr-hop") | .plugin_root')/bin/hop" "$@"
}
```

```sh
hop            # pick a session with fzf and attach to it
hop work       # attach to (or create) the "work" session directly
```

## Keybindings

Plugins cannot register keybindings, so you choose them. Bind the two actions from `~/.config/herdr/config.toml` with `type = "plugin_action"`:

| Action                | What it does                                      |
| --------------------- | ------------------------------------------------- |
| `herdr-hop.session`   | Opens the session switcher popup                  |
| `herdr-hop.workspace` | Opens the folder picker to create a new workspace |

For example, on keys that are free in herdr's default keymap:

```toml
[[keys.command]]
key = "prefix+f"
type = "plugin_action"
command = "herdr-hop.session"
description = "switch session"

[[keys.command]]
key = "prefix+shift+f"
type = "plugin_action"
command = "herdr-hop.workspace"
description = "open folder as workspace"
```

Then run `herdr server reload-config`. Avoid `prefix+s`, which herdr uses for settings by default.

## Usage

### Session switcher

Run `hop` from a shell, or trigger `herdr-hop.session` inside herdr. The current session is tagged with `◆ current`, and the session you came from is listed first, so `enter` hops back and forth between the last two.

| Key | Action |
| --- | --- |
| `enter` | Attach to the selected session, or create the typed one |
| `alt-enter` | Create a session with the typed name, even if it matches an existing one |
| `ctrl-k` | Stop the selected session |
| `ctrl-x` | Delete the selected session (it must be stopped) |
| `esc` | Close |

Typed names have spaces replaced with `-`.

### Workspace opener

Trigger `herdr-hop.workspace` inside herdr. It lists your zoxide folders first, most used on top, followed by the folders in `extra-dirs` (see [Configuration](#configuration)). Folders that already have a workspace in the current session are hidden. The preview shows the folder contents and, for repositories, the branch, how far it is ahead or behind its upstream, the number of changed files and the last commit.

`enter` creates a workspace for the selected folder, labeled with the folder name, and focuses it.

## Configuration

The workspace opener reads the folders to offer from `extra-dirs`, in the plugin config directory. The first time you open it, the file is created with common project roots, so it works without zoxide or any setup:

```text
# Folders offered by the herdr-hop workspace opener, after zoxide's.
# One folder or glob per line: `~/dev/*` lists the folders inside ~/dev.
# Blank lines and comments are ignored, paths that do not exist are skipped.
~/dev/*
~/code/*
~/projects/*
~/src/*
~/work/*
~/repos/*
~/Developer/*
```

Edit it to match where your projects live:

```sh
$EDITOR "$(herdr plugin config-dir herdr-hop)/extra-dirs"
```

Each line is either a folder (`~/dotfiles`) or a glob that lists the folders it matches (`~/work/*/*` for two levels). Hidden folders are only listed when you write them explicitly. Folders appear in file order, after the ones zoxide knows, without duplicates.

## How session switching works

1. `hop` attaches to a session and records its own PID in `${XDG_STATE_HOME:-~/.local/state}/herdr-hop/<session>.pid`.
2. Inside herdr, the session popup finds the `hop` process for the current session, writes the session you picked next to it and sends the herdr client `SIGHUP`. The client restores your terminal and exits; the herdr server and everything running in it keep going.
3. `hop` sees the recorded session and attaches to it in the same terminal.

Detaching normally (`prefix+q` by default) ends `hop` as usual.

### Limitations

- Switching only works for herdr clients started with `hop`. In a session started with plain `herdr`, the popup tells you to detach and run `hop`.
- If two terminals are attached to the same session with `hop`, the switch applies to the one that attached last.
- herdr cannot report which folder a workspace was created in, so the workspace opener compares folders against the working directory of each workspace's first pane.

## Troubleshooting

**The session popup says the session "was not started with hop".** Session switching only works for herdr clients started with `hop`. Detach (`prefix+q` by default) and start herdr again with `hop`.

**`hop` fails with `No such file or directory`.** The function could not find the plugin. Check that `herdr plugin list` shows `herdr-hop` and that your shell loaded the function (open a new terminal or `source` your rc file).

**A popup opens and closes right away.** Your fzf is probably older than 0.45, the minimum version. Check it with `fzf --version`; some distribution packages (for example Debian 12) ship older releases.

**The workspace opener says there are no folders to open.** None of the paths in `extra-dirs` exist and zoxide is not installed. Add the folders where your projects live to `extra-dirs` as described in [Configuration](#configuration), or install [zoxide](https://github.com/ajeetdsouza/zoxide).

## Uninstall

```sh
rm -rf "$(herdr plugin config-dir herdr-hop)" "${XDG_STATE_HOME:-$HOME/.local/state}/herdr-hop"
herdr plugin uninstall herdr-hop
```

Then remove the `hop` function from your shell rc file and the `herdr-hop.*` keybindings from `~/.config/herdr/config.toml`.

## Acknowledgements

herdr-hop is inspired by:

- [tmux-sessionizer](https://github.com/ThePrimeagen/tmux-sessionizer) by ThePrimeagen, the fuzzy "jump to a project" workflow.
- [tmux-sessionx](https://github.com/omerxx/tmux-sessionx) by omerxx, managing sessions from a popup with previews.

## License

[MIT](LICENSE)
