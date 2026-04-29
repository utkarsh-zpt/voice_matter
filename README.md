# Zepto Rider Feedback — Static + API Architecture

This is the **decoupled** version of the feedback form:
- `index.html` is a fully self-contained static page hostable anywhere
- `Code.gs` is the Apps Script API backend that writes to Google Sheets
- The form calls the API via fetch() — no Apps Script templating

## Why this architecture?

- The HTML lives on a public host (Vercel, Netlify, or your own infra) — no Google "unverified app" warning, no Workspace access restrictions
- The Sheet write happens server-side via the API — Sheet permissions stay locked down
- Easy to swap the backend later (Cloud Run, your own backend) without touching the HTML

## Deployment options for the backend

### Option A: Apps Script as the API (simplest)

Apps Script can serve a JSON API. Even when your Workspace blocks "Anyone" for web apps, programmatic API endpoints sometimes have looser policies — worth re-asking IT framed as "backend API endpoint" not "public web page."

1. Open your Google Sheet → Extensions → Apps Script
2. Paste `Code.gs`, replace `SHEET_ID` with your Sheet ID
3. Run `setupSheet()` once to create headers
4. Deploy → Web App → Execute as: Me → Who has access: Anyone
5. Copy the deployed URL — paste into `API_ENDPOINT` in `index.html`

### Option B: Cloud Run (preferred for production)

A tiny Node.js or Python service that uses a service account to write to the Sheet via the Google Sheets API. No "Anyone" warning concerns since it's not Apps Script.

```js
// Example Cloud Run handler
import { google } from 'googleapis';

const sheets = google.sheets({ version: 'v4', auth: serviceAccountAuth });

app.post('/feedback', async (req, res) => {
  await sheets.spreadsheets.values.append({
    spreadsheetId: SHEET_ID,
    range: 'Star Rating Feedback!A:M',
    valueInputOption: 'USER_ENTERED',
    requestBody: { values: [[new Date(), req.body.riderId, ...]] }
  });
  res.json({ success: true });
});
```

### Option C: Use Zepto's existing backend

Add one endpoint to your existing internal API that handles the same JSON payload. Probably the most "production-correct" answer if Zepto has a Node/Python service already.

## Hosting the HTML

### Vercel (recommended for prototypes)
1. Drag `index.html` into vercel.com/new
2. Get a URL like `zepto-rider-feedback.vercel.app`
3. Optionally connect a custom domain (`feedback.zepto.com`)

### Netlify
Same idea — drag-and-drop deploy at app.netlify.com/drop

### GitHub Pages
1. Create a public repo, add `index.html`
2. Settings → Pages → Deploy from branch
3. Get a URL like `username.github.io/zepto-rider-feedback`

### Zepto's own infra
If Zepto hosts internal tools at `tools.zepto.com` or similar, just drop the HTML there. Most likely the right answer for compliance.

## The deeplink URL

Once everything is deployed, share this URL pattern with riders:

```
https://feedback.zepto.com?deeplinkType=IN_APP&riderId=R1234
```

Note the **`&`** separator (not `?`) between params.

The HTML reads both params client-side; only `riderId` is required, `deeplinkType` is informational and stored in the sheet for analytics.

## API contract

### Eligibility check
```
GET {API_ENDPOINT}?action=eligibility&riderId=R1234

→ { eligible: true }
  OR
  { eligible: false, reason: "already_submitted", nextAvailable: "07 May 2026" }
```

### Submit feedback
```
POST {API_ENDPOINT}
Content-Type: text/plain;charset=utf-8

{
  "riderId": "R1234",
  "deeplinkType": "IN_APP",
  "rating": 4,
  "tags": ["payouts", "zepto_support"],
  "comments": "Loved the support team!"
}

→ { success: true, message: "..." }
  OR
  { success: false, alreadySubmitted: true, nextAvailable: "07 May 2026" }
```

## Why `text/plain` for POST?

Browsers do a CORS preflight (OPTIONS request) for `application/json` POSTs to a different origin. Apps Script doesn't handle OPTIONS well, so we send as `text/plain` to skip the preflight. The body is still JSON; the backend parses it explicitly.

If you switch to Cloud Run / your own backend, you can use `application/json` with proper CORS headers and skip this workaround.
