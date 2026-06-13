# FinalNot_mat2_26 — Deployment & Debugging Guide

This guide collects everything we discussed and the concrete steps to get the site working, debug problems you saw in the console, and maintain safe tooling and workflow permissions. Put this file under `.guide_and_manuals/` in the repository so you (and collaborators) have a single canonical reference.

---

## Purpose

This document helps with:
- GitHub Actions / workflow permission troubleshooting
- Putting the Supabase anon key into the site securely (via GitHub Secrets + workflow)
- Common index.html fixes and runtime debugging ("Unexpected token '<'" etc.)
- Local testing and deployment via GitHub Pages
- Removing or editing workflows safely

---

## Quick checklist (most common fixes)

1. Make sure GitHub Actions Workflow permissions allow the workflow to create files if you rely on the workflow to generate `config.js`:
   - Repository → Settings → Actions → General → Workflow permissions → choose **Read and write permissions**.
2. Add `SUPABASE_KEY` as a repository secret (Settings → Secrets and variables → Actions → New repository secret). Use the anon/public key from Supabase (NOT the service_role key).
3. Ensure workflows (the deploy workflow) have `permissions: contents: write` so they can commit/publish files when needed.
4. Do NOT commit service_role keys or other sensitive credentials into the repo. If you added `config.js` containing the anon key for testing, remove it and rely on secrets + workflow for production.
5. If you see `Uncaught SyntaxError: Unexpected token '<'`, usually you are fetching a JS file but getting HTML (often a 404 HTML page). Check Network → config.js and open the Response.

---

## Detailed Steps

### 1) Set Workflow Permissions

1. Open: `https://github.com/<owner>/<repo>/settings/actions` (or Repository → Settings → Actions).
2. Scroll to **Workflow permissions**.
3. Select: **Read and write permissions**.
4. Click **Save**.

Why: If your workflow needs to create `config.js` at deploy time or push to `gh-pages`, it needs write permission to contents.

Note: This is different than the top-level Actions permissions (Allow all actions / Allow local org actions / etc.). Those control which actions can run, not the workflow token permissions.

---

### 2) Add the SUPABASE_KEY secret

1. Go to: `Settings → Secrets and variables → Actions`.
2. Click **New repository secret**.
   - Name: `SUPABASE_KEY`
   - Value: your Supabase anon/public key (from Supabase Dashboard → Settings → API → Project API keys). Do NOT paste service_role key.
3. Click **Add secret**.

If the UI shows an empty value after you open the secret again, that is expected behavior: GitHub hides secret values once stored. If the secret truly isn't saved (you didn't get a success confirmation or Actions still cannot read it), re-add it and ensure you have write/admin permission in the repo.

---

### 3) Workflow example: create config.js and deploy

If you want the workflow to generate a `config.js` during deploy (recommended so you don't commit keys), here's a recommended workflow snippet. Put this under `.github/workflows/deploy.yml`.

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v3

      - name: Create config.js
        env:
          SUPABASE_KEY: ${{ secrets.SUPABASE_KEY }}
        run: |
          cat > config.js << 'EOF'
          window.SUPABASE_URL = 'https://<PROJECT>.supabase.co';
          window.SUPABASE_KEY = '$SUPABASE_KEY';
          EOF

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./
```

Notes:
- The workflow uses `permissions.contents: write` to allow pushing files.
- The `SUPABASE_KEY` is injected from repository secrets.
- Make sure branch used by the workflow is the branch that GitHub Pages is serving (or your action publishes to gh-pages branch).

---

### 4) index.html recommended fixes and checks

1. Replace HTML comments inside `<script>` blocks with JS comments. Example:
   - Bad (inside <script>): `<!-- comment -->`
   - Good: `// comment` or `/* comment */`
2. Don't keep the fallback string `<ANON_PUBLIC_KEY>` in production. It's fine as a development placeholder, but always check for it and show a helpful error when missing.
3. Use a robust safety check before creating the Supabase client:

```javascript
if (typeof supabase === 'undefined' || typeof supabase.createClient !== 'function') {
  document.getElementById('sonuc').innerHTML = '<div class="error">Supabase SDK yüklenemedi. CDN engellenmiş olabilir.</div>';
  throw new Error('Supabase SDK not loaded');
}

const supabaseUrl = window.SUPABASE_URL || 'https://<PROJECT>.supabase.co';
const supabaseKey = window.SUPABASE_KEY || '<ANON_PUBLIC_KEY>';

if (!supabaseKey || supabaseKey === '<ANON_PUBLIC_KEY>') {
  const msg = `Supabase anon anahtarı ayarlı değil. Lütfen Supabase Dashboard → Settings → API → Project API keys → anon public key'i GitHub Actions secret (SUPABASE_KEY) olarak ekleyin, veya test için config.js dosyası oluşturun.`;
  document.getElementById('sonuc').innerHTML = `<div class="error">${msg.replace(/\n/g,'<br>')}</div>`;
  throw new Error(msg);
}

