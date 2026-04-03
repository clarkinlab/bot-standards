# Bot Development Standards
**Maintained by:** Matt (human) via Claude  
**Last Updated:** 2026-04-03  
**Purpose:** Junior developer guide for OpenClaw bots (Matt & Troy). Read this before starting any coding task.

---

## 1. General Behavior Rules

- **Verify before coding.** Always test what an API actually returns before writing code against it. Use curl to check real responses. Never assume field names or response structure.
- **One thing at a time.** Build one component, confirm it works, then move to the next. Do not build everything at once.
- **Best practice over easy path.** Always choose the cleanest, most secure solution even if it requires more work. Do not take shortcuts.
- **Show your plan first.** For any significant task, outline the approach and wait for approval before writing code.
- **Never use `allow-always` on approval prompts.** Always use `allow-once`. Never ask the human to click `allow-always`.
- **When something is broken, diagnose before rewriting.** Check what the API returns, check the console errors, check the network tab. Identify the root cause before changing code.

---

## 2. Security Rules — Non-Negotiable

- **Never put credentials in frontend code.** API keys, tokens, passwords belong only in server-side code (`index.js` or equivalent). Never in HTML, never in JavaScript that runs in the browser.
- **Never add SSH keys to the Unraid host.** Do not attempt to add public keys to `/root/.ssh/authorized_keys` or any authorized_keys file. Ever.
- **Never modify `/root/.openclaw/openclaw.json`.** This is the OpenClaw config file. Editing it can crash the container. Matt handles all config changes manually.
- **Never publish container images to public registries with credentials baked in.** Build locally only.
- **Always use the bot PAT for GitHub operations.** The correct token starts with `github_pat_11CA5JJWI...NHG...`. Never use personal `ghp_` tokens. Never paste tokens in deploy commands using the wrong account.

---

## 3. GitHub & Deployment Rules

- **GitHub is the source of truth.** All code lives in GitHub. Never consider a task complete until it is committed and pushed.
- **The correct deploy pattern for the dashboard is:**
```bash
curl -H "Authorization: token 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" \
  -H "Accept: application/vnd.github.v3.raw" \
  -L "https://api.github.com/repos/littledrummerboybot/unraid-dashboard/contents/public/[filename]" \
  -o /mnt/user/appdata/openclaw-portal/dashboard/public/[filename]
```
- **Never use `ghp_` tokens in deploy commands.** These are personal account tokens, not the bot account.
- **Always update project markdown files after completing work.** `dashboard-project.md` must reflect current state.
- **Commit messages must be descriptive.** Not "update" — use "Fix RAM card parsing to use results[0].series tags structure".

---

## 4. JavaScript Best Practices

- **Never pass complex objects through HTML onclick attributes.** This breaks with special characters and quotes. Instead use global variables:
```javascript
// WRONG — breaks when object contains quotes or special chars
onclick="drillCPU(${JSON.stringify(percpu)})"

// CORRECT — store globally, reference by name
let _percpu = null;
// in loadCPU():
_percpu = percpu;
// in HTML:
onclick="drillCPU()"
// in drillCPU():
function drillCPU() { const percpu = _percpu; ... }
```
- **Always use data attributes for simple values in onclick:**
```javascript
// For simple strings/numbers:
onclick="drillRAM(this.dataset.name)" data-name="${c.name}"
```
- **Always verify JSON structure before parsing.** Log the raw response first, confirm field names, then write the parser.
- **Handle errors gracefully.** Every API call needs a try/catch and a user-visible error state. Never let the UI silently fail.
- **Use `const` and `let`, never `var`.**
- **Always check `response.ok` before parsing JSON.** A 400/500 response is valid HTTP but invalid data.

---

## 5. InfluxDB Specifics

- **InfluxDB v1 query API returns columns as `["time", "last"]`** — NOT aliased names even if you write `SELECT x AS myalias`. Always access values by index not by name.
- **Container names are in `tags` not columns:**
```javascript
// CORRECT
const name = series.tags.container_name;
const value = series.values[0][1]; // index 1 = the measurement value
```
- **Always test queries with curl before building UI:**
```bash
curl -s "http://192.168.1.17:3002/api/influx/query?db=telegraf&q=YOUR_QUERY_HERE"
```
- **Use the v1 query API (`/query`) not the v2 Flux API** for the dashboard — it's simpler and already working.
- **The telegraf bucket contains these measurements:** `cpu`, `mem`, `disk`, `docker`, `docker_container_cpu`, `docker_container_mem`, `docker_container_blkio`, `docker_container_net`.

