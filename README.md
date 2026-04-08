# claude-web-remote

A Claude Code skill that gives you remote browser access to Claude Code + your dev site from any device (iPad, phone, another computer) — with scannable QR codes.

**Zero config.** Just say `/remote` in Claude Code and it auto-detects your project type, generates `claude-remote.sh`, and launches everything.

## What it does

1. Starts a **web terminal** (ttyd) running Claude Code
2. Starts your **dev server** (auto-detected: Flask, Next.js, Vite, Django, static)
3. Creates **Cloudflare quick tunnels** for both — free, no account needed
4. Prints **QR codes** you can scan from your phone/tablet

```
═══════════════════════════════════════════════════════
  CLAUDE CODE REMOTE
═══════════════════════════════════════════════════════

  Claude Code Terminal:
  https://random-words.trycloudflare.com

  [QR CODE]

  Dev Site Preview:
  https://other-random-words.trycloudflare.com

  [QR CODE]

  Press Ctrl+C to stop everything
═══════════════════════════════════════════════════════
```

## Prerequisites

```bash
brew install ttyd cloudflared qrencode
```

On Linux:
```bash
# ttyd: https://github.com/tsl0922/ttyd
# cloudflared: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/
# qrencode
sudo apt install qrencode
```

## Install as Claude Code skill

```bash
# Create the skill directory
mkdir -p ~/.claude/skills/remote

# Copy the skill file
curl -o ~/.claude/skills/remote/SKILL.md \
  https://raw.githubusercontent.com/lunaticfluker/claude-web-remote/main/SKILL.md
```

Then in any project, just tell Claude Code:
```
/remote
```

or:
```
set up remote access so I can work from my iPad
```

Claude will detect your project type and generate the right script.

## Supported project types

| Type | Dev command | Default port |
|------|-----------|------|
| Next.js | `npm run dev` | 3000 |
| Vite / React | `npm run dev` | 5173 |
| Flask | `python app.py` | 5000 |
| Django | `python manage.py runserver` | 8000 |
| Static HTML | `python3 -m http.server` | 8000 |

## Standalone usage (without Claude Code skill)

You can also just grab the shell script template from `SKILL.md` and adapt it manually:

```bash
# Edit SITE_PORT and the dev server command, then:
chmod +x claude-remote.sh
./claude-remote.sh
```

## How it works

- **ttyd** serves your terminal as a web page (with `-W` for writable/interactive mode)
- **cloudflared** creates free quick tunnels — no Cloudflare account needed, URLs are random and temporary
- **qrencode** renders QR codes in the terminal so you can scan from your phone

Tunnel URLs die when the script stops. Re-run to get new ones.

## Notes

- Works in any browser (Safari, Chrome on iPad/phone)
- No Cloudflare account required
- Tunnels are temporary and random — they expire when the script stops
- For persistent sessions, wrap Claude Code in `tmux` before running

## License

MIT
