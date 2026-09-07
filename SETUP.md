# CM3D Cashier

Mobile order entry for a single cashier, with password authentication and an owner-controlled Google Sheets backend.

## Deployment status

The frontend and backend source are built and unit tested. Live Google Apps Script deployment, credential setup and an end-to-end submission check are required before production use. The login stays disabled until `config.json` contains the deployed API URL. No demo password or mock order data is included.

## What the cashier can do

Enter customer name, phone, address, date, quantity, design and total selling price. After a successful submission, the form is cleared. The order list receives only name, phone and delivery status. There is no detail, export, financial or edit API. The cashier does not need spreadsheet access.

## Google backend setup (owner only)

1. Open the existing spreadsheet, choose Extensions → Apps Script. Paste `backend/Code.gs` into the script editor.
2. In Project Settings enable the manifest view, then use `backend/appsscript.json`. The script uses the Google Sheets REST API for atomic writes. If the API reports disabled, enable Google Sheets API in this script's associated Google Cloud project.
3. Open `owner/password-setup.html` locally. Choose the cashier username and a unique password of at least 16 characters. It derives a PBKDF2-SHA256 key (600,000 iterations), then a SHA256 verifier. No password leaves that page.
4. In Apps Script → Project Settings → Script Properties add `CASHIER_USERNAME`, `PASSWORD_SALT`, and `PASSWORD_VERIFIER` from the local tool. Also add `SPREADSHEET_ID` using the ID between `/d/` and `/edit` in your spreadsheet URL. Do not put these values in GitHub or frontend configuration.
5. Deploy → New deployment → Web app. Execute as **yourself**, access **Anyone**. This makes the API reachable, not the spreadsheet: the script rejects data requests without its own valid cashier session. Approve only the displayed spreadsheet and external request scopes. Keep the project and spreadsheet owner-only.
6. Put the deployed `https://script.google.com/macros/s/.../exec` URL into `public/config.json` (or root `config.json` for the prebuilt site). It is a public endpoint address, not a password.
7. Publish the frontend; sign in with the cashier account. Check one test order appears in Sheet1 and the dashboard. Verify an incognito unauthenticated API call cannot list orders and the authenticated response exposes only name, phone and status. Remove the clearly labeled test order as the owner after checking. Live cross-origin behavior must be verified against the deployed web app.

## Sheet mapping

- A date, B name, C address, D phone stored as text, E quantity, F design, G selling price.
- H starts as `Not Delivered`; the owner changes it in Sheet1. Cashier sees status when refreshing the order list.
- I starts at 0; no payment is assumed. J uses `=G[row]*I[row]`, K uses the existing penalty calculation `=J[row]-INT(J[row]/100)`. The owner controls payment recognition.
- AF stores a submission UUID for safe retries. Setup fails closed if AF already contains unrelated data or if A:K headers change.
- A:K and AF are written together in one atomic batch; unrelated expense, salary and dashboard cells are untouched. Blank prefills in I:K do not push new orders to row 1002.
- This is one order/customer per row, matching the existing workbook.

## GitHub Pages

The prebuilt `index.html`, `config.json`, `manifest.webmanifest`, `favicon.svg` and `.nojekyll` can be hosted directly from main → root in repository Settings → Pages. `source.zip` contains the editable source and owner setup files, without credentials or customer data.

For a normal source checkout, run `npm ci`, `npm test`, `npm run build`. `pages-dist` is the publish directory. The included GitHub Actions workflow builds and deploys it when Pages uses GitHub Actions. GitHub Pages hosts only the interface; Apps Script runs the private backend.

## Security and operational notes

Session tokens are random, hashed at rest, kept only in browser memory, expire in 8 hours and are invalidated by logout, password rotation or another login. Five failed attempts impose a five-minute account lockout. This account-wide limit can temporarily block legitimate access during an attack; Apps Script quotas are not suitable for a large public service. Use one cashier account and a strong unique password.

The derived login key is a password-equivalent secret and travels only in an HTTPS POST body. The script stores only its hash. No password, token or customer records are written to localStorage, URLs, logs or the public repository. There is no offline queue or service-worker cache. Previously viewed information or details a cashier originally typed cannot be erased from their memory or screenshots.

A lost network response offers Retry with the same UUID; do not reload until the result is resolved. Reloading loses the in-memory pending submission. A duplicate UUID returns success without adding another order. Keep the sheet owner-only and preserve AF IDs.

Built-in tests use in-memory substitutes, not your real spreadsheet. They test authentication, lockouts, field whitelists, session invalidation, input validation, formula-injection prevention, and atomic/idempotent writes. They do not replace the live deployment check.
