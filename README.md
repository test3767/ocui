# OCUI

A web panel for running and managing [opencode](https://opencode.ai) sessions across any number of project directories, local or over SSH.

`opencode web` gives you one browser UI bound to whatever directory the server was started in. There is no flag to change that directory, no way to work across several project roots from one page, and no overview of what is running where. OCUI is a single binary that solves those problems: one command, one port, one password.

```
ocui --auth "your-password" --port 1212
```

## Contents

- [Install](#install)
- [Quick start](#quick-start)
- [How it works](#how-it-works)
- [Command line reference](#command-line-reference)
- [The panel](#the-panel)
- [SSH remotes](#ssh-remotes)
- [Public access](#public-access)
- [Mobile and PWA](#mobile-and-pwa)
- [Files and state](#files-and-state)
- [Security](#security)
- [Requirements](#requirements)
- [Limitations](#limitations)

## Install

Download the archive for your platform from the [latest release](https://github.com/sandeepb1/ocui/releases/latest), extract it, and put the binary somewhere on your PATH. Everything the panel needs is embedded in the binary: there is no runtime to install, no `node_modules`, and no assets to copy alongside it.

| Archive | Platform |
| --- | --- |
| `ocui-<version>-macos-arm64.tar.gz` | Apple silicon |
| `ocui-<version>-macos-x64.tar.gz` | Intel Mac |
| `ocui-<version>-linux-x64.tar.gz` | glibc Linux (Debian, Ubuntu, Fedora, RHEL) |
| `ocui-<version>-linux-arm64.tar.gz` | glibc Linux on ARM (Raspberry Pi, Graviton) |
| `ocui-<version>-linux-x64-musl.tar.gz` | musl Linux (Alpine) |
| `ocui-<version>-linux-arm64-musl.tar.gz` | musl Linux on ARM |
| `ocui-<version>-windows-x64.zip` | Windows 10 and later |
| `ocui-<version>-linux-x64-baseline.tar.gz` | glibc Linux, CPUs without AVX2 |
| `ocui-<version>-windows-x64-baseline.zip` | Windows, CPUs without AVX2 |

If a binary exits immediately with an illegal instruction error, your CPU predates AVX2 and you want the `baseline` build.

### macOS and Linux

```sh
tar -xzf ocui-0.3.0-macos-arm64.tar.gz
chmod +x ocui
sudo mv ocui /usr/local/bin/
ocui --version
```

On macOS, Gatekeeper will block the binary the first time because it is not notarised. Either right click the binary and choose Open, or clear the quarantine attribute:

```sh
xattr -d com.apple.quarantine /usr/local/bin/ocui
```

### Windows

Extract `ocui.exe` from the zip and run it from PowerShell or CMD.

### Verifying downloads

Every release includes `SHA256SUMS`. Download it alongside the archives and check them:

```sh
shasum -a 256 -c SHA256SUMS      # macOS
sha256sum -c SHA256SUMS          # Linux
```

## Quick start

```sh
# Local only, password protected
ocui --auth "your-password" --port 1212

# Pre-register some project directories
ocui --auth "your-password" --workspace ~/code/api --workspace ~/code/web

# Reachable on your LAN
ocui --auth "your-password" --host 0.0.0.0

# Reachable from anywhere, via a Cloudflare Quick Tunnel
ocui --auth "your-password" --tunnel

# Include an SSH host that also runs opencode
ocui --auth "your-password" --remote build-box --remote user@192.0.2.10
```

Open the printed URL, sign in, and use "Add workspace" to point OCUI at a project directory. Sessions, terminals and statistics appear from there.

## How it works

OCUI starts one `opencode serve` process on a private loopback port and reverse proxies it. It is the only thing listening publicly.

```
ocui :1212                        the only public listener
  GET /                           the panel: sidebar, tabs, dashboard
      /_ocui/*                    panel assets, panel API, login
      everything else  ------->   127.0.0.1:<random>  (opencode serve)
      /<dir>/session/<id>         the opencode UI, embedded in an iframe
      /api/*, /assets/*, /event, /pty/*/connect (websocket)
```

Two details make this work:

Nearly every endpoint in the opencode API accepts a `directory` query parameter, so a single opencode process can serve any number of project roots. No process per workspace is needed.

The opencode web UI addresses sessions as `/<base64url-of-directory>/session/<id>`, so OCUI can deep link straight into a session for any directory, and embed it.

opencode is mounted at the root of OCUI's origin rather than under a path prefix, because its frontend uses absolute asset paths. Everything OCUI owns lives under the reserved `/_ocui` prefix, so nothing collides.

## Command line reference

```
ocui [options]
```

### Server

| Flag | Default | Description |
| --- | --- | --- |
| `--auth <password>` | none | Require a password. Also reads `OCUI_AUTH`. |
| `--username <name>` | `opencode` | Username for HTTP basic auth, used by scripts. |
| `--port <n>` | `1212` | Public port. Also reads `OCUI_PORT`. |
| `--host <addr>` | `127.0.0.1` | Bind address. Use `0.0.0.0` to expose on your network. Also reads `OCUI_HOST`. |
| `--state <dir>` | see [Files and state](#files-and-state) | Where workspaces, open tabs and the signing secret are stored. |
| `--open` | off | Open a browser once the server is ready. |
| `-h`, `--help` | | Show help. |
| `-v`, `--version` | | Print the version. |

### Workspaces and hosts

| Flag | Description |
| --- | --- |
| `--workspace <path>` | Pre-register a project directory. Repeatable. Workspaces added in the UI persist, so this is only needed for first run or automated deployment. |
| `--remote <host>` | An SSH host that also runs opencode. Accepts anything ssh understands: `host`, `user@host`, or an alias from `~/.ssh/config`. Repeatable. |

### opencode backend

| Flag | Default | Description |
| --- | --- | --- |
| `--opencode-bin <path>` | `opencode` | Path to the opencode executable. |
| `--opencode-url <url>` | none | Attach to an opencode server that is already running instead of starting one. If it is password protected, set `OPENCODE_SERVER_PASSWORD`. |

### Public access

| Flag | Default | Description |
| --- | --- | --- |
| `--tunnel` | off | Expose OCUI on a public HTTPS URL using a Cloudflare Quick Tunnel. One tunnel is created per listener, so SSH remotes get their own URLs. |
| `--cloudflared <path>` | `cloudflared` | Path to the cloudflared executable. |
| `--public-url <url>` | none | Declare the public origin when you put OCUI behind your own proxy or named tunnel. |
| `--insecure-tunnel` | off | Permit `--tunnel` without `--auth`. See [Security](#security) before using this. |

### Environment variables

| Variable | Equivalent |
| --- | --- |
| `OCUI_AUTH` | `--auth` |
| `OCUI_PORT` | `--port` |
| `OCUI_HOST` | `--host` |
| `XDG_STATE_HOME` | Parent of the default state directory |
| `OPENCODE_SERVER_PASSWORD` | Credentials for `--opencode-url` |

## The panel

### Sidebar

Workspaces are collapsible groups with their sessions listed underneath. Each session shows a status dot: muted when idle, a pulsing accent dot while a query is running, amber while the provider is being retried. Status comes from the opencode API and is kept current by its event stream, so the sidebar reflects what is actually happening rather than what happened when the page loaded. Cost and last activity are shown per session.

### Tabs

Opening a session gives it a tab. Tabs are kept mounted and hidden rather than torn down, so switching between them never interrupts a running query and never loses a draft prompt or scroll position. Six panes stay live at once; beyond that the least recently used is dropped, which costs nothing because the session itself lives on the server and reopens instantly.

Middle click closes a tab. Right click gives reload, close others, close all, and open in a new browser tab.

### Chat and terminal views

Any session can be viewed either as the opencode chat UI or as the real opencode terminal interface. The toggle sits at the end of the tab strip on desktop and in the top bar on mobile, and is bound to Cmd+. or Ctrl+.

The terminal view runs `opencode attach --session <id>` in a pty on whichever machine owns the session, so both views drive the same session. Send a prompt in one and it shows up in the other. The pty is created the first time you switch and both panes then stay alive, so toggling afterwards is instant. On a narrow screen OCUI uses opencode's `--mini` interface, which is built for small terminals.

### Shell terminals

Each workspace can open plain shell terminals, rendered with xterm.js against the opencode pty API. They run in the workspace directory, on the workspace's host.

### Dashboard

Totals across every workspace: session count, how many are running, aggregate spend, token usage including cache reads and writes, files touched, lines added and removed. Below that is a per workspace table sorted by last activity, a list of everything currently running, and a live event feed. Numbers come from the same API the sidebar uses, so they agree with it.

### Context menus

Right click, or long press on a touch device, works on sessions, terminals, workspaces, remotes, tabs, the tab strip, the sidebar background and dashboard rows. Menus are rebuilt each time they open, so entries reflect current state.

### Folder picker

Keyboard driven. Type to filter, arrow keys to move, right arrow to descend, left arrow to go up, Cmd+Enter or Ctrl+Enter to add the directory you are in. Paste an absolute path and press Enter to jump straight there. Breadcrumbs, home and recent shortcuts, and git repositories are marked. The same picker browses remote hosts over SSH.

### Theme

Dark and light, using opencode's own design tokens so the panel and the embedded UI look like one product. The toggle also sets opencode's own theme preference, so both halves stay in step.

### Keyboard shortcuts

| Shortcut | Action |
| --- | --- |
| `Cmd/Ctrl + K` | Show the dashboard |
| `Cmd/Ctrl + O` | Add a workspace |
| `Cmd/Ctrl + W` | Close the active tab |
| `Cmd/Ctrl + .` | Toggle chat and terminal view |
| `Escape` | Close a menu, dialog, or the mobile drawer |

## SSH remotes

opencode has no SSH feature of its own. Its only workspace adapter is `worktree`, and the `ssh` transport in its client code belongs to the desktop app. OCUI does the work instead:

```
ssh -L <local>:127.0.0.1:<remote> <host> "opencode serve --port <remote> --hostname 127.0.0.1"
```

That single invocation starts opencode on the remote host and forwards it back, so the tunnel and the remote process share one lifetime. If ssh drops, OCUI reconnects with backoff. The system `ssh` binary is used deliberately, so `~/.ssh/config`, agent forwarding, `ProxyJump` and anything else you have configured all apply.

Each remote gets its own OCUI listener port, starting one above `--port`. This is a requirement rather than a convenience: the opencode frontend uses absolute paths, so a path prefix cannot be rewritten reliably and each server needs an origin of its own. Cookies ignore port numbers, so one sign in covers every listener on the same hostname, and sessions on every host stay live at the same time.

Add hosts with `--remote`, or through "Add SSH host" in the panel. Remote workspaces are browsed and added the same way as local ones.

The remote host needs opencode installed and SSH key authentication that works without a prompt, since OCUI connects with `BatchMode=yes`. Password and passphrase prompts cannot be answered by a background process.

## Public access

```sh
ocui --auth "your-password" --tunnel
```

OCUI runs `cloudflared tunnel --url` against each of its listeners and prints the public URLs on startup:

```
  ocui 0.3.0
  -> http://127.0.0.1:1212
  -> 1 workspace(s), password protected

  -> https://random-words-here.trycloudflare.com  public
  -> https://other-words-here.trycloudflare.com   build-box (ssh)
```

Quick Tunnels need no Cloudflare account, no DNS and no configuration. The URL is ephemeral and changes on every restart.

Each SSH remote gets its own tunnel, because each is served from its own port and a Quick Tunnel maps one URL to one port. The panel is told the mapping, so when you load it over a tunnel it points remote sessions at the remote's public URL rather than a `host:port` address that only exists on your LAN.

For a stable hostname, run your own named Cloudflare tunnel, or any reverse proxy, pointing at `127.0.0.1:1212`, and pass `--public-url https://ocui.example.com` so the panel knows its public origin. Each SSH remote then needs its own ingress rule to its own port.

cloudflared is not bundled. Install it from [Cloudflare's downloads page](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/) or with `brew install cloudflared`.

## Mobile and PWA

The panel is one responsive layout rather than a separate mobile build. Below 768 pixels the sidebar becomes a drawer that can be swiped in from the left edge, behind a top bar carrying the active tab title and the chat and terminal toggle. Dialogs become full height sheets and the dashboard collapses to two columns. Right click menus are also bound to long press, with a movement threshold so scrolling never triggers them.

Layout uses dynamic viewport units so collapsing browser chrome does not clip anything, and safe area insets so content clears the notch and home indicator. Terminals resize when the on screen keyboard opens.

OCUI installs as a progressive web app from any browser that supports it. The service worker is deliberately limited to the panel shell: it caches the page, script, stylesheet and icons network first, and does not intercept anything else, so the embedded opencode UI, the API, event streams and websockets are untouched.

## Files and state

State is a single JSON file holding your workspaces, SSH hosts, open tabs and theme, plus a signing secret used for the session cookie.

| Platform | Location |
| --- | --- |
| Linux and macOS | `~/.local/state/ocui` |
| Linux and macOS with `XDG_STATE_HOME` set | `$XDG_STATE_HOME/ocui` |
| Windows | `%LOCALAPPDATA%\ocui` |

Override with `--state <dir>`. Sessions, message history and everything else belonging to opencode stay wherever opencode keeps them, and removing a workspace from OCUI does not delete anything on disk.

## Security

OCUI runs commands on your machine. Sessions execute tools, and the pty API is a shell. Treat access to the port exactly as you would treat SSH access.

- `--auth` gates everything, including the proxied opencode UI and API, behind a signed HttpOnly cookie. HTTP basic auth is accepted as well, for scripts.
- The signing secret is stored with `0600` permissions and persists across restarts, so restarting does not sign everyone out.
- The opencode process OCUI starts binds to loopback with no password of its own. It is not reachable from outside the machine, and OCUI is the only gate in front of it.
- Without `--auth`, binding to anything other than loopback prints a warning.
- `--tunnel` refuses to start without `--auth`, because a public URL onto OCUI is a public URL onto a shell. `--insecure-tunnel` overrides this if you genuinely intend it.

## Requirements

- [opencode](https://opencode.ai) installed and on your PATH, or pointed at with `--opencode-bin`. OCUI does not bundle it.
- `ssh`, only if you use `--remote`.
- `cloudflared`, only if you use `--tunnel`.

## Limitations

- Quick Tunnel URLs are ephemeral and change on every restart. Use a named tunnel or your own proxy with `--public-url` for anything permanent.
- Each SSH remote consumes a port and, when tunnelling, a tunnel. Behind a reverse proxy, each needs its own ingress rule.
- Remote hosts must accept SSH key authentication without a prompt.
- The binaries are not code signed. macOS Gatekeeper and Windows SmartScreen will warn on first run.

## License

MIT
