# LoToos Audit Session Walkthrough

Target: https://lotoos-luxury-bar-chairs-931650077520.us-west1.run.app
Source: Lotoos-main.zip (React SPA, Vite, Firebase, Cloud Run, Persian luxury furniture)

## Key Findings Found in This Session

### 1. Firestore wide open
`firestore.rules` had `allow read, write: if true`. The single most dangerous
finding. Anyone with the Firebase project ID (exposed in the JS bundle) can
read/write all chairs, bookings, and materials.

### 2. Hardcoded admin passcode in client-side JS
```
if (nextPass === "1379" || nextPass === "2026") { setIsUnlocked(true); }
```
Hint shown: "سال تولد یا سال اجرای آتلیه (۱۳۷۹ یا ۲۰۲۶)". Entire admin panel
(inventory CRUD, booking management, material editing) ships to every visitor.

### 3. Guessable booking IDs
```
id: "LT-" + Math.floor(100000 + Math.random() * 900000)
```
Not crypto-random. Combined with open Firestore, enumerable.

### 4. Seed data overwrites Firestore on every load
`AppContext.tsx` syncs all chairs and materials to Firestore on mount. If an
admin modified stock levels, a visitor loading the page resets everything.

### 5. Unused deps in package.json
`express`, `dotenv`, `@types/express` listed but app has no server.
Pure SPA on Cloud Run serving static files.

### 6. No security headers
CSP, X-Frame-Options, HSTS, X-Content-Type-Options all missing.

### 7. SEO blind spots
No meta description, no OG tags, no Twitter cards, no JSON-LD, no sitemap,
no robots.txt. Zero alt text on all images. Your Persian luxury brand is
invisible to social media previews.

## Commands That Worked

Header inspection:
```bash
curl -s -D- -L -A "Mozilla/5.0" "https://target.site" | grep -i -E "content-security-policy|x-frame-options|strict-transport|content-type"
```

Path probing:
```bash
for path in /robots.txt /sitemap.xml /.env /admin /api; do
  curl -s -o /dev/null -w "%{http_code}" "https://target.site$path"
done
```

Secret scanning:
```bash
grep -oP '(?:sk_live|sk_test|pk_live|pk_test|AIza|ghp_|api[Kk]ey|secret|password|token).{0,50}' /tmp/site.js
```

Firebase config exposure:
```bash
grep -oP '(?:firebase|projectId|storageBucket|apiKey|authDomain)[^,}]*' /tmp/site.js | head -20
```

Auth bypass patterns:
```bash
grep -oP 'passcode|unlock|bypass|isAdmin|isUnlocked|=== ['"'"'"]' /tmp/site.js | sort | uniq -c
```

Persian text extraction:
```bash
python3 -c "import re; c=open('/tmp/site.js').read(); persian=re.findall(r'[\u0600-\u06FF\uFB50-\uFDFF\uFE70-\uFEFF]{4,}',c); [print(p) for p in sorted(set(persian))]"
```

Image format audit:
```bash
grep -c 'webp' /tmp/site.js; grep -c '\.jpg' /tmp/site.js
```

## Patching Strategy (ordered by impact)

1. Fix `firestore.rules` first — this is the only thing preventing data theft.
2. Add meta/OG tags to `index.html` — five minutes, big SEO + shareability win.
3. Replace passcode with real Firebase Auth + admin claim check.
4. Stop auto-seeding Firestore on page load.
5. Prune unused deps, add security headers via the hosting platform.
