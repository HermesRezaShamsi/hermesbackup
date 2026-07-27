---
name: web-application-audit
description: Audit web apps via curl, bundle inspection, and source code.
version: 1.1.0
metadata:
  hermes:
    tags: [security-audit, seo-audit, performance, web, review, code-analysis]
    related_skills: [dogfood, requesting-code-review]
---

# Web Application Audit

Systematic audit of a live web application using server-side tooling: curl,
bundle analysis, header inspection, source code review. No browser required.
Covers security, SEO, performance, architecture, and UX.

## When to Use

- User says "check this website", "find weaknesses", "audit this site", "what needs improvement"
- Before deploying a new web app to production
- When evaluating a third-party site or competitor
- After the user provides a URL or source code (ZIP/repo)

**Skip for:** simple design feedback ("how does this look?" — use sketch or claude-design), browser-level functional QA (use dogfood).

**This skill vs dogfood:** dogfood uses the browser tool to click around and find functional bugs. This audit uses curl, source code analysis, and header inspection to find architectural, security, and SEO issues.

**This skill vs requesting-code-review:** code-review verifies a git diff before commit. This audit reviews a deployed live site or its full source bundle.

## Phase 1 — Reconnaissance

### 1a. Fetch and inspect the HTML shell

```
curl -s -L -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64)" \
  "https://target.site" -o /tmp/site.html
```

Check: meta tags, title, favicon, OG tags, RTL direction, lang attribute, JSON-LD structured data, CSP/nonce in meta tags.

### 1b. Check response headers

```
curl -s -D- -L -A "Mozilla/5.0" "https://target.site" | head -20
```

Checklist:
- `content-security-policy` — missing = XSS risk
- `x-frame-options` — missing = clickjacking risk
- `strict-transport-security` — missing = no HSTS
- `x-content-type-options` — missing = MIME sniffing risk
- `referrer-policy` — missing = referrer leakage
- `cache-control` — should be set for static assets
- `set-cookie` — check for secure/httponly flags

### 1c. Common path probing

Probe for exposed endpoints that an SPA catch-all (200 for every path) might hide:

```
for path in /robots.txt /sitemap.xml /.env /admin /api /config.json; do
  curl -s -o /dev/null -w "%{http_code}" "https://target.site$path"
done
```

- 200 on /.env or /admin in an SPA = shell page, not the real file.
- 200 on /robots.txt with real content = likely legitimate.
- 404 or 410 = proper handling.

### 1d. Scan for robots.txt / sitemap

```
curl -s "https://target.site/robots.txt" | head -20
curl -s "https://target.site/sitemap.xml" | head -20
```

## Phase 2 — Source Code / Bundle Analysis

### 2a. Download JS and CSS bundles

Find paths from the HTML `<script src>` and `<link rel="stylesheet">` tags.

```
curl -s "https://target.site/assets/index-xxxxx.js" -o /tmp/site.js
curl -s "https://target.site/assets/index-xxxxx.css" -o /tmp/site.css
```

### 2b. Bundle size check

```
ls -lh /tmp/site.js /tmp/site.css
```

- JS > 500KB for a marketing/content site = too large, consider code splitting.
- Large bundle + no code splitting = performance issue.

### 2c. Check for secrets in the bundle

```
grep -oP '(?:sk_live|sk_test|pk_live|pk_test|AIza|ghp_|api[Kk]ey|secret|password|token).{0,50}' /tmp/site.js
```

### 2d. Check for exposed Firebase config

```
grep -oP '(?:firebase|projectId|storageBucket|apiKey|authDomain)[^,}]*' /tmp/site.js | head -20
```

If found, rate this finding based on Firestore rules (Phase 3).

### 2e. Check for client-side auth/bypass patterns

```
grep -oP 'passcode|unlock|bypass|isAdmin|isUnlocked|=== ['"'"'"]' /tmp/site.js | sort | uniq -c
```

### 2f. Extract component structure and routes

```
grep -oP '(?:route|path|to|navigate)[":]?\s*["'"'"'][^"'"'"]+["'"'"']' /tmp/site.js | sort -u
grep -oP '(?:const|function|class)\s+\w+' /tmp/site.js | sort -u | head -30
```

### 2g. Extract meaningful RTL text content

```
python3 -c "
import re
with open('/tmp/site.js', 'r') as f: c = f.read()
persian = re.findall(r'[\u0600-\u06FF\uFB50-\uFDFF\uFE70-\uFEFF]{4,}', c)
for p in sorted(set(persian)): print(p)
"
```

### 2h. Extract image references

```
grep -oP 'https?://[^"'"'"');,]+\.(?:jpg|png|webp|avif|svg)' /tmp/site.js | sort -u
```

Check: webp/avif usage, lazy loading, alt text, responsive breakpoints.

### 2i. Check content-type usage

```
grep -c 'webp' /tmp/site.js
grep -c '\.jpg' /tmp/site.js
```

Zero webp references on a modern site = image optimization opportunity.

### 2j. Extract API endpoints

```
grep -oP '(?:fetch|axios|api)[^)]*\)' /tmp/site.js | grep -oP 'https?://[^"'"'"');,]+' | sort -u | head -20
```

### 2k. Check for analytics

```
grep -oP 'gtag|GA_MEASUREMENT|google-analytics|facebook-pixel|meta-pixel' /tmp/site.js | sort | uniq -c | sort -rn
```

## Phase 3 — Source Code Review (if repo/ZIP is available)

### 3a. Security configs

- **Firestore rules** (`firestore.rules`): `allow read, write: if true` is the single most dangerous finding in a Firebase app.
- **Auth logic**: hardcoded passcode/PIN or client-side admin gating = security finding (client-side auth = no auth).
- **Secrets in source**: API keys, tokens, passwords in committed code.
- **Environment config**: `.env.example` documenting real secrets, or `.env` not in `.gitignore`.

