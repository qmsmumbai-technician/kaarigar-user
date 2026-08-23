# Kaarigar — Project Reference

*A two-sided local services marketplace (like Uber/Ola, but for home services). This document is the single source of truth for the project — architecture, decisions made, current status, and what's left.*

Last updated: reflects everything built through this conversation. Update this file yourself as things change — it's yours to keep.

---

## 1. What this is

- **User app**: browse 19 service categories, see technicians online *within 2 km* in real time, call them directly
- **Technician app**: register name + phone, select up to **2 services** they offer, toggle "Go Online" to broadcast live GPS
- **Model**: directory + call, not in-app booking/payment/tracking. User calls technician (or technician calls back) using the phone's native dialer — no in-app messaging or payment built.

## 2. Architecture

| Layer | Choice | Why |
|---|---|---|
| Frontend (both apps) | Plain HTML/CSS/JS, no framework | Works in any browser, no build step, easy to host free, convertible to an Android app without a laptop |
| Backend | **Firebase** — Firestore (database) | Built for exactly this: many devices writing live location, others reading in real time |
| Hosting | **GitHub Pages** (free, static) | No laptop needed — upload via GitHub's web interface |
| Android packaging | **PWABuilder** (pwabuilder.com) | Converts the hosted web app into an installable Android package (APK/AAB) without Android Studio |
| Distribution | Google Play Store | Requires a one-time $25 Play Console account |

**Why not native Android (Kotlin) or Flutter:** ruled out early because there's no laptop available — Android Studio requires a computer. Everything here works from a phone browser only.

## 3. Firebase project

- **Project ID:** `kaarigar-technician`
- **Firestore collection:** `technicians` — one document per technician, keyed by their phone number (digits only)

Document shape:
```
{
  name: string,
  phone: string,
  trades: string[],       // e.g. ["plumbing", "ac_technician"] — max 2
  lat: number,
  lng: number,
  online: boolean,
  rating: number,
  updatedAt: timestamp
}
```

**Web config** (safe to keep in client code — this is normal for Firebase, not a secret key):
```js
const firebaseConfig = {
  apiKey: "AIzaSyCVp07idh5slisdnJ6MWXc439_mEp17cns",
  authDomain: "kaarigar-technician.firebaseapp.com",
  projectId: "kaarigar-technician",
  storageBucket: "kaarigar-technician.firebasestorage.app",
  messagingSenderId: "717561353020",
  appId: "1:717561353020:web:0e4cc925fb7ab3d9b74439"
};
```

**Current Firestore rules status:** ⚠️ still open/permissive (anyone can read and write). A validated-but-unauthenticated rule set was provided earlier in this project to reduce junk data — confirm whether it was applied. Full protection (only the real phone-number owner can edit their own record) requires OTP/Firebase Auth, which was intentionally **deferred** due to cost (see §6).

## 4. The 19 service categories

Plumber · Electrician · Carpenter · AC Technician · RO/Water Purifier · Washing Machine · Refrigerator · TV Repairing · Home Cleaning · Pest Control · Window Sliding · Fabrication · Pooja Pandit · Computer Repair · Painter · Photography · Key Maker · Water Supplier · Kabadi Wala

Icons: real **Material Symbols** vector icons (Google's free icon font, via CDN), not emoji — colored per-trade, matches a polished reference design the founder shared.

**Location update frequency:** technicians broadcast their GPS every **60 seconds** while online (changed from an initial 15s — see §9, this was a deliberate cost-control decision).

**Branding footer:** both apps display `© 2022 Abode Digital Technology LLP - All rights reserved.` at the bottom of the main screen.

## 5. Repos & live URLs *(fill these in / confirm)*

| App | GitHub repo | Live URL | Play Package ID |
|---|---|---|---|
| Technician | `qmsmumbai-technician/kaarigar-technician` | `https://qmsmumbai-technician.github.io/kaarigar-technician/` | `io.github.qmsmumbai_technician.twa` |
| User | ______________ | ______________ | ______________ *(must differ from Technician's)* |

**Signing key reminder:** ⚠️ Whatever keystore file PWABuilder generated for each app must be kept safe (Drive, email-to-self, etc.) — it's required for every future update of that app and can't be recovered if lost.

## 6. Decisions made along the way

- **OTP/phone verification**: built once (Firebase Phone Auth), then **removed** to avoid the Blaze (pay-as-you-go) plan and per-SMS cost until the business has revenue. Can be re-added later — it's a clean add-on, not a rebuild.
- **Max 2 services per technician**: enforced in the UI; picking a 3rd shows "remove one first."
- **2 km range limit**: User app hides any technician beyond 2 km, even if online.
- **"Either party can call"**: no extra engineering needed — once the user calls once, the technician has their number via normal phone caller ID and can call back anytime.
- **Icons**: switched from emoji to Google's Material Symbols icon font — free, no copyright issue, matches the polished reference look.
- **App name**: not finalized. Candidates discussed: Kaarigar *(current)*, FixWala, Apna Kaarigar, GharSeva, Sahayak, LocalMistri, TurantFix, SewaSetu, GharGuru, FixMitra.

## 7. Known limitations (be upfront about these)

- Location only updates while the app is **open and screen on** — no true background tracking yet
- Firestore security rules are not production-hardened (see §3)
- No push notifications — User app updates via a live Firestore listener (instant while open), not background push
- No in-app payments, ratings are currently static placeholder values, no booking/scheduling — by design, matches the original "directory + call" scope

## 9. Firebase cost management (important, read this)

Firestore storage itself scales to billions of documents — that's never the limit. The real constraint is the **free Spark plan's daily quota**: 50,000 reads/day, 20,000 writes/day, 1 GB storage.

Because technicians write their location on a timer AND users have a live real-time listener open, **reads multiply fast** — every technician update gets re-delivered to every user currently watching the list. With even a few dozen concurrent users, the free daily read quota can be exhausted within an hour or two, well before technician *count* becomes the bottleneck.

**Mitigation already applied:** heartbeat interval increased from 15s → **60s**, roughly a 4x reduction in both writes and the read fan-out cost, at the cost of positions being up to a minute stale (acceptable for a "call nearby" directory, not a live-tracking map).

**When you outgrow the free tier:** move to the Blaze (pay-as-you-go) plan — same upgrade OTP would have required. Blaze has **no spending cap by default**, so set a budget alert in Google Cloud Console the same day you upgrade.

## 10. Not done yet / next steps

- [ ] Confirm User app's GitHub repo + Play package ID (fill into §5)
- [ ] Tighten Firestore rules (validated rule set was provided; confirm it's live)
- [ ] Write and host a **privacy policy** page (required by Play Store since the app collects location + phone numbers)
- [ ] Submit both apps' `.aab` files via Google Play Console
- [ ] Decide on final app name + icon, regenerate via PWABuilder if changed (icon/name changes require repackaging; content-only changes don't)
- [ ] Revisit OTP/Firebase Auth once there's budget, for real account security

---

*Tip: save this file (or a copy) inside each GitHub repo as `PROJECT.md` so it's version-controlled alongside the code — GitHub renders markdown nicely right on the repo page.*
