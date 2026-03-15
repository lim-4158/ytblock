# ytblock

A YouTube addiction blocker for macOS that rickrolls you every time you try to visit YouTube.

![rickroll page](https://img.shields.io/badge/rickrolls-∞-ff6b6b)
![macos](https://img.shields.io/badge/platform-macOS-000)
![python](https://img.shields.io/badge/python-3.9+-blue)
![zero deps](https://img.shields.io/badge/dependencies-0-green)
![vibecoded](https://img.shields.io/badge/vibecoded-yes-ff69b4)

> **Warning: This project was vibecoded.** The entire thing — script, server, friction system, security hardening — was built through a conversational back-and-forth with Claude Code (Opus 4.6). No line was written by hand. It was reviewed and audited by AI agents, bugs were found and fixed, but it has **not been battle-tested in production** by a large user base.
>
> **Why you should be careful:**
> - It modifies `/etc/hosts` (a system-critical file that controls DNS resolution)
> - It installs a LaunchDaemon that runs as **root** on ports 80 and 443
> - It adds a self-signed SSL certificate to your **system keychain**
> - A bug in any of these areas could break your network, lock you out, or leave orphaned system-level changes
>
> **Read the code before you install.** It's a single Python file — no hidden dependencies, no minification, no magic. If something goes wrong, `sudo ytblock uninstall` should clean everything up, but "should" is doing some heavy lifting in a vibecoded project. Use at your own risk.

## What it does

- Blocks YouTube at the system level via `/etc/hosts` (all browsers, all apps)
- Serves a local rickroll page with a downloaded video when you try to visit YouTube
- Tracks how many times you've been rickrolled
- Makes it deliberately hard to unblock with a **5-step friction process**
- Auto-reblocks after 30 minutes so you can't binge

## The 5-step disable process

Want to watch YouTube? You'll need to:

1. **Wait 30 minutes** — cooldown timer to let the impulse pass
2. **Type a long confession** — character by character, no copy-paste
3. **Solve 5 math problems** — from arithmetic to algebra (fail = restart from step 1)
4. **Write 100+ words** — journal why you need YouTube right now
5. **Do physical exercise** — pushups, squats, or planks (honor system)

After all 5 steps, YouTube unblocks for **30 minutes only**, then automatically reblocks.

## Requirements

- macOS (uses `/etc/hosts`, `launchd`, and Keychain)
- Python 3.9+ (ships with macOS)
- No other dependencies — the rickroll video (10s, 1MB) is bundled in the repo

## Install

```bash
git clone https://github.com/lim-4158/ytblock.git
cd ytblock
chmod +x ytblock
sudo ./ytblock install
```

The installer will:
1. Copy the script to `~/.youtube-blocker/`
2. Copy the bundled rickroll video
3. Generate and trust an SSL certificate (for HTTPS interception)
4. Block 10 YouTube domains in `/etc/hosts`
5. Start a persistent server daemon via `launchd`
6. Symlink `ytblock` to `/usr/local/bin/`

### Chrome autoplay (optional)

To get the rickroll to autoplay with sound:

```bash
defaults write com.google.Chrome AutoplayAllowlist -array "https://127.0.0.1" "http://127.0.0.1"
```

Restart Chrome after running this. For Safari: Safari > Settings > Websites > Auto-Play > set `127.0.0.1` to "Allow All Auto-Play".

## Usage

```bash
ytblock status           # Check if YouTube is blocked
ytblock disable          # Start the 5-step unblock process
sudo ytblock enable      # Re-block YouTube immediately
sudo ytblock uninstall   # Remove everything
```

### Status output

```
ytblock status
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Status:    BLOCKED
Rickrolls: 42
Hosts:     present
Video:     found
Cert:      found
```

## How it works

```
┌─────────────────────────────────────────────────────┐
│  You type youtube.com                               │
│        │                                            │
│        ▼                                            │
│  /etc/hosts maps youtube.com → 127.0.0.1            │
│        │                                            │
│        ▼                                            │
│  Local HTTPS server on port 443 receives request    │
│        │                                            │
│        ▼                                            │
│  Serves rickroll.html + rickroll.mp4                 │
│  Increments rickroll counter                        │
│        │                                            │
│        ▼                                            │
│  🎵 Never gonna give you up 🎵                      │
└─────────────────────────────────────────────────────┘
```

### Blocked domains

| Domain | What it covers |
|--------|---------------|
| `youtube.com` | Main site |
| `www.youtube.com` | Main site (www) |
| `m.youtube.com` | Mobile site |
| `youtu.be` | Short links |
| `youtube-nocookie.com` | Privacy-mode embeds |
| `www.youtube-nocookie.com` | Privacy-mode embeds (www) |
| `music.youtube.com` | YouTube Music |
| `tv.youtube.com` | YouTube TV |
| `studio.youtube.com` | YouTube Studio |
| `gaming.youtube.com` | YouTube Gaming |

### Persistence

- A `launchd` daemon keeps the server running and restarts it if it crashes
- A watchdog thread checks every 60 seconds that the hosts entries haven't been tampered with
- The server auto-starts on boot

### Security

- The SSL certificate is constrained to YouTube domains only (name constraints prevent misuse)
- The private key is stored with `600` permissions (owner-only)
- State file access uses file locking to prevent corruption
- The server only listens on `127.0.0.1` (not accessible from your network)
- Internal commands are guarded against bypass attempts

## Files

| Location | Purpose |
|----------|---------|
| `~/.youtube-blocker/ytblock` | The script |
| `~/.youtube-blocker/rickroll.mp4` | Downloaded rickroll video |
| `~/.youtube-blocker/cert.pem` | SSL certificate |
| `~/.youtube-blocker/key.pem` | SSL private key |
| `~/.youtube-blocker/state.json` | Rickroll count, block status, timers |
| `~/.youtube-blocker/journal.log` | Your journal entries from step 4 |
| `~/.youtube-blocker/server.log` | Server daemon logs |
| `/etc/hosts` | YouTube domain redirects |
| `/Library/LaunchDaemons/com.local.ytblock.server.plist` | Server daemon config |
| `/usr/local/bin/ytblock` | Symlink for PATH access |

## Uninstall

```bash
sudo ytblock uninstall
```

This removes everything: hosts entries, daemon, certificate, symlink, and the `~/.youtube-blocker` directory.

To also remove the Chrome autoplay policy:

```bash
defaults delete com.google.Chrome AutoplayAllowlist
```

## Limitations

- **macOS only** — uses `/etc/hosts`, `launchd`, and Keychain APIs
- **Firefox** needs `security.enterprise_roots.enabled = true` in `about:config` to trust the cert (or just enjoy the extra friction of the cert warning)
- **Not security software** — a determined user can bypass it. The point is friction, not enforcement. If you really want to watch YouTube, you can `sudo` your way past anything. But you'll have to work for it.
- **Doesn't block YouTube in native apps** — the iOS YouTube app, for example, won't be affected. This blocks browser-based access only.

## License

MIT
