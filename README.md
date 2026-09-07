# Klaviyo Audit Copilot

AI-assisted lifecycle email auditor for Ditto Daily.

Sync Klaviyo flows, templates, triggers, delays, subjects, and email copy into a local JSON dump — then chat with Gemini over that full context to audit links, recommend placements, and draft brand-aligned copy.

**Live app:** [https://klaviyo-audit-ditto.streamlit.app](https://klaviyo-audit-ditto.streamlit.app)  
**Repo:** [Ditto-Daily/klaviyo-audit](https://github.com/Ditto-Daily/klaviyo-audit)

---

## Why this exists

Auditing lifecycle flows in Klaviyo by hand is slow: hunting manage-subscription / portal links, finding where a VIP email should sit, matching voice across adjacent emails. This app pulls the account once, caches it, and lets a Gemini copilot answer strategy questions against the real program — not against memory.

---

## How it works

```
┌─────────────────┐     read-only GET      ┌──────────────────┐
│  Klaviyo API v3 │ ─────────────────────► │ klaviyo_sync.py  │
└─────────────────┘                        └────────┬─────────┘
                                                    │ writes
                                                    ▼
                                           ┌──────────────────┐
                                           │ klaviyo_dump.json│
                                           │  (committed cache)│
                                           └────────┬─────────┘
                                                    │ loaded as context
                                                    ▼
┌─────────────────┐     generate_content     ┌──────────────────┐
│  Gemini 3.5 Flash│ ◄───────────────────── │ gemini_copilot.py│
└─────────────────┘                          └────────┬─────────┘
                                                      │
                                                      ▼
                                             ┌──────────────────┐
                                             │     app.py       │
                                             │  Streamlit chat  │
                                             └──────────────────┘
```

1. **Sync** (`klaviyo_sync.py`) — read-only Klaviyo REST calls. Fetches flows (+ definitions), templates, email subjects / preview / body text / links. Writes `klaviyo_dump.json`.
2. **Cache** — one snapshot file. Chat always reads from it. Sync only overwrites when you click **Sync Klaviyo Data**. The dump is committed to git so Streamlit Cloud boots with data even after sleep/redeploy.
3. **Copilot** (`gemini_copilot.py`) — packs a compact projection of the dump into the model system context and streams answers with chat history.
4. **UI** (`app.py`) — sidebar status + sync + metrics; main chat with starter prompts.

**Important:** Klaviyo access is **GET-only**. Nothing is created, updated, paused, or deleted in Klaviyo.

---

## Repo layout

| File | Role |
|------|------|
| `app.py` | Streamlit UI |
| `klaviyo_sync.py` | Klaviyo → JSON pipeline |
| `gemini_copilot.py` | Gemini client + context packing |
| `config.py` | Secrets: `st.secrets` → env / `.env` |
| `chat_store.py` | Chat persistence (disk + browser localStorage) |
| `klaviyo_dump.json` | Latest synced snapshot (committed) |
| `requirements.txt` | Dependencies |
| `.streamlit/config.toml` | Theme / server defaults |
| `.env` | Local secrets only (gitignored) |

---

## Local setup

```bash
git clone https://github.com/Ditto-Daily/klaviyo-audit.git
cd klaviyo-audit
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Create `.env`:

```bash
KLAVIYO_API_KEY=pk_...
GEMINI_API_KEY=...
GEMINI_MODEL=gemini-3.5-flash
```

Klaviyo private key scopes (read only): `flows:read`, `templates:read`.

Run:

```bash
# optional refresh
python klaviyo_sync.py

streamlit run app.py
```

Open [http://localhost:8501](http://localhost:8501).

---

## Streamlit Cloud deploy

1. [share.streamlit.io](https://share.streamlit.io) → **New app**
2. Repo: `Ditto-Daily/klaviyo-audit` · Branch: `main` · File: `app.py`
3. **Advanced settings → Secrets:**

```toml
KLAVIYO_API_KEY = "pk_..."
GEMINI_API_KEY = "..."
GEMINI_MODEL = "gemini-3.5-flash"
```

4. Deploy → app URL (currently [klaviyo-audit-ditto.streamlit.app](https://klaviyo-audit-ditto.streamlit.app))

If the org repo is private, grant Streamlit access to the **Ditto-Daily** GitHub org.

### Persistence notes (Streamlit Cloud)

| Data | How it survives |
|------|-----------------|
| Klaviyo dump | Committed `klaviyo_dump.json` + Sync overwrites at runtime |
| Chat | Browser localStorage + `chat_history.json` (disk; may clear if the container is fully replaced) |
| Secrets | Streamlit Cloud secrets (not in git) |

Cloud can sleep/restart the server without you refreshing. That used to wipe an uncommitted dump — committing the dump fixed the “library went to zero” issue.

---

## Using the copilot

Examples that work well:

- “Which emails link to manage subscription / Recharge / customer portal?”
- “Where should a VIP/loyalty email sit in the live journeys?”
- “Draft a follow-up for Abandoned Cart matching adjacent voice.”
- “Summarize live vs draft lifecycle coverage.”

Gemini sees flow names, statuses, triggers, delays, subjects, preview text, body text, and links — not raw HTML (stripped to keep the dump small and token-efficient).

---

## Config / secrets

| Key | Required | Default |
|-----|----------|---------|
| `KLAVIYO_API_KEY` | for Sync | — |
| `GEMINI_API_KEY` | for chat | — |
| `GEMINI_MODEL` | no | `gemini-3.5-flash` |

Fallbacks if the primary model 404s: `gemini-3.6-flash`, `gemini-3-flash-preview`.

Never commit `.env` or `.streamlit/secrets.toml`.

---

## For future developers

### Mental model

- **Source of truth for chat** = `klaviyo_dump.json`, not live Klaviyo.
- **Sync** = expensive, rate-limited, read-only refresh.
- **Copilot** = dump + history → Gemini stream.

### Extending safely

- Prefer new read endpoints / richer parsing in `klaviyo_sync.py`.
- Keep write scopes off the Klaviyo key unless you intentionally add write features (do not by default).
- After a meaningful Sync locally, commit the updated `klaviyo_dump.json` so Cloud users get the new snapshot without waiting for a cloud Sync.
- Keep dump compact (body text + links). Re-adding full HTML will blow tokens and repo size.

### Good first improvements

See [Future expansion](#future-expansion) below.

### Local remotes

```text
ditto  → https://github.com/Ditto-Daily/klaviyo-audit.git   (primary)
origin → https://github.com/Aditii2112/klaviyo-audit.git    (older personal copy)
```

Prefer pushing to `ditto` / the org repo.

---

## Future expansion

Rough roadmap ideas (not committed work):

| Area | Idea |
|------|------|
| **Coverage** | Campaigns, segments, forms, SMS copy depth |
| **Audits** | Saved audit playbooks (portal links, discount hygiene, missing unsubscribe, tone drift) |
| **Diffs** | Compare two dump timestamps / highlight what changed after Sync |
| **Exports** | One-click PDF / Notion / Slack summary of an audit |
| **Auth** | Streamlit auth / allowlist so the app isn’t open to the world |
| **Storage** | Durable chat + dump in S3 / GCS / Supabase if Cloud sleep still bites |
| **Actions** | Optional *draft-only* Klaviyo writebacks (explicit, separate write key) |
| **Eval** | Golden questions + expected citations for regression on prompt/model changes |
| **Multi-brand** | Switch dumps per brand / account |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Library all zeros | Empty dump / Cloud boot before dump was committed | Sync once, or pull latest main with `klaviyo_dump.json` |
| `Gemini request failed for all candidate models` | Retired model name in secrets | Set `GEMINI_MODEL = "gemini-3.5-flash"` |
| Truncated replies | Old token cap | Already raised + streaming in current main |
| Chat “forgot” prior turns | Cloud restart wiped session_state | localStorage restore should reload; avoid Clear chat |
| Sync proxy / DNS errors | Local sandbox / network | Retry Sync; chat still works off existing dump |

---

## License / ownership

Internal tool for **Ditto Daily**. Treat API keys and the dump as sensitive — the dump contains lifecycle copy and URLs.
