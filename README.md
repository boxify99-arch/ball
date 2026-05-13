# DuckyCasino — Telegram Bot

Crypto Telegram casino bot (dice / bowling / soccer / darts / basketball / mines / blackjack / etc.)
backed by OxaPay for deposits & withdrawals.

---

## Deploy to Railway (recommended)

Railway gives you $5/month of free credit, which is enough to run this bot 24/7.
Unlike PythonAnywhere's free tier, Railway has **no outbound IP whitelist**, so OxaPay
API calls work out of the box, and your process stays alive permanently.

### 1. Prep your repo

Put these files together at the root of a new GitHub repo:

```
ducky.py
requirements.txt
Procfile
runtime.txt
.gitignore
README.md
```

> ⚠️ **Never commit your real tokens.** They are read from environment variables only —
> the hardcoded defaults that used to live in `ducky.py` have been removed.

Push to GitHub (private repo recommended).

### 2. Create the Railway project

1. Go to <https://railway.app> → **New Project** → **Deploy from GitHub repo**.
2. Pick your repo. Railway auto-detects Python and installs `requirements.txt`.
3. When the first build finishes it will likely **crash** — that's expected, you haven't set the env vars yet.

### 3. Add environment variables

In the Railway service → **Variables** tab, add:

| Variable                  | Value                                                |
| ------------------------- | ---------------------------------------------------- |
| `TELEGRAM_TOKEN`          | Your *new* BotFather token (rotate the leaked one!)  |
| `OXAPAY_MERCHANT_API_KEY` | Your new OxaPay merchant key                         |
| `OXAPAY_PAYOUT_API_KEY`   | Your new OxaPay payout key                           |
| `OXAPAY_GENERAL_API_KEY`  | Your new OxaPay general key                          |
| `PRIVATE_LOG_GROUP_ID`    | (optional) Telegram chat ID for deposit/withdraw log |
| `SLOTS_WEBAPP_URL`        | (optional) Override for the slots mini-app URL       |

Click **Deploy** (or push a new commit). The bot will boot and start polling.

### 4. Add a persistent volume (so balances survive restarts)

By default `balances.json` is written next to the script and is **wiped on every redeploy**.
To keep balances permanent:

1. Service → **Settings** → **Volumes** → **+ New Volume**
2. Mount path: `/data`
3. Redeploy.

The hardened `ducky.py` automatically detects `/data` and writes `balances.json` there.

### 5. Verify it's running

- Service → **Deployments** → **View Logs** — you should see
  `Application started` and no tracebacks.
- Message your bot on Telegram with `/start`. It should reply.

---

## What was hardened in `ducky.py`

The original file had a few issues that caused "won't respond / sometimes doesn't respond":

1. **Hardcoded API keys** → now required from env vars (`os.environ[...]`). Boot fails loudly if missing instead of using leaked keys.
2. **`run_polling()` had no restart-on-crash** → wrapped in a `while True` loop that logs the exception and retries every 5 seconds, so a transient network blip doesn't kill the bot for good.
3. **`drop_pending_updates=True`** added → prevents the 409 Conflict that happens when you redeploy while another instance is still draining old updates (a very common cause of "bot stops responding after redeploy").
4. **`balances.json` path is configurable** → defaults to `/data/balances.json` when a Railway volume is mounted, so balances persist across deploys.

No game logic was changed.

---

## Updating the bot

```bash
git add .
git commit -m "tweak something"
git push
```

Railway auto-redeploys on push. Your env vars and volume persist.

---

## Common issues

| Symptom                                          | Fix                                                                 |
| ------------------------------------------------ | ------------------------------------------------------------------- |
| Boot crash: `KeyError: 'TELEGRAM_TOKEN'`         | You forgot to set the variable in Railway → Variables.              |
| Bot replies twice to every message               | You have two deployments running. Stop the local one, or redeploy.  |
| `telegram.error.Conflict: terminated by other getUpdates` | Same as above — only one process can poll a bot token at a time.    |
| Balances reset after deploy                      | You didn't add the `/data` volume (step 4).                         |
| OxaPay calls time out                            | Check the OxaPay key is correct; check Railway logs for the URL.    |

---

## Why not PythonAnywhere free tier?

- Free tier blocks outbound HTTP except to a small whitelist → `api.oxapay.com` is **not** on it, so deposits/withdrawals always fail.
- Free tier doesn't support always-on tasks → polling loop dies when your console disconnects.

If you want to stay on PythonAnywhere, the **Hacker plan ($5/mo)** removes both limits and your bot will work there too — just run `python ducky.py` as an Always-on Task and set the same env vars.
