# GithubToDiscordNotification

## 🔔 Discord Notifications for GitHub Push & Pull Requests

This GitHub Actions workflow automatically sends a **Discord notification** every time a **push** or **pull request** event occurs in your repository.  
It displays commit details, author info, branch, and a direct diff link — all in a sleek Discord embed.

---

### 🧩 Setup Instructions

1. **Create a Discord webhook**
   - Go to your Discord server → select the channel where you want to receive notifications.
   - Click **Edit Channel → Integrations → Webhooks → New Webhook**.
   - Copy the **Webhook URL** (it looks like `https://discord.com/api/webhooks/...`).

2. **Add the webhook URL as a GitHub secret**
   - In your GitHub repository, go to **Settings → Secrets and variables → Actions → New repository secret**.
   - **Name:** `DISCORD_WEBHOOK_URL`  
   - **Value:** paste your Discord webhook URL.

3. **Add the workflow file**
   - In your repository root, create a new folder path:  
     `.github/workflows/`
   - Inside that folder, create a file named:  
     `discord-push.yml`
   - Paste the full YAML code below into that file.

4. **Commit the file and push**
   - After committing, any new **push** or **pull request** will automatically send a formatted message to your Discord channel.

---

### ⚙️ Example Workflow File

```yaml
name: Discord Push & PR Notifications

on:
  push:
    branches: ['**']   # tutto
  pull_request:
    types: [opened, reopened, synchronize, closed]
  workflow_dispatch: {}

permissions:
  contents: read

concurrency:
  group: discord-${{ github.repository }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  notify:
    runs-on: ubuntu-latest
    env:
      DISCORD_WEBHOOK: ${{ secrets.DISCORD_WEBHOOK_URL }}
    # evita run da fork PR (segreti non visibili) e opzionale: ignora bot
    if: >
      (github.event_name != 'pull_request' || github.event.pull_request.head.repo.fork == false) &&
      (github.sender.type != 'Bot')

    steps:
      - name: Early exit if webhook missing
        if: ${{ env.DISCORD_WEBHOOK == '' }}
        run: echo "Secret DISCORD_WEBHOOK_URL not set; no notification sent." && exit 0

      - name: Build Discord payload
        id: payload
        env:
          WEB_URL: ${{ github.server_url }}/${{ github.repository }}
          REPO: ${{ github.repository }}
          REF_NAME: ${{ github.ref_name }}
          ACTOR: ${{ github.actor }}
          SHA: ${{ github.sha }}
          EVENT_NAME: ${{ github.event_name }}
          COMPARE_URL: ${{ github.event.compare }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          PR_TITLE: ${{ github.event.pull_request.title }}
          PR_HTML_URL: ${{ github.event.pull_request.html_url }}
          PR_MERGED: ${{ github.event.pull_request.merged }}
        run: |
          set -euo pipefail
          EVENT_JSON="$GITHUB_EVENT_PATH"

          # skip volontario
          if jq -e '
              (.head_commit.message? // .pull_request.title? // "") | test("\\[skip-notify\\]"; "i")
            ' "$EVENT_JSON" > /dev/null; then
            echo "payload={\"skip\":true}" >> "$GITHUB_OUTPUT"
            exit 0
          fi

          COLOR_SUCCESS=3066993
          COLOR_INFO=3447003
          COLOR_WARN=15105570
          AVATAR_URL="https://github.com/${ACTOR}.png?size=64"

          if [ "$EVENT_NAME" = "push" ]; then
            SHORT_SHA=$(printf "%s" "$SHA" | cut -c1-7)
            COMMITS=$(jq -r '.commits | length // 0' "$EVENT_JSON")
            if [ "$COMMITS" -gt 0 ]; then
              LIST=$(jq -r '.commits[0:5][] | "• \(.message | gsub("\\n"; " ") | sub(" +$"; ""))  [link](\(.url)) — _\(.author.username // .author.name)_"' "$EVENT_JSON")
              [ "$COMMITS" -gt 5 ] && LIST="$LIST\n… and $(("$COMMITS"-5)) more commits"
            else
              LIST=""
            fi
            DIFF_URL="${COMPARE_URL:-}"

            EMBED=$(jq -n \
              --arg title "🚀 Push su $REPO (@$REF_NAME)" \
              --arg url   "${WEB_URL}/commit/${SHA}" \
              --arg author_name "$ACTOR" \
              --arg author_icon "$AVATAR_URL" \
              --arg field_commit "\`$SHORT_SHA\`" \
              --arg field_diff   "$DIFF_URL" \
              --arg field_commits "$LIST" \
              --arg footer "GitHub • $REPO" \
              --arg avatar_url "https://github.githubassets.com/images/modules/logos_page/GitHub-Mark.png" \
              --argjson color $COLOR_INFO \
              --arg ts "$(date --iso-8601=seconds)" '
            {
              username: "Repo Bot",
              avatar_url: $avatar_url,
              allowed_mentions: { parse: [] },
              embeds: [{
                title: $title,
                url: $url,
                color: $color,
                timestamp: $ts,
                author: { name: $author_name, icon_url: $author_icon },
                fields: (
                  [
                    {name:"Author", value: ("**" + $author_name + "**"), inline:true},
                    {name:"Commit", value: $field_commit, inline:true}
                  ] +
                  (if $field_diff != "" then [ {name:"Diff", value: ("[View on GitHub](" + $field_diff + ")"), inline:true} ] else [] end) +
                  (if $field_commits != "" then [ {name:"Commits", value: $field_commits, inline:false} ] else [] end)
                ),
                footer: { text: $footer }
              }]
            }')

          else
            ACTION=$(jq -r '.action // ""' "$EVENT_JSON")
            COLOR=$COLOR_INFO
            TITLE="📦 PR #${PR_NUMBER} ${ACTION} — ${PR_TITLE}"
            if [ "$ACTION" = "closed" ] && [ "${PR_MERGED:-false}" = "true" ]; then
              COLOR=$COLOR_SUCCESS; TITLE="✅ PR #${PR_NUMBER} merged — ${PR_TITLE}"
            elif [ "$ACTION" = "closed" ]; then
              COLOR=$COLOR_WARN; TITLE="⛔️ PR #${PR_NUMBER} closed — ${PR_TITLE}"
            fi

            EMBED=$(jq -n \
              --arg title "$TITLE" \
              --arg url   "$PR_HTML_URL" \
              --arg author_name "$ACTOR" \
              --arg author_icon "$AVATAR_URL" \
              --arg footer "GitHub • $REPO" \
              --arg avatar_url "https://github.githubassets.com/images/modules/logos_page/GitHub-Mark.png" \
              --argjson color $COLOR \
              --arg ts "$(date --iso-8601=seconds)" '
            {
              username: "Repo Bot",
              avatar_url: $avatar_url,
              allowed_mentions: { parse: [] },
              embeds: [{
                title: $title, url: $url, color: $color, timestamp: $ts,
                author: { name: $author_name, icon_url: $author_icon },
                footer: { text: $footer }
              }]
            }')
          fi

          { echo "payload<<JSON"; echo "$EMBED"; echo "JSON"; } >> "$GITHUB_OUTPUT"

      - name: Send to Discord
        if: ${{ steps.payload.outputs.payload && fromJSON(steps.payload.outputs.payload).skip != true }}
        env:
          DISCORD_WEBHOOK: ${{ env.DISCORD_WEBHOOK }}
          PAYLOAD: ${{ steps.payload.outputs.payload }}
        run: |
          set -euo pipefail
          echo "$PAYLOAD" | jq '.' > payload.json

          # Discord field max ~1024 chars -> tronca a 1000
          HAS_COMMITS=$(jq -r '.embeds[0].fields[]? | select(.name=="Commits") | 1' payload.json || echo 0)
          if [ "$HAS_COMMITS" = "1" ]; then
            VAL_LEN=$(jq -r '.embeds[0].fields[]? | select(.name=="Commits") | .value | length' payload.json)
            if [ "$VAL_LEN" -gt 1000 ]; then
              jq '(.embeds[0].fields[]? | select(.name=="Commits") | .value) |= (.[:1000] + "…")' \
                payload.json > payload.json.tmp && mv payload.json.tmp payload.json
            fi
          fi

          code=$(curl -sS -o /tmp/resp.txt -w "%{http_code}" \
            -H "Content-Type: application/json" \
            -d "@payload.json" "$DISCORD_WEBHOOK")
          echo "HTTP $code"
          if [ "$code" -lt 200 ] || [ "$code" -ge 300 ]; then
            echo "Discord webhook failed:"; cat /tmp/resp.txt; exit 1; fi
```
### 🧠 Notes & Privacy

This workflow is **non-intrusive** by design.  
It does **not** mention or ping anyone in Discord (`@everyone`, `@here`, or specific users).

Mentions are explicitly disabled via:

```json
"allowed_mentions": { "parse": [] }
```
This ensures that your repository notifications remain informative, quiet, and spam-free — ideal for teams using shared channels.

### ‼️ N.B.

This solution works well for a single repository on a free GitHub plan. However, if you want to extend and automate it across all repositories, you have two options:
   Upgrade your plan and use built-in GitHub Actions at the organization level.
   Stay on the free plan and use Cloudflare to host the webhooks and automate notifications with a Worker.
