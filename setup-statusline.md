Set up Claude Code statusline to show real-time plan usage limits (current session + weekly) along with current conversation cost and context window usage.

## What this does

Installs a statusline script and configures `~/.claude/settings.json` to display:

```
💰 $0.0023 | 📊 12% ctx | ⏱ 13% (↺4h35m) | 📅 26% (↺Fri 17:59)
```

- `💰` — current conversation cumulative cost (USD)
- `📊` — current context window usage %
- `⏱` — plan session usage % (5-hour rolling window), with time until reset
- `📅` — plan weekly usage % (7-day window), with reset day/time

Data sources:
- Cost and context: injected by Claude Code per response
- Session/weekly limits: fetched from `https://api.anthropic.com/api/oauth/usage` using the OAuth token stored in macOS Keychain, cached for 3 minutes

## Steps

### 1. Write the statusline script

Create `~/.claude/statusline.sh` with the following content (overwrite if exists):

```bash
#!/bin/bash
# Claude Code statusline: cost, context, session limit, weekly limit
# API result cached for 3 minutes

CACHE_FILE="/tmp/claude_usage_cache"
CACHE_TTL=180

fetch_usage() {
  TOKEN_JSON=$(security find-generic-password -s "Claude Code-credentials" -w 2>/dev/null)
  if [ -z "$TOKEN_JSON" ]; then
    echo "⚠ no token"
    return
  fi

  ACCESS_TOKEN=$(echo "$TOKEN_JSON" | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(d.get('claudeAiOauth', {}).get('accessToken', ''))
" 2>/dev/null)

  if [ -z "$ACCESS_TOKEN" ]; then
    echo "⚠ no token"
    return
  fi

  RESPONSE=$(curl -s -m 5 \
    -H "Accept: application/json" \
    -H "Content-Type: application/json" \
    -H "User-Agent: claude-code/2.0.31" \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    -H "anthropic-beta: oauth-2025-04-20" \
    "https://api.anthropic.com/api/oauth/usage" 2>/dev/null)

  if [ -z "$RESPONSE" ]; then
    echo "⚠ API error"
    return
  fi

  echo "$RESPONSE" > "$CACHE_FILE"
  echo "$RESPONSE"
}

get_usage() {
  if [ -f "$CACHE_FILE" ]; then
    CACHE_AGE=$(( $(date +%s) - $(stat -f %m "$CACHE_FILE" 2>/dev/null || echo 0) ))
    if [ "$CACHE_AGE" -lt "$CACHE_TTL" ]; then
      cat "$CACHE_FILE"
      return
    fi
  fi
  fetch_usage
}

SESSION_INPUT=$(cat)
COST=$(echo "$SESSION_INPUT" | jq -r '.cost.total_cost_usd // 0' 2>/dev/null)
CTX_PCT=$(echo "$SESSION_INPUT" | jq -r '.context_window.used_percentage // 0' 2>/dev/null | cut -d. -f1)
COST_FMT=$(printf '%.4f' "${COST:-0}")

DATA=$(get_usage)

if [[ "$DATA" == "⚠"* ]]; then
  echo "💰 \$$COST_FMT | 📊 ${CTX_PCT}% ctx | $DATA"
  exit 0
fi

SESSION_PCT=$(echo "$DATA" | python3 -c "
import sys, json, datetime
d = json.load(sys.stdin)
fh = d.get('five_hour') or {}
pct = fh.get('utilization', 0)
resets = fh.get('resets_at')
if resets:
    try:
        t = datetime.datetime.fromisoformat(resets.replace('Z', '+00:00'))
        now = datetime.datetime.now(datetime.timezone.utc)
        diff = t - now
        mins = int(diff.total_seconds() / 60)
        hrs = mins // 60
        mins = mins % 60
        reset_str = f'{hrs}h{mins:02d}m'
    except:
        reset_str = '?'
else:
    reset_str = '?'
print(f'{int(pct)}%|{reset_str}')
" 2>/dev/null)

WEEKLY_PCT=$(echo "$DATA" | python3 -c "
import sys, json, datetime
d = json.load(sys.stdin)
sd = d.get('seven_day') or {}
pct = sd.get('utilization', 0)
resets = sd.get('resets_at')
if resets:
    try:
        t = datetime.datetime.fromisoformat(resets.replace('Z', '+00:00'))
        day = t.astimezone().strftime('%a %H:%M')
        reset_str = day
    except:
        reset_str = '?'
else:
    reset_str = '?'
print(f'{int(pct)}%|{reset_str}')
" 2>/dev/null)

SESSION_USED=$(echo "$SESSION_PCT" | cut -d'|' -f1)
SESSION_RESET=$(echo "$SESSION_PCT" | cut -d'|' -f2)
WEEKLY_USED=$(echo "$WEEKLY_PCT" | cut -d'|' -f1)
WEEKLY_RESET=$(echo "$WEEKLY_PCT" | cut -d'|' -f2)

echo "💰 \$$COST_FMT | 📊 ${CTX_PCT}% ctx | ⏱ ${SESSION_USED} (↺${SESSION_RESET}) | 📅 ${WEEKLY_USED} (↺${WEEKLY_RESET})"
```

### 2. Make the script executable

```bash
chmod +x ~/.claude/statusline.sh
```

### 3. Configure settings.json

Read `~/.claude/settings.json`. Check if a `statusLine` key already exists.

- If NOT present: add the following at the top level:

```json
"statusLine": {
  "type": "command",
  "command": "/Users/<USERNAME>/.claude/statusline.sh"
}
```

Replace `<USERNAME>` with the actual username (from `$HOME` or `whoami`).

- If already present and pointing to a different script: update it to the new path.
- If already configured correctly: skip, report already configured.

### 4. Test and report

Run a quick smoke test:

```bash
echo '{"cost":{"total_cost_usd":0.0010},"context_window":{"used_percentage":5}}' | ~/.claude/statusline.sh
```

Print the output. Then tell the user:
- What was created/updated
- That they need to **restart Claude Code** for the statusline to take effect
- The statusline refreshes after every AI response; API usage data is cached for 3 minutes