### 3b. Architecture

- Check `package.json` dependencies for unused packages (express + dotenv in a pure SPA, etc.).
- Server vs static serving (Cloud Run hosting static files vs SSR).
- Data flow: server-side validation or direct client-to-database writes?
- Booking/order ID generation: crypto-random or predictable?

### 3c. Performance in source

- Image `loading="lazy"` usage
- Alt text on images
- Hero image preloading
- Bundle splitting / lazy routes
- Seed-data overwriting on every page load

### 3d. SEO in source

- Meta description in `index.html`
- OG / Twitter card meta tags
- JSON-LD structured data (Product, Organization, Review schemas)
- Semantic heading hierarchy (h1, h2)
- `dir="rtl"` and `lang` attributes for Persian/Arabic sites

### 3e. UX / Business

- Contact channels (phone, WhatsApp, Telegram, email, form)
- Cart / checkout / payment flow
- Admin/booking notifications (email, SMS, Telegram bot)
- Input validation patterns
- Error handling on failed API calls

## Phase 4 — Report

Structure findings in priority order:

### 🔴 Critical (security vulnerabilities)
Must fix before production. Open Firestore rules, hardcoded passcodes, exposed admin panels, no rate limiting on writes.

### 🔴 High (security hardening)
Should fix. Missing CSP/HSTS/XFO headers, Firebase config exposed with permissive rules, guessable booking IDs.

### 🟡 Medium (SEO / Performance / Architecture)
Should fix for product readiness. Missing meta tags, no OG cards, no sitemap, no image lazy loading, large bundle, unused deps, no analytics, no notifications.

### 🟢 Low (UX / Polish)
Nice to have. Phone-only contact, no checkout flow, no alt text, seed-data overwriting on every load.

End with a **Quick Wins** table:
| Priority | Fix | Effort | Impact |
|---|---|---|---|

## Phase 5 — Fix (when requested)

When the user says "fix these" or "implement these improvements" after the audit:

### 5a. Deploy fixes in priority order

Always fix Firestore rules and auth first — these are the only things preventing data loss.

### 5b. Common fix patterns for Firebase React SPAs

**Firestore rules lockdown (role-based)**:
```
match /chairs/{chairId} {
  allow read: if true;
  allow write: if request.auth != null
    && request.auth.token.email == "admin@lotoos.design";
}
match /bookings/{bookingId} {
  allow create: if true;
  allow read, update, delete: if request.auth != null
    && request.auth.token.email == "admin@lotoos.design";
}
match /{document=**} { allow read, write: if false; }
```

**Replacing client-side passcode with Firebase Auth**:
- Import `signInWithEmailAndPassword`, `signOut`, `onAuthStateChanged` from firebase/auth
- Add a `useEffect` with `onAuthStateChanged` listener for session persistence
- Replace numeric keypad with email + password form
- Add per-error-code localized error messages (use Persian for fa-lang sites)
- Add logout button with auto-lock on close

**Removing seed-data overwrites**:
- Check AppContext / provider files for `setDoc` calls on mount
- Remove the sync function, keep only `onSnapshot` subscriptions

**Pruning unused deps** in a Vite SPA:
- `express` and `dotenv` are dead in a pure SPA (no server)
- Remove `@types/express`, `@types/node` too
- Vite in devDependencies is a mistake — it should be in dependencies for build

**Stripping console logs in production** (vite.config.ts):
```ts
esbuild: {
  drop: ['console', 'debugger'],
},
```

**Meta/OG/SEO overhaul for index.html** — add in one pass: description, keywords, canonical, OG tags, Twitter Card, JSON-LD structured data (Organization schema for Persian brands), preload LCP image with responsive imagesrcset/imagesizes, preconnect/dns-prefetch to external image CDN.

**Image performance** — add `loading="lazy"` to every `<img>`, add Persian alt text.

### 5c. Build verification

After applying fixes, always:
```bash
npm install   # verify deps resolve
npm run build # verify production build compiles cleanly
ls dist/      # verify robots.txt and sitemap.xml made it in
```

### 5d. Delivery

Offer the user:
1. A zipped copy of the fixed source
2. Or commit instructions for their existing repo
3. A summary of what changed and why

## Linked References

- `references/lotoos-audit-session.md` — Walkthrough of a real audit against a Persian luxury furniture SPA (Firebase, Cloud Run, React + Vite). Contains specific curl commands and grep patterns used to find the issues listed.
- `references/lotoos-fix-session.md` — Continuation: fixes applied to the same app, with code patterns for Firebase Auth integration, Firestore rules, SEO overhaul, and performance fixes.
- `references/https-git-backup-silent-cron.md` — Pattern for HTTPS git backups with silent cron jobs (port 22 bypass).

## Pitfalls

- **SPA catch-all returns 200 for everything** — A `/.env` returning 200 with shell HTML is NOT a real /.env file. Distinguish SPA client-side routing from actual exposed files.
- **Minified JS is huge** — For 1MB+ bundles, extract key patterns with grep rather than reading the full file. Focus on: hardcoded strings (especially Persian), API URLs, config objects, conditional auth logic.
- **Unsplash direct URLs** — Common in prototypes. Flag as optimization opportunity (not error) and suggest webp/avif + lazy loading.
- **Firebase API key is not a secret** — Firebase API keys are meant to be public. Flag rules, not the key itself.
- **User owns source, not production build** — User may send ZIP (audit source) or URL only (audit bundles). The two may have different findings (e.g. API key redacted in source but inlined during build).