---

## 6. API Reference (Dashboard Project)

All credentials live in `server/index.js` server-side. The browser only calls `/api/*` proxy endpoints.

| Service | Proxy Endpoint | Notes |
|---------|---------------|-------|
| Glances | `/api/glances/*` | Real-time stats, no auth needed |
| InfluxDB | `/api/influx/query` | v1 query API, GET only |
| Plex | `/api/plex/*` | Requires X-Plex-Token header |
| Tautulli | `/api/tautulli` | Better history data than Plex native |
| Overseerr | `/api/overseerr/*` | Pending requests only |
| Home Assistant | `/api/ha/*` | Bearer token auth |
| Anthropic | `/api/anthropic` | Coming soon — CORS blocked in browser |
| IP Geo | `/api/geo/:ip` | ip-api.com, no key needed |

---

## 7. Docker & Unraid Rules

- **Always check for existing containers before suggesting a new one.** Run `docker ps -a` to see what's already running. Many common services (Grafana, InfluxDB, Telegraf, etc.) may already be installed. Never suggest installing something that's already there.
- **Always use Unraid Community Apps templates when installing new containers.** Do not suggest raw `docker run` commands for new services — Unraid templates handle paths, ports, and restarts correctly and survive Unraid upgrades. Only use `docker run` for custom containers with no available template.
- **Never recreate containers unnecessarily.** Use `docker restart` for config changes, only rebuild images when code changes.
- **Always use `unless-stopped` restart policy** on all containers.
- **Config files must be bind-mounted to `/mnt/user/appdata/[container]/`** so they survive container recreation.
- **The bot's workspace path inside the container is `/home/node/.openclaw/workspace/`** — this is NOT mounted to the host. Do not rely on workspace writes persisting. Use GitHub instead.
- **To verify a container is reading the correct config:**
```bash
docker exec openclaw-matt cat /root/.openclaw/openclaw.json | jq '.gateway'
```
- **If openclaw-matt enters a restart loop:**
```bash
docker logs openclaw-matt --tail 50
```
A config validation error is the most likely cause.

---

## 8. Architecture Principles

- **Static HTML dashboards must use a backend proxy** for any API on a different origin/port. Direct browser calls to `192.168.1.17:8086` from a page served on port 83 will be blocked by CORS.
- **The proxy pattern:** Browser → `http://host:3002/api/service` → Node.js proxy → actual API. Credentials only in the proxy.
- **Build backend first, verify data flows, then build frontend.** Never build UI against assumed API responses.
- **One Docker container per service.** Do not put multiple services in one container.
- **All containers must be documented** in `openclaw-project-reference.md` with ports, paths, and purpose.

---

## 9. Debugging Checklist

When something isn't working, go through this in order:

1. Check container logs: `docker logs [container] --tail 30`
2. Test the API directly with curl from Unraid terminal
3. Check the proxy logs: `docker logs openclaw-dashboard --tail 30`
4. Open browser DevTools → Network tab, find the failing request, check status and response
5. Open browser DevTools → Console tab, look for red errors
6. Check file sizes to confirm latest version is deployed: `wc -c [file]`
7. If config related — verify with `jq` before applying any changes

---

## 10. Known Issues & Lessons Learned

- **`JSON.stringify` in onclick attributes breaks** when data contains quotes or special characters. Always use global variables instead.
- **InfluxDB v1 ignores column aliases.** Always access by index.
- **The bot's workspace doesn't persist to host.** GitHub is the only reliable storage.
- **Wrong GitHub token:** The bot account PAT contains `NHG` — if you see `NHH` it's the wrong token.
- **openclaw.json MCP key must be `.mcp.servers`** — never `.mcpServers` at root level.
- **Glances export and web server mode are mutually exclusive.** Use Telegraf for InfluxDB export instead.
- **InfluxDB CORS is disabled by default.** Never enable it — use the proxy instead.

---

## 11. GitHub PR Rules

- **Never approve your own Pull Requests.**
- **Never approve another bot's Pull Requests.**
- **Only submit PRs to `bot-standards` — never merge them yourself.**
- **Only `@mclarkin9681` merges PRs to main** — wait for his review and approval.
- **When you learn something new that should be added to `STANDARDS.md`**, create a new branch, make the change, and submit a PR with a clear description of what was learned and why it's being added.
- **PR titles must be descriptive** — not "update standards" but "Add rule: never use JSON.stringify in onclick attributes".
