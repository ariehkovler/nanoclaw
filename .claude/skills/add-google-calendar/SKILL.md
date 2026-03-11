---
name: add-google-calendar
description: Add Google Calendar integration to NanoClaw. Agents can read events, create/update/delete events, check availability, and manage calendars. Guides through GCP OAuth setup and wires the @cocal/google-calendar-mcp server into the container agent runner.
---

# Add Google Calendar Integration

This skill adds Google Calendar capabilities to the container agents. Once set up, agents can list events, create/update/delete events, check free/busy availability, and manage multiple calendars.

## Phase 1: Pre-flight

### Check if already applied

Check if credentials and code are already in place:

```bash
ls -la google-calendar-credentials.json 2>/dev/null || echo "No credentials file found"
grep -q 'google-calendar' container/agent-runner/src/index.ts && echo "Already wired in" || echo "Not yet configured"
```

If the MCP server entry already exists in `index.ts`, skip to Phase 3 (Setup). Verify tokens:

```bash
ls -la groups/main/google-calendar-tokens.json 2>/dev/null || echo "No tokens — need to authorize"
```

## Phase 2: Apply Code Changes

### Add `google-calendar` to `mcpServers`

Read `container/agent-runner/src/index.ts` and find the `mcpServers` block. Add the `google-calendar` entry:

```typescript
'google-calendar': {
  command: 'npx',
  args: ['-y', '@cocal/google-calendar-mcp'],
  env: {
    GOOGLE_OAUTH_CREDENTIALS: '/workspace/project/google-calendar-credentials.json',
    GOOGLE_CALENDAR_MCP_TOKEN_PATH: '/workspace/group/google-calendar-tokens.json',
    GOOGLE_ACCOUNT_MODE: 'default',
  },
},
```

`/workspace/project` is mounted from the NanoClaw project root (credentials live there). `/workspace/group` is mounted from the group folder (tokens are per-group). No changes to `src/container-runner.ts` are needed.

### Add `mcp__google-calendar__*` to `allowedTools`

In the same file, add `'mcp__google-calendar__*'` to the `allowedTools` array.

### Update group memory (optional)

Append to `groups/main/CLAUDE.md`:

```markdown
## Calendar (Google Calendar)

You have access to Google Calendar via MCP tools. Key tools include:
- `mcp__google-calendar__list-calendars` — list all calendars
- `mcp__google-calendar__list-events` — list events (accepts start/end, calendarId)
- `mcp__google-calendar__create-event` — create a new event
- `mcp__google-calendar__update-event` — update an existing event
- `mcp__google-calendar__delete-event` — delete an event
- `mcp__google-calendar__get-freebusy` — check free/busy for scheduling

Use these when asked about schedule, meetings, appointments, or anything calendar-related.
```

Use `ToolSearch` to discover the full list of available tools after the container is rebuilt.

## Phase 3: Setup

### Check existing credentials

```bash
ls -la google-calendar-credentials.json 2>/dev/null || echo "No credentials file found"
```

If `google-calendar-credentials.json` already exists, skip to "OAuth Authorization" below.

### GCP Project Setup

Tell the user:

> I need you to set up Google Cloud OAuth credentials for Calendar access:
>
> 1. Open https://console.cloud.google.com — create a new project or select existing
> 2. Go to **APIs & Services > Library**, search "Google Calendar API", click **Enable**
> 3. Go to **APIs & Services > Credentials**, click **+ CREATE CREDENTIALS > OAuth client ID**
>    - If prompted for consent screen: choose "External", fill in app name and email, save
>    - Application type: **Desktop app**, name: anything (e.g., "NanoClaw Calendar")
> 4. Click **DOWNLOAD JSON** and save as `google-calendar-credentials.json` in the NanoClaw project root
>
> Where did you save the file? (Give me the full path, or paste the file contents here)

If user provides a path, copy it:

```bash
cp "/path/user/provided/google-calendar-credentials.json" ./google-calendar-credentials.json
```

If user pastes JSON content, write it to `./google-calendar-credentials.json`.

### OAuth Authorization

Tell the user:

> I'm going to run the Google Calendar authorization. A browser window will open — sign in and grant access. If you see an "app isn't verified" warning, click "Advanced" then "Go to [app name] (unsafe)" — this is normal for personal OAuth desktop apps.

Run the authorization:

```bash
GOOGLE_OAUTH_CREDENTIALS=./google-calendar-credentials.json \
GOOGLE_CALENDAR_MCP_TOKEN_PATH=./groups/main/google-calendar-tokens.json \
npx @cocal/google-calendar-mcp auth default
```

Verify tokens were saved:

```bash
ls -la groups/main/google-calendar-tokens.json && echo "Authorization successful!" || echo "ERROR: token file not found — try again"
```

### Build and restart

Clear stale per-group agent-runner copies (they only get re-created if missing, so existing copies won't pick up the new Calendar server):

```bash
rm -r data/sessions/*/agent-runner-src 2>/dev/null || true
```

Rebuild the container (agent-runner changed):

```bash
cd container && ./build.sh
```

Then compile and restart:

```bash
npm run build
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
# Linux: systemctl --user restart nanoclaw
```

## Phase 4: Verify

Tell the user:

> Google Calendar is connected! Send this in your main channel:
>
> `@Andy what's on my calendar today?` or `@Andy do I have any meetings this week?`

Check logs if needed:

```bash
tail -f logs/nanoclaw.log
```

Look for lines mentioning `google-calendar` in the MCP init output.

## Troubleshooting

### OAuth token expired

Re-authorize:

```bash
GOOGLE_OAUTH_CREDENTIALS=./google-calendar-credentials.json \
GOOGLE_CALENDAR_MCP_TOKEN_PATH=./groups/main/google-calendar-tokens.json \
npx @cocal/google-calendar-mcp auth default
```

### Container can't find credentials

Verify the file is in the project root:

```bash
ls -la google-calendar-credentials.json
```

The container mounts the project root at `/workspace/project`, so the path inside is `/workspace/project/google-calendar-credentials.json`. If stored elsewhere, move the file or update `GOOGLE_OAUTH_CREDENTIALS` in `index.ts`.

### Wrong Google account

Delete the token file and re-authorize:

```bash
rm groups/main/google-calendar-tokens.json
```

Then re-run the auth command above.

### Group tokens missing for a non-main group

Tokens are per-group. Re-run auth with a different token path for each group:

```bash
GOOGLE_OAUTH_CREDENTIALS=./google-calendar-credentials.json \
GOOGLE_CALENDAR_MCP_TOKEN_PATH=./groups/<group-name>/google-calendar-tokens.json \
npx @cocal/google-calendar-mcp auth default
```

## Removal

1. Remove `'google-calendar'` from `mcpServers` in `container/agent-runner/src/index.ts`
2. Remove `'mcp__google-calendar__*'` from `allowedTools`
3. Remove the Calendar section from `groups/*/CLAUDE.md` (if added)
4. Delete credentials and tokens (optional): `rm google-calendar-credentials.json groups/*/google-calendar-tokens.json`
5. Rebuild and restart:
   ```bash
   rm -r data/sessions/*/agent-runner-src 2>/dev/null || true
   cd container && ./build.sh && cd .. && npm run build
   launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
   # Linux: systemctl --user restart nanoclaw
   ```
