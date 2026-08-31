# CLAUDE.md

Guidance for working in this repository.

## What this is

A single static HTML page served by nginx in a container. `index.html` holds the
whole app: one inline `<script>`, one inline `<style>`, no build step, no
framework, no client-side router (it reads `window.location.search` only).
`404.html` is the error page. Everything else is nginx config, CI, and docs.

## Never close a DAST issue by hand

The ZAP action files its findings as **issue #N — "ZAP Baseline DAST Report"**
and manages that issue's lifecycle itself: it appends each new report as a
comment, and it closes the issue on its own once a scan comes back with no
findings (see #12, closed by `github-actions[bot]`).

So when fixing a DAST finding:

- **Do not** put `Closes #N` / `Fixes #N` in the commit or merge message.
  GitHub then closes the issue the moment the PR merges — before any scan has
  verified the fix. The issue asserts "resolved" based on intent, not evidence.
- **Do** reference it (`Refs #N`) and leave it open.
- After the deploy is live, trigger the scan
  (`gh workflow run dast-scan.yml --ref main`). A clean report closes the issue
  by itself, and the closure then means a scanner confirmed it.

This went wrong once: #30 was closed by a `Closes #30` in the merge of #32,
90 minutes before the verifying scan ran. The fix happened to be complete, so
the outcome was right by luck. Had it not been, the closed issue would have
hidden a live finding, and the next scan would have opened a *new* issue rather
than updating #30 — splitting the history.

## Reading a DAST result

Two traps, both of which have caused wrong conclusions here:

1. **A green job does not mean no findings.** The ZAP action exits 0 with
   findings present. Check the alerts, not the check mark.
2. **The issue *body* is not the current report.** It stays frozen at the first
   report ever filed. Each later scan is appended as a **comment**. Read the
   newest comment — the body will still show old alerts and an old `RunnerID`.

The reliable source is the run's artifact:

```bash
gh run download <run-id> -D /tmp/zap
jq '.site[].alerts[] | {id: .pluginid, name, count, uri: .instances[0].uri, evidence: .instances[0].evidence}' \
  /tmp/zap/*/report_json.json
```

Suppressed rules and the reasoning behind each live in `.zap/rules.md`.

## CSP hashes

`security-headers.conf` pins a sha256 hash for every inline script. Change one
byte of the inline `<script>` in `index.html` — including a comment — and the
hash no longer matches, the browser blocks the script silently, and the page
still returns 200 while doing nothing. This shipped twice (#19, #20).

`.github/workflows/csp-hash-check.yml` gates every push and PR against this. If
it fails, recompute:

```bash
python3 -c "
import base64,hashlib,re
h=open('index.html',encoding='utf-8').read()
for m in re.finditer(r'<script\b(?![^>]*\bsrc\s*=)[^>]*>(.*?)</script>',h,re.S|re.I):
    print('sha256-'+base64.b64encode(hashlib.sha256(m.group(1).encode()).digest()).decode())"
```

and paste the value into `script-src` in `security-headers.conf`. Add
`'unsafe-hashes'` only if inline event handlers exist — they currently do not.

## Verifying a change locally

Repo files are mode 640; Docker `COPY` preserves that, so nginx cannot read them
and returns 403. Copy to a scratch dir and `chmod 644` before building:

```bash
mkdir /tmp/t && git archive HEAD | tar -x -C /tmp/t && chmod 644 /tmp/t/*
cd /tmp/t && docker build -q -t color-test . && docker run -d --name color-test -p 8099:80 color-test
```

CI checkouts are world-readable, so this is a local-only quirk — do not "fix" it
by changing file modes in the repo.

Worth checking after any nginx or CSP change:

```bash
curl -sS http://localhost:8099/sitemap.xml | grep -i nginx   # must be empty: no version banner
for p in / /?00ff80 /robots.txt /sitemap.xml /admin /404.html; do
  echo "$p $(curl -so /dev/null -w '%{http_code}' http://localhost:8099$p)"; done
chromium --headless --dump-dom "http://localhost:8099/?00ff80" | grep -o '<body style="[^"]*"'
```

Expected: `/`, `/?00ff80`, `/robots.txt` → 200; `/sitemap.xml`, `/admin`,
`/404.html` → 404; body background `rgb(0, 255, 128)`.

## Pre-commit

There is no lint/test/build here — no npm scripts, no source to compile. The
gates are the container build and the checks above. Bump the patch version in
`package.json` on every PR and run `npm install` to sync the lockfile.
