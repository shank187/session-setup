# session-setup

Set up a 1337 / 42 workstation in one command: GPU-accelerated Brave and VS Code installed in goinfre, your data moved over from the flatpak versions, and your Docker project up and running.

## The problem

On the 1337 workstations (Ubuntu 22.04, kernel 5.15), flatpak apps can't use the GPU. The current flatpak runtime ships Mesa 26, which needs kernel 6.6 or newer, so it falls back to software rendering (llvmpipe). Brave, VS Code and every other flatpak app end up drawing everything on the CPU, which is why they feel slow.

You can check it yourself: open `brave://gpu` in the flatpak Brave and look at the renderer, it says `llvmpipe`.

The official Linux builds of Brave and VS Code use the system graphics drivers and get full GPU acceleration. They don't fit in the 10 GB home quota, so session-setup installs them in goinfre and keeps your profiles in your home.

## Features

- Installs the latest Brave and VS Code in goinfre, verifies their checksums and keeps them up to date
- Copies your flatpak Brave and VS Code data once: logins, saved passwords, bookmarks, open tabs, extensions, settings, history
- Starts rootless Docker and your project's `docker compose` stack
- Opens VS Code on your project and Brave on your frontend
- Adds `code` and `brave` commands and app menu entries, which set the apps up by themselves on a new workstation
- Finds everything at runtime (goinfre, your project, the flatpak data, Node), so the same script works for anyone, on any workstation

## Installation

```sh
git clone https://github.com/shank187/session-setup.git
./session-setup/session-setup
```

The first run installs the script to `~/.local/bin` and adds that folder to your PATH, so afterwards `session-setup` works from any new terminal.

Close the flatpak Brave and VS Code before the first run so your data can be copied. If they're still open, the script tells you and you can simply run it again once they're closed.

## Usage

Run it at the start of every session:

```sh
session-setup
```

On a workstation you haven't used before, it downloads Brave and VS Code (about a minute). On one you have, it only checks for updates.

After that, use "Brave (native)" and "VS Code (native)" from the app menu, or the `brave` and `code` commands. Pin them to the dock in place of the old flatpak entries.

### Options

| Command | What it does |
|---|---|
| `session-setup` | Full setup: install, copy data, start Docker, open the apps |
| `session-setup --prepare` | Install and copy data only |
| `session-setup --no-docker` | Full setup without Docker |
| `session-setup --project DIR` | Use `DIR` as the project (remembered for next time) |
| `session-setup --reimport [vscode\|brave]` | Copy the flatpak data again (the current data is backed up first) |

## How it works

### Storage

| Location | Contents | Lifetime |
|---|---|---|
| Home (`~`) | The script, your profiles and settings | Follows you to every workstation |
| goinfre (`/goinfre/$USER`) | Brave and VS Code, Docker data, backups | Stays on that workstation |

Because Docker data lives in goinfre, your local database starts empty on a new workstation. Keep the data you need in migrations or seeds.

### Data migration

- Done once per app, caches excluded. The flatpak data is never modified, so you can always go back.
- VS Code extensions are shared with the flatpak VS Code rather than copied, to save space.
- If you already have native Brave or VS Code data in `~/.config`, it is kept. `--reimport` replaces it and moves the previous data to `goinfre/backups`.
- Configs that tools created from inside the flatpak terminal (git's global ignore file, for example) are copied to `~/.config` when you don't have them already.

### Project detection

In order: the `--project` option, the git repository you're in (if it has a compose file), the last project used, and finally the most recently used git repository in your home that has a compose file. The page Brave opens is the published port of the frontend/web service in that compose file.

### Also

- Stops `gnome-software`, which uses around 300 MB of RAM in the background (it comes back at next login)
- Limits Brave's disk cache to 500 MB
- Reuses the Node version from nvm, so VS Code terminals don't fall back to the old system Node

## Requirements

A 1337 / 42 Linux workstation. Everything the script uses (bash, curl, unzip, tar, rsync, python3, flatpak) is already installed. Docker is optional.

## Uninstall

```sh
rm ~/.local/bin/{session-setup,code,brave}
rm ~/.local/share/applications/{code,brave}-native.desktop
rm -r ~/.config/session-setup /goinfre/$USER/apps
```

Then remove the `# Added by session-setup` lines from your `.zshrc` or `.bashrc`. Your Brave and VS Code data in `~/.config/BraveSoftware` and `~/.config/Code` is left in place.

## License

[MIT](LICENSE)
