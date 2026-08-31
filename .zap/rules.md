# ZAP DAST Ignore Rules

Rules suppressed in `.zap/rules.tsv` with justification:

| Rule ID | Name | Reason |
|---------|------|--------|
| 10035 | Strict-Transport-Security Header Not Set | HSTS is configured in Cloudflare (SSL/TLS → Edge Certificates → HSTS). ZAP scans the HTTP→HTTPS redirect path and does not see the header Cloudflare injects on HTTPS responses. |
| 10055 | CSP: style-src unsafe-inline / Missing Directives | `style-src unsafe-inline` is required by the single inline `<style>` block in `index.html`. `form-action` and `base-uri` are explicitly defined in the CSP — ZAP flags these as informational when `default-src` has no fallback for them. |
| 10109 | Modern Web Application | Informational finding — ZAP heuristically classifies the page as a modern web app. Not a vulnerability. |
| 10015 | Re-examine Cache-control Directives | Informational finding about cache headers. Not a vulnerability. |
| 10049 | Storable and Cacheable Content | Informational finding. Static assets are intentionally cacheable. |
| 10096 | Timestamp Disclosure - Unix | Build timestamps embedded in JS bundles by the build tool. Not sensitive data. |
| 10050 | Retrieved from Cache | Cloudflare CDN caching behavior. Not a vulnerability. |

## Rules deliberately NOT suppressed

| Rule ID | Name | Handling |
|---------|------|----------|
| 10027 | Information Disclosure - Suspicious Comments | Fixed at the source instead of suppressed. ZAP matches a keyword list (`from`, `user`, `select`, `todo`, …) against comments, so plain English wording can trip it. The comments in `index.html` were reworded rather than the rule silenced, keeping 10027 active for genuine leaks. |

## Why findings were reported three times

`nginx.conf` used an SPA fallback (`try_files $uri $uri/ /index.html`), so every
unmatched path returned `index.html` with HTTP 200 — including `/sitemap.xml`,
which this site does not have. ZAP therefore scanned the same page under several
URLs and reported each finding once per URL.

This page has no client-side router (it only reads `location.search`), so the
fallback served no purpose. It is now `try_files $uri $uri/ =404`: real files are
served, everything else 404s, and security headers still apply to the 404
response because they are declared with `always`.
