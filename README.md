# Brand Assistant

A branded AI chat assistant for your website. The page runs free on **GitHub Pages**. A small **Cloudflare Worker** keeps your AI key secret.

```
Visitor's browser  ->  Cloudflare Worker  ->  AI service
(site/ on GitHub)      (worker/, holds key)   (Anthropic API)
```

**Why a Worker?** Anything on GitHub Pages is public. If the AI key were in the page, anyone could steal it. The Worker holds the key and the page never sees it.

## What it does

- Streams replies word by word, with Stop, Copy, Regenerate and Try again.
- Keeps the chat in the visitor's own browser, with a Clear chat button.
- Works on phones, in light and dark mode, with large touch targets.
- Shows a clear "Setup needed" note until it is connected. It never fakes a reply.
- Shows replies as plain text (safe), with simple bold, bullet and numbered lists.
- Blocks other websites from using your Worker, and caps message size and reply length.

## Before you start: costs

| Part | Cost |
| --- | --- |
| GitHub Pages | Free for public repositories. Private repositories need a paid GitHub plan. |
| Cloudflare Workers | Has a free plan with daily limits. Check Cloudflare's current limits. |
| AI service (Anthropic API) | **Paid, by use.** It is not free. Set a monthly spend limit in the Anthropic Console. |

## Setup (about 20 minutes)

Everything below can be done in a web browser, also on a phone.

### Step 1. Get an AI key

1. Make an account in the Anthropic Console.
2. Add billing, and set a **monthly spend limit**.
3. Create an API key and keep it private. Do not paste it into chat, email, or any file in the repository.

### Step 2. Put the files on GitHub

1. Create a **new public repository**. Use the default branch name `main`.
2. Upload all the files and keep the folders: `site`, `worker`, `scripts`, `tests`, `.github`.
   - On a computer: **Add file > Upload files**, then drag the whole folder in. Make sure `.github` is included. Some systems hide folders that start with a dot.
   - On a phone: **Add file > Create new file**. Type the full name, for example `site/index.html`, paste the contents, and commit. Typing `/` makes a folder. Repeat for each file.

### Step 3. Create the proxy on Cloudflare

1. Make a free Cloudflare account.
2. Go to **Workers & Pages**, create a new Worker, and deploy the starter code.
3. Open **Edit code**. Delete everything. Paste the whole of `worker/worker.js`. Deploy.
4. Open the Worker's **Settings > Variables and Secrets** and add:

   | Name | Type | Value |
   | --- | --- | --- |
   | `ANTHROPIC_API_KEY` | Secret | your key from step 1 |
   | `ALLOWED_ORIGINS` | Text | `https://YOUR-USERNAME.github.io` |

   `ALLOWED_ORIGINS` is the website address only. No path, no `/` at the end. If you use your own domain, use that instead. Separate several with commas.
5. Copy your Worker's address. It looks like `https://brand-assistant-proxy.something.workers.dev`.

Menu names in Cloudflare change from time to time. Look for the same ideas if a label is different.

### Step 4. Turn on GitHub Pages

In your repository: **Settings > Pages > Build and deployment > Source: GitHub Actions**.

### Step 5. Connect the page to the proxy

1. In your repository: **Settings > Secrets and variables > Actions > Variables > New repository variable**.
2. Name: `WORKER_URL`. Value: the Worker address from step 3. It must start with `https://`.
3. Open the **Actions** tab, choose **Deploy website**, and press **Run workflow**.
4. When it turns green, open `https://YOUR-USERNAME.github.io/YOUR-REPOSITORY/`.

The deploy runs the tests first. If a test fails, nothing is published.

### Step 6. Make it yours

- **Name, greeting, quick questions:** edit `site/config.js`.
- **Brand facts:** this is the most important part. The assistant only knows what you tell it. Add a variable named `SYSTEM_PROMPT` to the Worker (Text). Example:

  ```
  You are the friendly website assistant for Example Brand.
  Use simple English. Be warm and brief.

  About Example Brand:
  - We sell: ...
  - We are in: ...
  - Opening hours: ...
  - Contact: ...

  If you do not know something, say so and suggest contacting the team.
  Never ask for passwords or card numbers.
  ```
