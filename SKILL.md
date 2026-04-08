---
name: remote
description: Set up remote browser access to Claude Code and dev site preview with QR codes. Use when the user wants to access Claude Code from outside (iPad, phone, another computer).
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# Claude Remote — Browser Access Setup

Set up remote access to Claude Code + dev site preview from any browser, with scannable QR codes.

## Prerequisites

Check and install if missing:
```bash
brew install ttyd cloudflared qrencode
```

## Steps

1. **Detect the project type** — look at package.json, docker-compose.yml, etc. to find the dev server command and port.

2. **Create `claude-remote.sh`** in the project root with this template, adapting SITE_PORT and the dev server command:

```bash
#!/bin/bash
# Claude Remote — access Claude Code + site preview from anywhere
# Uses ttyd (web terminal) + Cloudflare quick tunnels + QR codes

CLAUDE_PORT=7682
SITE_PORT=3000  # Adapt: match your dev server port
CLAUDE_TUNNEL_LOG="/tmp/cloudflared-claude.log"
SITE_TUNNEL_LOG="/tmp/cloudflared-site.log"
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

cleanup() {
    echo ""
    echo "Shutting down..."
    kill $TTYD_PID $CLAUDE_TUNNEL_PID $SITE_PID $SITE_TUNNEL_PID 2>/dev/null
    rm -f "$CLAUDE_TUNNEL_LOG" "$SITE_TUNNEL_LOG"
    exit 0
}
trap cleanup INT TERM

# --- Claude Code terminal ---
echo "Starting Claude Code web terminal on port $CLAUDE_PORT..."
ttyd -W -p $CLAUDE_PORT -t fontSize=14 -t 'theme={"background":"#1a1a2e","foreground":"#e0e0e0"}' \
    bash -c "cd $SCRIPT_DIR && claude" &
TTYD_PID=$!
sleep 2

if ! kill -0 $TTYD_PID 2>/dev/null; then
    echo "Failed to start ttyd"
    exit 1
fi

# --- Dev server ---
echo "Starting dev server on port $SITE_PORT..."
# Adapt: change this to your dev server start command
cd "$SCRIPT_DIR" && npm run dev &
SITE_PID=$!
cd "$SCRIPT_DIR"
echo "Waiting for dev server to start..."
sleep 8

# --- Tunnels ---
echo "Creating Cloudflare tunnels..."
cloudflared tunnel --url http://localhost:$CLAUDE_PORT > "$CLAUDE_TUNNEL_LOG" 2>&1 &
CLAUDE_TUNNEL_PID=$!

cloudflared tunnel --url http://localhost:$SITE_PORT > "$SITE_TUNNEL_LOG" 2>&1 &
SITE_TUNNEL_PID=$!

# Wait for both tunnel URLs
echo "Waiting for tunnel URLs..."
CLAUDE_URL=""
SITE_URL=""
for i in {1..30}; do
    [ -z "$CLAUDE_URL" ] && CLAUDE_URL=$(grep -o 'https://[a-zA-Z0-9-]*\.trycloudflare\.com' "$CLAUDE_TUNNEL_LOG" 2>/dev/null | head -1)
    [ -z "$SITE_URL" ] && SITE_URL=$(grep -o 'https://[a-zA-Z0-9-]*\.trycloudflare\.com' "$SITE_TUNNEL_LOG" 2>/dev/null | head -1)
    if [ -n "$CLAUDE_URL" ] && [ -n "$SITE_URL" ]; then
        break
    fi
    sleep 1
done

echo ""
echo "================================================"
if [ -n "$CLAUDE_URL" ]; then
    echo "  Claude Code:  $CLAUDE_URL"
else
    echo "  Claude Code:  FAILED (check $CLAUDE_TUNNEL_LOG)"
fi
if [ -n "$SITE_URL" ]; then
    echo "  Site Preview: $SITE_URL"
else
    echo "  Site Preview: FAILED (check $SITE_TUNNEL_LOG)"
fi
echo "================================================"

if [ -n "$CLAUDE_URL" ]; then
    echo ""
    echo "--- Claude Code QR ---"
    qrencode -t ANSIUTF8 "$CLAUDE_URL"
fi

if [ -n "$SITE_URL" ]; then
    echo ""
    echo "--- Site Preview QR ---"
    qrencode -t ANSIUTF8 "$SITE_URL"
fi

echo ""
echo "Press Ctrl+C to stop"
echo ""

wait
```

3. **Make executable and run:**
```bash
chmod +x claude-remote.sh
./claude-remote.sh
```

4. **Show the user** the tunnel URLs and QR codes from the output.

## Adaptation Guide

- **Next.js**: `SITE_PORT=3000`, dev command = `cd apps/web && npm run dev`
- **Vite/React**: `SITE_PORT=5173`, dev command = `npm run dev`
- **Static HTML**: Replace dev server block with `python3 -m http.server $SITE_PORT --directory ./public &`
- **Django**: `SITE_PORT=8000`, dev command = `python manage.py runserver`

## Notes

- Access via any browser (Safari, Chrome on iPad/phone works fine)
- Cannot access through the Claude app — it's a chat interface, not a browser
- Tunnel URLs are random and temporary — they die when the script stops
- `-W` flag on ttyd enables writable/interactive mode
- No Cloudflare account needed for quick tunnels
