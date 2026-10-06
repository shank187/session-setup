# session-setup

[![lint](https://github.com/shank187/session-setup/actions/workflows/lint.yml/badge.svg)](https://github.com/shank187/session-setup/actions/workflows/lint.yml)

GPU-accelerated Brave and VS Code on 1337 / 42 workstations, set up in one command.

<p align="center">
  <img src="docs/demo.webp" alt="The same animated test page in the flatpak Brave (about 6 frames per second) and in the Brave installed by session-setup (60 frames per second)" width="836">
  <br>
  <sub>The same test page, recorded in both browsers on a 1337 workstation.</sub>
</p>

session-setup installs the official Brave and VS Code builds in goinfre, moves your data over from the flatpak versions, and keeps everything working as you move from one workstation to another.

## Why

The 1337 iMacs run Ubuntu 22.04 with kernel 5.15 and an AMD Radeon GPU. Brave and VS Code are installed as flatpaks, and the flatpak runtime ships Mesa 26, whose AMD driver needs kernel 6.6 or newer. So it can't use the GPU and falls back to software rendering (llvmpipe): everything those apps draw goes through the CPU, the same CPU your compiler, containers and dev servers need.

You can see it in `brave://gpu`:

| Flatpak Brave | Brave from session-setup |
|---|---|
| ![brave://gpu in the flatpak Brave: canvas, compositing and rasterization are software only, OpenGL and WebGL are disabled](docs/gpu-flatpak.png) | ![brave://gpu in the Brave from session-setup: canvas, compositing, rasterization, OpenGL and WebGL are hardware accelerated](docs/gpu-native.png) |

In practice that means:

- Animated or graphics-heavy pages stutter, and the browser competes with your work for CPU time
- WebGL is turned off, so Figma (which requires WebGL), 3D viewers and other WebGL sites don't work
- VS Code runs in a sandbox: its terminal can't see the tools installed on the system, so everything goes through `host-spawn`

The official Linux builds of Brave and VS Code use the system's own graphics drivers, which work with the GPU. They don't fit in the 10 GB home quota, so session-setup puts them in goinfre and keeps your data in your home.

Not sure about your workstation? Open `brave://gpu` in the flatpak Brave. If it says "Software only" like above, you're affected. If it already says "Hardware accelerated", the GPU part doesn't apply to you, though the rest (VS Code outside the sandbox, apps outside your quota) still does.

### Before and after

Measured on a 1337 workstation (Intel i5-7500, AMD Radeon RX 470/580, 8 GB RAM), same test page in both browsers:

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/fps-dark.svg">
  <img src="docs/fps-light.svg" alt="Frames per second. 400 animated elements: flatpak Brave 6, Brave from session-setup 60. Scrolling a long page: 60 in both." width="760">
</picture>

| | Flatpak Brave | Brave from session-setup |
|---|---|---|
| Renderer | llvmpipe (CPU) | AMD Radeon (GPU) |
| WebGL, needed by Figma | Not available | Available |
| Page with 400 animated elements | 6 fps | 60 fps |
| CPU usage during that test | 34% | 32% |
| Scrolling a long page | 60 fps | 60 fps |

Ten times the frame rate for the same CPU usage (60 fps is the screen's limit). Plain scrolling was already smooth; the difference shows on anything animated or graphics-heavy, like design tools, 3D viewers and animated dashboards. With rendering on the GPU, the CPU stays free for your work.

VS Code changes the same way, plus it leaves the sandbox:

| | Flatpak VS Code | VS Code from session-setup |
|---|---|---|
| Renderer | llvmpipe (CPU) | AMD Radeon (GPU) |
| Terminal can run `docker` and your nvm `node` | No, only through `host-spawn` | Yes, it's a regular terminal |
| Extensions | | The same ones, shared with the flatpak VS Code |
| Settings, recent projects, open files | | Copied over on the first run |

WebGL, checked on [get.webgl.org](https://get.webgl.org):

| Flatpak Brave | Brave from session-setup |
|---|---|
| ![get.webgl.org in the flatpak Brave: WebGL is disabled or unavailable](docs/webgl-flatpak.png) | ![get.webgl.org in the Brave from session-setup: your browser supports WebGL, with the spinning cube](docs/webgl-native.png) |

### Setup time

| | Time |
|---|---|
| First run on a workstation (downloads Brave and VS Code) | about 35 seconds |
| Every run after that on the same workstation | about 2 seconds |

Your data is copied from the flatpak apps once, on your very first run.

## Features

- Installs the latest Brave and VS Code in goinfre, verifies their checksums and keeps them up to date
- Copies your flatpak Brave and VS Code data once: logins, saved passwords, bookmarks, open tabs, extensions, settings, history
- Keeps the large app files out of your home quota, and your data in your home where it follows you
- Adds `brave` and `code` commands and app menu entries, which install the apps by themselves on a workstation that doesn't have them yet
- Makes links from other apps open the new Brave, if the flatpak Brave was your default browser
- Closes the old flatpak apps for you when they're still running in the background
- Runs VS Code outside the flatpak sandbox, with a regular terminal and your nvm Node on the PATH
- Can set up IntelliJ IDEA the same way if you use it, out of the sandbox and with more free features (`--add idea`)
- Detects everything at runtime, so the same script works for anyone, on any workstation

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

It makes sure Brave and VS Code are installed and up to date on the workstation you're on, then opens them.

Day to day, use "Brave (native)" and "VS Code (native)" from the app menu, or the `brave` and `code` commands. Pin them to the dock in place of the old flatpak entries.

### Options

| Command | What it does |
|---|---|
| `session-setup` | Install or update the apps, copy your data if needed, open both apps |
| `session-setup --prepare` | Same, without opening the apps |
| `session-setup --close-old` | Close the flatpak Brave and VS Code first, even if they're running in the background |
| `session-setup --reimport [vscode\|brave\|idea]` | Copy the flatpak data again (your current data is backed up first) |
| `session-setup --add idea` | Also set up IntelliJ IDEA, see [below](#intellij-idea-optional) |
| `session-setup --remove idea` | Stop setting up IntelliJ IDEA and remove it from goinfre (your settings are kept) |
| `session-setup open vscode\|brave\|idea [args]` | Start one app, installing it first if needed (this is what `code`, `brave`, `idea` and the app menu entries run) |
| `session-setup --version` | Print the version |

### IntelliJ IDEA (optional)

The workstations also have IntelliJ IDEA Community as a flatpak. If you use it, session-setup can set up the official IntelliJ IDEA the same way:

```sh
session-setup --add idea
```

It's off unless you turn it on, because it's a 1.6 GB download (4.3 GB unpacked) on every workstation you use. Once it's on, every `session-setup` keeps it installed and up to date, after Brave and VS Code have opened so it never holds them up. It doesn't open IntelliJ for you, since it takes 2 to 3 GB of RAM: use the `idea` command or "IntelliJ IDEA (native)" in the app menu. On a workstation that doesn't have it yet, `idea` installs it first, which took about a minute and a half in our test.

What you get over the flatpak:

| | Flatpak IntelliJ | IntelliJ from session-setup |
|---|---|---|
| Version | Community 2025.1 | IntelliJ IDEA 2026.2, kept up to date |
| Terminal, builds and run configurations see | Only the sandbox: no `java`, no system JDKs, no `docker`, `zsh`, `node` or `norminette` | The whole workstation, including Java 11 and 17 in `/usr/lib/jvm` |
| Free features | Community | Community plus Spring Boot, Spring Initializr, Database Tools and SQL, Docker, HTTP Client, JavaScript and TypeScript, React, Lombok and more |
| With a JetBrains license (free for students) | Can't be used | Everything else: Spring Web, Data and Security, Hibernate, Node.js, Kubernetes, ... |
| Caches and indexes | In your home, against the 10 GB quota | In goinfre |

It isn't faster, though, and the newer version is heavier. Brave and VS Code are slow in the flatpak because they draw through its graphics driver; IntelliJ draws its window through the system's display server either way, so leaving the flatpak doesn't speed it up (the same version, 2025.1, ran about as fast in and out of it). Measured on a 1337 iMac with 8 GB of RAM and the same 1,900-file Java project:

| | Flatpak IntelliJ 2025.1 | IntelliJ 2026.2 |
|---|---|---|
| First opening of the project (empty caches) | 69 to 98 s | 116 to 141 s |
| Opening it again | 43 to 72 s | 32 to 127 s |
| Highlighting a 9,600-line file | 18 to 29 s | 30 to 52 s |
| Typing latency | 10 to 16 ms | 17 to 24 ms |
| Memory at peak | 2.0 to 2.3 GB | 2.9 to 3.4 GB |

Both froze for a few seconds now and then while typing, when the post ran out of memory and started swapping. The extra gigabyte is what you'll notice on 8 GB with Brave, VS Code and Docker open. And because its caches are in goinfre, the first opening of a project on each new post is the slow one, where the flatpak keeps its caches in your home. So turn it on for what the table above it gives you, not for speed.

Your settings come with you: the first time, session-setup copies the flatpak IntelliJ settings and plugins to `~/.config/JetBrains`, and IntelliJ imports them itself on its first start (keymap, fonts, live templates, JDKs, recent projects, plugins, memory settings). On that first start it also asks you to accept the JetBrains User Agreement and whether to share usage statistics.

`session-setup --remove idea` turns it off and removes it from goinfre. Your settings stay in `~/.config/JetBrains`.

### Updating

From the folder you cloned:

```sh
git pull && ./session-setup
```

Brave and VS Code are kept up to date by the script; this updates the script itself.

## What it changes on your account

No sudo, nothing outside your home and your goinfre. Specifically:

- Adds `session-setup`, `code` and `brave` to `~/.local/bin` (an existing `code` or `brave` that isn't from session-setup is left alone)
- Adds "Brave (native)" and "VS Code (native)" to `~/.local/share/applications`
- Adds one line to your `.zshrc` or `.bashrc` to put `~/.local/bin` on your PATH
- Creates `~/.config/BraveSoftware`, `~/.config/Code` and `~/.config/session-setup`
- Sets "Brave (native)" as your default browser, only if the flatpak Brave was the default
- Closes the flatpak Brave and VS Code only when you say so (by answering yes, or with `--close-old`)
- Removes the lock Brave leaves in its profile when you leave a post with Brave open, so it starts on the next post
- Only with `--add idea`: adds an `idea` command and an "IntelliJ IDEA (native)" app entry, copies the flatpak IntelliJ settings to `~/.config/JetBrains` once, keeps IntelliJ's caches in goinfre and clears the locks it leaves when it doesn't shut down cleanly
- Puts the apps, a download cache and backups in `/goinfre/$USER`
- Stops the `gnome-software` background service when you run it (it comes back at next login)

The flatpak apps and their data are never modified.

## How it works

What a run does:

```mermaid
flowchart LR
    run([session-setup]) --> apps{Apps missing<br/>or outdated?}
    apps -- yes --> dl[Download to goinfre<br/>and verify SHA-256]
    apps -- no --> first
    dl --> first{First run?}
    first -- yes --> copy[Copy your flatpak data<br/>once, without caches]
    first -- no --> open
    copy --> open([Open Brave and VS Code<br/>on the GPU])
```

### Storage

42 workstations give you two places to keep files, and session-setup uses each for what it's good at:

```mermaid
flowchart LR
    subgraph goinfre["goinfre: hundreds of GB, this workstation only"]
        direction TB
        bapp[Brave]
        vapp[VS Code]
        bak[Backups from --reimport]
    end
    subgraph home["Home: 10 GB, on every workstation"]
        direction TB
        bprof[Brave profile<br/>logins, tabs, extensions, history]
        vdata[VS Code settings and state]
        script[session-setup, code, brave]
    end
    bapp -- uses --> bprof
    vapp -- uses --> vdata
```

| | Home (`~`) | goinfre (`/goinfre/$USER`) |
|---|---|---|
| Size | 10 GB quota | Hundreds of GB |
| Where it's available | On every workstation you log into | Only on the workstation it's on |
| How long it lasts | Permanent | Can be cleaned at any time |
| What session-setup keeps there | The script, your Brave profile, your VS Code settings and state | Brave and VS Code themselves, downloaded extension packages, backups |

The apps are large (about 1.5 GB together) and can be downloaded again at any time, so they live in goinfre. Your data is what matters and has to follow you, so it stays in your home. Home is also the better place for it performance-wise: goinfre is a local hard drive, and it's noticeably slower than home at the many small writes a browser profile makes.

When you log into a workstation whose goinfre doesn't have the apps (a workstation you haven't used, or one whose goinfre was cleaned), the next `session-setup` run, or a click on one of the app entries, downloads them again. Nothing is lost, because your data was never in goinfre.

Exact locations:

| What | Where |
|---|---|
| Brave profile | `~/.config/BraveSoftware/Brave-Browser` |
| Brave cache (limited to 500 MB) | `~/.cache/BraveSoftware` |
| VS Code settings and state | `~/.config/Code` |
| VS Code extensions | Shared with the flatpak VS Code in `~/.var/app/com.visualstudio.code/data/vscode/extensions`, or `~/.vscode/extensions` if you never used the flatpak |
| Brave and VS Code | `/goinfre/$USER/apps` |
| Backups made by `--reimport` | `/goinfre/$USER/backups` |

### Data migration

- Done once per app, caches excluded. The flatpak data is never modified, so you can always go back.
- VS Code extensions are shared with the flatpak VS Code rather than copied, to save space.
- If you already have native Brave or VS Code data in `~/.config`, it is kept. `--reimport` replaces it and moves the previous data to `goinfre/backups`, which only exists on that workstation.
- Configs that tools created from inside the flatpak terminal (git's global ignore file, for example) are copied to `~/.config` when you don't have them already.

### Detected at runtime

- goinfre: `/goinfre/$USER`, or `$GOINFRE` if you set it. Without a goinfre, the apps go to `~/.local/opt` and you get a warning, since that takes about 1.5 GB of your home.
- The flatpak (or snap) Brave and VS Code data, wherever it is in `~/.var/app`.
- The VS Code extensions folder in use.
- Node from nvm, wherever nvm is installed, so VS Code terminals don't fall back to the old system Node.

### Also

- Stops the `gnome-software` background service, which uses around 300 MB of RAM (it comes back at next login)
- Limits Brave's disk cache to 500 MB
- Runs one instance at a time, so running it in a terminal while an app entry sets things up is safe

## FAQ

**It says the flatpak Brave or VS Code is still running.**
Brave can keep running in the background after you close its window. When you run `session-setup` in a terminal, it offers to close it for you; `session-setup --close-old` does it without asking. The app is asked to quit properly first, so Brave saves your tabs, and is only forced if it hasn't quit after 15 seconds. Your data is only copied while the old app is closed, so the copy is consistent.

**Brave says the profile is in use by another Brave process on another computer.**
That happens after you leave a post with Brave still open: Brave doesn't get to remove the lock it keeps in your profile, and on the next post it can't check whether that process is still running. Your home is only used on one post at a time, so session-setup removes a lock left by another post before it starts Brave (since 1.1.1, see [Updating](#updating)). If you still get the message, Brave was started another way; "Unlock profile and relaunch" is safe as long as you're not logged in on the other post. If your home is on a network drive (NFS), session-setup leaves the lock alone, since the other Brave could really be running.

**IntelliJ says "Start Failed" and that a process "is still running and does not respond".**
Like Brave, IntelliJ notes its process ID in your settings folder and leaves it behind when it doesn't shut down cleanly. If that number belongs to another program the next time it starts, IntelliJ takes it for a stuck copy of itself. Started through `idea` or the app entry, session-setup removes those leftovers first, as long as no IntelliJ is running on the workstation.

**Can I go back to the flatpak apps?**
Yes. Their data is untouched, just open them from the app menu. Anything you did in the new apps since the copy won't be there.

**I kept using the flatpak Brave for a while. How do I bring that over?**
`session-setup --reimport brave` copies it again. Your current native profile goes to `goinfre/backups` first.

**Is the download safe?**
VS Code comes from Microsoft's download servers and Brave from its official GitHub releases, both over HTTPS, and their published SHA-256 checksums are verified before anything is installed. The script is a single readable bash file, so you can check what it does before running it.

## Requirements

A 1337 / 42 Linux workstation. Everything the script uses (bash, curl, unzip, tar, rsync, python3, flatpak) is already installed.

## Uninstall

```sh
session-setup --remove idea   # only if you turned IntelliJ IDEA on
rm ~/.local/bin/{session-setup,code,brave}
rm ~/.local/share/applications/{code,brave}-native.desktop
rm -r ~/.config/session-setup /goinfre/$USER/apps
```

Then remove the `# Added by session-setup` lines from your `.zshrc` or `.bashrc`. Your Brave and VS Code data in `~/.config/BraveSoftware` and `~/.config/Code` is left in place.

## License

[MIT](LICENSE)