const supabaseClient = supabase.createClient(supabaseUrl, supabaseKey);
```

4. Fix the broken/multiline `msg` string by using template literals (backticks) or properly escaped newlines.

5. Verify DB table names and columns used in JS match your Supabase table exactly (case and names):
   - In the repo we saw `.from('final_haziran26_mat2')` for SELECT, and earlier code used `final_mat2not` for update. Confirm these are correct and intentional.

6. Avoid committing `config.js` with a real key. If you have committed it for testing, remove it and rotate keys if necessary.

---

### 5) Debugging "Uncaught SyntaxError: Unexpected token '<'"

This error indicates a JS file was requested but HTML returned instead (starts with '<'). For example, `config.js` may return an HTML 404 page or GitHub's file view.

Steps:
1. Open DevTools → Network and filter for `config.js`.
2. Check HTTP status.
   - 200 + content-type `application/javascript` → OK.
   - 404 or 200 but HTML content → wrong resource being served.
3. If you are opening the file on GitHub (https://github.com/your/repo/blob/...), the server returns HTML (the repo page), not the raw file. Use either raw URL (`raw.githubusercontent.com/...`) or serve via GitHub Pages or a local server.
4. If GitHub Pages is serving the site, open the Pages URL (e.g. `https://<username>.github.io/<repo>/`) and check the Network response for `config.js` there.

---

### 6) Supabase permission issues (RLS)

If your fetch to Supabase returns 401/403 or empty results, it's often due to RLS (Row Level Security) or missing policies for the `anon` role.

To check from the browser console quickly (replace with your values):

```javascript
const url = window.SUPABASE_URL;
const key = window.SUPABASE_KEY;
fetch(`${url}/rest/v1/final_haziran26_mat2?select=*`, {
  headers: { 'apikey': key, 'Authorization': `Bearer ${key}` }
}).then(r => { console.log(r.status); return r.text(); }).then(console.log).catch(console.error);
```

If status is `401` or `403`, go to Supabase dashboard → Database → Policies and either disable RLS for the table (not recommended for prod) or add a policy that allows `anon` role to SELECT, for example:

```sql
-- allow read to anon (for testing only)
CREATE POLICY "Allow select for anon" ON public.final_haziran26_mat2
  FOR SELECT USING (true);
```

Make sure you understand the security implications.

---

### 7) Local testing

To avoid confusion between GitHub's file view and raw static hosting, test locally:

- Python 3:
  ```bash
  python -m http.server 8000
  # open http://localhost:8000/index.html
  ```

- Node (http-server):
  ```bash
  npx http-server -p 8000
  ```

Open the local URL and confirm no `Unexpected token '<'` errors and that `config.js` and the supabase CDN load with status 200.

---

### 8) Git push/pull conflicts ("fetch first")

If `git push` is rejected because the remote contains work you don't have locally:

```bash
git pull --rebase origin main
# resolve any conflicts if present
git push origin main
```

If you intentionally want to overwrite remote (rare and dangerous):

```bash
git push --force-with-lease origin main
```

Prefer pulling and resolving conflicts.

---

### 9) Remove `.github/workflows` folder

If you want to remove workflows:

A) Using git CLI (recommended):

```bash
# optional: create backup branch
git checkout -b backup/remove-workflows

# switch to main
git checkout main
git pull --rebase origin main

# remove the folder
git rm -r .github/workflows
git commit -m "Remove GitHub Actions workflows"
git push origin main
```

B) Using GitHub UI: delete each file in the `.github/workflows` folder and commit via the web editor.

---

### 10) Security notes

- Never commit service_role keys or other secrets. If you accidentally commit a secret, rotate it immediately and remove it from the repo history.
- Use repo secrets + workflows to inject secrets into build time.
- Do not rely on `config.js` with secrets in it being kept secret in a public repo.

---

## Final checklist before opening site publicly

- [ ] Workflow permissions set to **Read and write** (if workflow must push files)
- [ ] SUPABASE_KEY added as secret
- [ ] Deploy workflow creates `config.js` from secret OR you use an environment that provides it server-side
- [ ] index.html updated to use safe checks and clean JS comments
- [ ] Supabase policies allow anon SELECT if you expect client-side public reads (or provide a server-side endpoint)
- [ ] Test locally with `python -m http.server` and then test via Pages URL

---

If you want, I can:
- Commit the recommended index.html cleanup (convert remaining HTML comments to JS comments, fix the message string and add SDK checks).
- Modify or add the deploy workflow that creates config.js from secrets.

Tell me which change(s) to commit and I'll create the files/commits for you.