- **Colours:** change the variables at the top of `site/styles.css`.
- **Model and reply length:** optional Worker variables `MODEL` and `MAX_TOKENS`.

## Security notes (please read)

- The key lives only in the Worker as a secret. Never put it in `site/` or `config.js`.
- `ALLOWED_ORIGINS` stops other **websites** from using your Worker in a browser. It does **not** stop someone who sends requests from a script and fakes the header. So also:
  - keep the **spend limit** on your AI account, and
  - consider a rate-limiting rule in Cloudflare (check what your plan allows).
- The page has a Content Security Policy that only allows its own scripts.
- Reply text is never inserted as HTML, so a reply cannot run code on the page.
- Visitors' messages go to Cloudflare and to the AI provider. Tell your visitors. The page shows a short note under the message box. Edit `privacyNote` in `config.js` to match your policy.
- Set `saveChat: false` in `config.js` if you do not want chats kept in the visitor's browser.

## Test it

You need Node.js 20 or newer. There is nothing to install for the main tests.

```
npm test
```

This runs 28 checks: 23 for the Worker (allowed origins, message rules, safe errors, no key leaks) and 5 for the deploy script.

**Optional browser test.** It starts the page and the real Worker code on your computer, replaces only the AI service with a fake one, and clicks through the page in Chromium:

```
npm install --save-dev playwright
npx playwright install chromium
npm run test:browser
```

## Run it on your computer (optional)

```
cp worker/.dev.vars.example worker/.dev.vars   # then put your key in it
npm run worker:dev                              # starts the Worker on port 8787
npm run serve                                   # in a second window: page on port 8080
```

Then open `http://localhost:8080` and, in `site/config.js`, temporarily set `apiUrl: "http://localhost:8787"`. Do not commit that change.

## Troubleshooting

| What you see | What to check |
| --- | --- |
| "Setup needed" on the page | The `WORKER_URL` variable is missing, or you did not run **Deploy website** again after adding it. |
| "We could not reach the assistant" | `ALLOWED_ORIGINS` must match your site address exactly (`https`, no path, no `/` at the end). Also check the Worker address. The browser console shows a CORS message if this is the cause. |
| "The assistant is not set up yet" | The Worker has no `ANTHROPIC_API_KEY` secret. |
| "The assistant is not set up correctly yet" | The key is wrong, or the account has no billing. Open the Worker's logs for details. |
| "The assistant is busy right now" | A rate limit or heavy load. Wait a moment and try again. |
| Old version of the page | Check the **Actions** tab for a green run, then refresh the page without cache. |
| Deploy fails with "Get Pages site failed" | Step 4 is not done. |

## Files

| File | Purpose |
| --- | --- |
| `site/index.html`, `styles.css`, `app.js` | The chat page. No frameworks, no outside scripts. |
| `site/config.js` | Your brand settings. Public. No secrets. |
| `worker/worker.js` | The secure proxy. One file, no packages. |
| `worker/wrangler.toml` | Optional settings for the Wrangler tool. |
| `scripts/inject-config.js` | Puts `WORKER_URL` into `config.js` when deploying. |
| `.github/workflows/` | Deploy website (with tests), Tests, and an optional Worker deploy. |
| `tests/browser-smoke.mjs` | Optional real-browser test. |

## Using a different AI provider

Only `worker/worker.js` talks to the AI. To change provider, edit the address, headers and request body there. The page reads Anthropic-style streaming events (`content_block_delta` with `text_delta`). If your provider streams another format, change `handleSseEvent` in `site/app.js`.

If you deploy the Worker with Wrangler or the optional workflow, the values in `worker/wrangler.toml` replace the ones set in the dashboard. Keep them in one place.
