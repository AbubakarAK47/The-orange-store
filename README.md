# Orange (crypto checkout for a physical shop)

A small FastAPI + SQLite store with a mobile-first checkout: product page, checkout form, live waiting-for-payment page, success page, emailed receipt, and a password-protected admin page.

## How it works
1. Customer picks a product and enters name + email.
2. The server creates an order (id like HC-1A2B3C4D) and computes the amount to pay.
   - Self-hosted mode (default, no signup, no key): the customer sends a unique amount (e.g. 25.417 USDT) to YOUR wallet. A watcher checks public explorer APIs (TronScan for USDT TRC20, Blockstream for BTC) every 45s and matches the exact amount + time window to the order.
   - Hosted mode (optional): if NOWPAYMENTS_API_KEY is set, an invoice is created on NOWPayments instead and the status is polled and also accepted by webhook.
3. The payment page polls order status every few seconds and redirects to the success page once confirmed.
4. A receipt email (HTML, order id, USD and coin amount, tx hash) is queued and sent by SMTP. Failed sends retry with exponential backoff (1, 2, 4 ... minutes, up to 8 tries). Nothing is lost while SMTP is not yet configured: receipts stay queued.
5. /admin lists all orders and payment status.

## Setup
1. Install: pip install -r requirements.txt
2. Copy .env.example to .env and fill in:
   - ADMIN_PASSWORD: pick a strong one. Admin login is disabled while this is empty.
   - USDT_TRC20_ADDRESS: your USDT (TRON network, TRC20) receiving address, starts with T. Recommended: a wallet you control (e.g. Trust Wallet, Binance deposit address on TRC20 network).
   - BTC_ADDRESS (optional) and ENABLED_COINS=USDT or USDT,BTC.
   - SMTP_HOST, SMTP_PORT, SMTP_USER, SMTP_PASS, SMTP_FROM: any SMTP provider. Gmail: smtp.gmail.com, port 587, an App Password (not your normal password). Brevo, Mailgun and Zoho also have free SMTP tiers.
   - PUBLIC_BASE_URL: your live site URL (e.g. https://hype-checkout.onrender.com).
   - BUSINESS_NAME: your shop name.
   - NOWPAYMENTS_API_KEY (optional): only if you want hosted invoices instead of self-hosted watching.
3. Run locally: python run.py  then open http://localhost:8000

## Products
Edit PRODUCTS in app/config.py (name, price_usd, blurb, emoji).

## Deploy free (Render or Railway)
Render:
1. Push this folder to a GitHub repo (keep .env out; .gitignore already does).
2. On render.com: New > Web Service > connect the repo. Build: pip install -r requirements.txt. Start: uvicorn app.main:app --host 0.0.0.0 --port $PORT
3. Add the environment variables from .env in the Environment tab.
4. Free-tier caveat: the free instance SLEEPS after about 15 minutes with no traffic. It wakes on the next request, but the background watcher only runs while the instance is awake, so payments made while it sleeps are only matched once it wakes up (the watcher runs immediately on wake). A paid instance (about 7 USD/month) avoids this. The SQLite file is also wiped on redeploy on the free tier; for production add a persistent disk or move to Postgres.
Railway:
1. railway.app > New Project > Deploy from GitHub repo. Railway reads the Procfile.
2. Add the variables in Variables. Generate a public domain under Settings > Networking.
3. Railway's free trial credit is limited; check current pricing before relying on it.

## Notes and limits
- Self-hosted matching uses the exact unique amount. Do not tell customers to round the amount. Orders expire after ORDER_EXPIRY_MINUTES (default 60).
- USDT must be sent on the TRON (TRC20) network. Sending on another network will not be detected and funds may be lost.
- Verify a few test payments of small value before selling for real.
- Taxes, refunds and shipping are not built in. Refund manually from your wallet and tell the customer.
- Keep ADMIN_PASSWORD and SMTP_PASS secret. Never commit .env.
