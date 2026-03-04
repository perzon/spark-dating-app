# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Single-file SPA dating app (`index.html` only). No build step, no dependencies installed locally. Everything — HTML, CSS, and JS — lives in one file.

## Workflow
After every feature or fix: commit with a descriptive message and push, then deploy.

```bash
git add index.html
git commit -m "description"
git push
netlify deploy --dir=. --prod --site=dff57540-f87a-4733-a32e-7f614af04629
```

## Deploy

```bash
netlify deploy --dir=. --prod --site=dff57540-f87a-4733-a32e-7f614af04629
```

Live URL: https://sparkling-sunflower-665556.netlify.app

## Architecture

### Single-file structure (`index.html`)
- **CSS** (lines ~10–720): All styles inline in `<style>`. Page-specific sections are marked with `══` banners.
- **HTML** (lines ~721–952): Five pages as `<div class="page">` siblings inside `#app`, all `position:absolute;inset:0`. Visibility is controlled purely via the `active` class (opacity + pointer-events, no display toggling).
- **JS** (lines ~953–end): `<script type="module">` — module scope means nothing is global by default.

### Pages
| ID | Purpose |
|---|---|
| `page-loading` | Initial spinner shown while auth resolves |
| `page-signup` | Name/age/gender/preference/email form → sends magic link |
| `page-quiz` | 10-question personality quiz |
| `page-matches` | Ranked match grid fetched from Supabase |
| `page-profile` | Edit user profile and re-answer questions |

### Supabase backend
- **Auth**: Email magic link. `onAuthStateChange` is the sole auth event handler (uses `SIGNED_IN` + `SIGNED_OUT` events, plus initial `getSession()` with a `ready` flag to prevent race conditions).
- **DB tables**: `profiles` (15 seeded match candidates, read-only), `user_profiles` (one row per auth user, upserted on quiz completion and profile save).
- **Storage**: `avatars` bucket (public). Photos stored at path `avatars/{uid}/{uid}.jpg`.

### Key JS patterns
- All functions called from HTML `onclick` attributes must be explicitly exported: `Object.assign(window, { ... })` at the bottom of the script.
- `showPage(id)` — removes `active` from all `.page` elements, adds it to the target.
- `PROFILES` — populated async from Supabase `profiles` table (not hardcoded).
- `userPhoto` — holds base64 preview or public Storage URL. `userPhotoFile` holds the pending File object for upload. `currentPhotoPath` holds the existing Storage path.
- Match scoring: weighted Hamming distance across 10 answers. Max score per question = `weight × 2`.

### Auth flow
1. Signup form → `sendMagicLink()` → stores `{name,age,gender,preference}` in `sessionStorage` as `spark_pending`, sends OTP.
2. User clicks email link → redirected back → `SIGNED_IN` fires → `handleSignedIn()` checks for existing `user_profiles` row.
3. If no profile: reads `spark_pending` from sessionStorage → starts quiz.
4. Quiz completion → `saveUserProfileToDb()` upserts `user_profiles`.
5. Hard reload → `INITIAL_SESSION`/`getSession` → existing profile loaded → matches shown directly.
