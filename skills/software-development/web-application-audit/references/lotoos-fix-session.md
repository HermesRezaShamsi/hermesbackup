# LoToos Fix Session Walkthrough

Continuation of the LoToos audit: applying the fixes identified in lotoos-audit-session.md.

## Fixes Applied

### 1. Firebase Auth replacement for passcode

**Before:** Client-side hardcoded passcode check (`"1379" || "2026"`) in AdminPanel.tsx.
**After:** Firebase `signInWithEmailAndPassword` with full email/password login form.

Key changes:
- Import `auth`, `signInWithEmailAndPassword`, `signOut`, `onAuthStateChanged` from firebase.ts
- Add `useEffect` with `onAuthStateChanged` listener to detect existing login sessions
- Replace the numeric keypad UI with an email + password form
- Add `handleLogin` async function with per-error-code Persian error messages
- Add `handleLogout` button in the unlocked admin header
- Add `LogOut`, `Mail`, `KeyRound`, `AlertCircle` icons from lucide-react

Error handling pattern for Firebase Auth in Persian:
```tsx
if (err.code === "auth/user-not-found" || err.code === "auth/wrong-password") {
  setAuthError("ایمیل یا رمز عبور اشتباه است.");
} else if (err.code === "auth/too-many-requests") {
  setAuthError("تلاش‌های زیاد. لحظاتی بعد دوباره تلاش کنید.");
} else {
  setAuthError("خطا در ورود. لطفاً دوباره تلاش کنید.");
}
```

### 2. Stopping seed-data overwrite on every page load

**Problem:** AppContext.tsx called `setDoc(doc(db, "chairs", chair.id), chair)` for every chair and material on every mount, overwriting admin changes.
**Fix:** Remove the `syncLatestDataToFirestore()` function entirely. Keep only the `onSnapshot` subscriptions that read from Firestore.

### 3. Removing unused dependencies

Express, dotenv, @types/express, @types/node removed from package.json. Added @types/react and @types/react-dom which were missing.

### 4. Stripping console.logs in production

`vite.config.ts`:
```ts
esbuild: {
  drop: ['console', 'debugger'],
},
```

### 5. Firestore rules lockdown

Role-based rules: chairs/materials public-read/admin-write, bookings anyone-create/admin-read-update-delete. Catch-all deny.

### 6. Meta/OG/SEO overhaul for index.html

Added in one pass: meta description, keywords, canonical link, Open Graph / Facebook / Telegram meta tags, Twitter Card tags, JSON-LD structured data for Organization, preload LCP image with responsive imagesrcset, preconnect + dns-prefetch to Unsplash.

### 7. robots.txt + sitemap.xml

Added to public/ so Vite copies them to build output.

### 8. Image lazy loading + alt text

Added loading="lazy" to all 8 img tags across all components. Updated alt text to Persian descriptions.

## Build Verification

npm install (205 packages), npm run build (5.28s, no errors). Output: dist/ with index.html, CSS, JS, robots.txt, sitemap.xml.

## Key Architectural Insight

The app is a pure SPA on Cloud Run with no server. Express and dotenv deps were left over from a previous attempt. All data flows client-side via Firebase SDK. This means no server-side validation, rate limiting, or notifications. The Cloud Run deployment could be replaced with Firebase Hosting or Cloud Storage.

## Auth Gotcha

To use signInWithEmailAndPassword, enable the Email/Password sign-in provider in Firebase Console → Authentication → Sign-in method. Not enabled during the session.
