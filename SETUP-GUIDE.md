# FreeLLMAPI — Setup Guide

Repo: <https://github.com/tashfeenahmed/freellmapi>
Set up on: 2026-11-18 · Node 22 · local-dev mode (Node, no Docker)

FreeLLMAPI is a self-hosted router that stacks the free tiers of many LLM providers
behind one OpenAI-compatible (and Anthropic / Gemini / Ollama-compatible) `/v1` endpoint.

---

## 1. What was done (cloud session)

| Step | Result |
|---|---|
| Clone repo | ✅ |
| `npm ci` | ✅ installed (native `better-sqlite3` builds and loads) |
| `.env` created | ✅ random 64-hex `ENCRYPTION_KEY`, `PORT=3001`, file mode 600 |
| `npm run dev` | ✅ API on `:3001`, Vite dashboard on `:5173` |
| Admin account | ✅ created via `POST /api/auth/setup` (placeholder email `admin@example.com`) |
| Unified API key | ✅ retrieved from `GET /api/settings/api-key` |
| `GET /v1/models` (with key) | ✅ 82 models listed (`auto`, `fusion`, `kimi-k2.6`, `qwen3-coder`, …) |
| `GET /v1/models` (no key) | ✅ `401`, so auth is enforced |
| `POST /v1/chat/completions` | ⚠️ `503 no_providers_configured`, expected until provider keys are added |
| Dashboard `GET :5173/` | ✅ `200` |

### Test results

| Suite | Result |
|---|---|
| bootstrap / hooks | 2/2, 9/9 pass |
| server | **5575 pass, 1 fail**, 6 skipped (424 files) |
| cli | 182 pass, 1 skipped |
| client + i18n | 503 pass; i18n OK (60 locales) |

The one server failure (`proxy.test.ts › sends a *.localhost subdomain direct`) comes from
the cloud sandbox. The test needs `ollama.localhost` to resolve, and this container's DNS
doesn't resolve `*.localhost` names. On a normal macOS or Linux machine it should pass.

> ⚠️ The cloud container is temporary. The running server, database, and keys are
> deleted when the session ends. To run FreeLLMAPI for real, use section 2 or 3 on your own machine.

---

## 2. Run it on your own machine: Node (local dev)

Prereqs: **Node.js 20.18–24** (repo pins 22 in `.nvmrc`), npm ≥ 10, git.

```bash
git clone https://github.com/tashfeenahmed/freellmapi.git
```

```bash
cd freellmapi
```

```bash
npm install
```

Create `.env` with a fresh encryption key (keep this key: without it, your stored provider keys can't be decrypted):

```bash
printf "ENCRYPTION_KEY=%s\nPORT=3001\n" "$(node -e 'console.log(require("crypto").randomBytes(32).toString("hex"))')" > .env
```

```bash
npm run dev
```

- Dashboard: <http://localhost:5173>
- API: <http://localhost:3001/v1>

On first open, the dashboard asks you to create the admin account. No setup code is
needed from the same machine. From another device, use the one-time code printed in the server log.

Optional: `npm run dev:lan` exposes the dashboard to other devices on your LAN.

## 3. Alternative: Docker (best for long-running use)

One-liner (sets up `~/freellmapi`, generates the key, and pulls and starts the container):

```bash
curl -fsSL https://freellmapi.co/install.sh | bash
```

Then open <http://localhost:3001>. Full Docker Compose details are in `docs/en/install/01-install.md`.

---

## 4. After it's running

1. **Add provider keys.** Go to the **Keys** page and paste free-tier keys. Good ones to start with:
   Google AI Studio (Gemini), Groq, Cerebras, OpenRouter, Mistral, NVIDIA, Cloudflare, Cohere.
   Each key is encrypted at rest with your `ENCRYPTION_KEY`.
2. **Set the fallback order.** The **Fallback Chain** page sets which models are tried first.
3. **Copy your unified API key.** It's in the header of the **Keys** page. Clients use this one key.
4. **Test it:**

```bash
curl http://localhost:3001/v1/chat/completions -H "Authorization: Bearer <UNIFIED_KEY>" -H "Content-Type: application/json" -d '{"model":"auto","messages":[{"role":"user","content":"Say hi"}]}'
```

### Use it from code (OpenAI SDK)

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:3001/v1", api_key="<UNIFIED_KEY>")
print(client.chat.completions.create(model="auto",
      messages=[{"role": "user", "content": "Hello"}]).choices[0].message.content)
```

### Hook up a coding agent

```bash
npx freellmapi setup-claude --url http://localhost:3001 --api-key <UNIFIED_KEY>
```

Setup commands for other agents (Codex, Gemini CLI, Cursor, Cline, Aider, …) are in
`docs/en/clients/02-supported-agents.md`.

---

## 5. Useful commands

| Command | Purpose |
|---|---|
| `npm run dev` | API + dashboard (dev) |
| `npm test` | Full test suite |
| `npm run build` | Production build (server, cli, client) |
| `npm run db:migration:status` | DB migration state |
| `cd server && npm run rotate-encryption-key -- --new-key <hex>` | Rotate the encryption key (stop the server first) |

Data lives in `server/data/freeapi.db` (SQLite). Back up this file together with `.env`.

## 6. Notes / gotchas

- The server logs a warning that `REQUEST_ANALYTICS_*` is missing from `.env`. It's harmless, and the defaults apply (90 days / 100k rows).
- `npm ci` reports 38 audit advisories in dev dependencies. This is normal for the repo. Don't run `npm audit fix --force`, because it can break pinned versions.
- Keep the dashboard on localhost unless you're on a trusted network. It's single-user, and the unified key is the only thing guarding the API.
