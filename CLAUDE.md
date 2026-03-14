# Vera Oracle — veraoracle.com

## What this is

A single-page website for Vera Oracle, a tarot reader and spiritual advisor based in Richmond upon Thames, London. The site features an interactive WebGL aura background that responds to mouse/touch movement, a booking form that feeds into the Parinameh CRM, and SEO targeting local spiritual guidance searches.

## Architecture

```
veraoracle.com (static HTML on Vercel)
  └─ Booking form → Supabase `booking_requests` table (INSERT only, anon)
                            ↓
parinameh.veraoracle.com (Vite+React on Vercel)
  └─ Auto-imports bookings → creates clients + sessions in Parinameh
```

Both sites share one Supabase project (`odqnbtnbbvqktaqejuxh`) but use different tables. Veraoracle's form can only INSERT into `booking_requests`. Parinameh reads and updates that table via an authenticated session.

## Key files

| File | Purpose |
|---|---|
| `index.html` | The entire site — HTML, CSS, WebGL shader, booking form, Supabase integration |
| `Parinameh/src/parinameh.jsx` | CRM app — auto-imports bookings into client list |
| `Parinameh/src/supabaseClient.js` | Shared Supabase client config |

## Infrastructure

### Hosting
- **Veraoracle**: Vercel project `veraoracle`, auto-deploys from `github.com/timoarnall/veraoracle` on push to `main`
- **Parinameh**: Vercel project `parinameh`, auto-deploys from `github.com/timoarnall/parinameh` on push to `main`
- Both under Vercel org `team_z4yKKtSNu3IueF6SWJ6Yqdsh`

### DNS (Porkbun)
| Type | Host | Value |
|---|---|---|
| A | veraoracle.com | 76.76.21.21 (Vercel) |
| CNAME | www.veraoracle.com | cname.vercel-dns.com |
| A | parinameh.veraoracle.com | 76.76.21.21 (Vercel) |

### Supabase
- **Project**: parinameh (`odqnbtnbbvqktaqejuxh`), West Europe (London)
- **Tables used by Veraoracle**: `booking_requests` (INSERT only via anon key)
- **Tables used by Parinameh**: `parinameh_data` (full CRUD via authenticated session), `booking_requests` (SELECT/UPDATE via authenticated session)

## Database: booking_requests

```sql
id              uuid        PRIMARY KEY, auto-generated
name            text        NOT NULL
email           text        nullable (either email or phone required)
phone           text        nullable
preferred_date  date        nullable
preferred_time  text        nullable (15-min slots: "08:00", "08:15", ... "22:00")
reading_type    text        nullable (matches Parinameh's 13 READING_TYPES)
message         text        nullable (passes through to client Notes in Parinameh)
status          text        NOT NULL, default 'pending'
                            check: 'pending', 'accepted', 'dismissed', 'imported'
created_at      timestamptz default now()
```

### Row-Level Security
- **Anon** (public visitors): INSERT only — cannot read or delete
- **Authenticated** (Pari logged in): SELECT + UPDATE

### Rate limiting
Trigger function `check_booking_rate_limit()` blocks more than 3 submissions per email or phone per hour.

### Status flow
```
pending → imported (auto-imported into Parinameh client list)
pending → dismissed (Pari manually dismisses)
```

## Booking form → Parinameh flow

1. Visitor fills form on veraoracle.com (name, email/phone, date, time, reading type, message)
2. Form INSERTs into `booking_requests` with status `pending`
3. Contact field auto-detects email vs phone (checks for `@`)
4. Parinameh auto-imports on load: fetches all `pending`/`accepted` bookings
5. For each booking:
   - If client exists (matched by email or name) → adds session, appends message to notes
   - If new client → creates client with phone, email, notes from message, and a session
6. All imported bookings marked `status: 'imported'` so they don't re-import

### Reading types (shared between both apps)
Career & purpose, Love & relationships, Family & home, Money & abundance, Health & wellbeing, Grief & loss, Spiritual growth, Shadow work, Year ahead, Past life, Decision & crossroads, Creativity & self-expression, Travel & new beginnings

### Time format
Both apps use 15-minute slots from 08:00 to 22:00 (e.g. "14:30"). The booking form generates these dynamically to match Parinameh's `TIME_OPTIONS`.

## Design decisions

### Visual
- **WebGL aura background** — simplex noise FBM with 5 layers (nebula, volumetric light, tendrils, sparkles, halo) plus touch trail and energy burst
- **Dark, minimal aesthetic** — Apple-inspired, SF Pro on Apple devices, Inter on Windows/Android
- **No scroll** — single viewport, `overflow: hidden`. The aura responds to mouse/finger movement instead of scroll
- **Movement-driven energy** — pointer speed drives both the swirl phase (color shifting/morphing) and energy level (brightness). Touch gets boosted multipliers since events fire less frequently
- **"You were meant to find this"** — hidden reveal text that fades in after sustained interaction (threshold: 8.0 cumulative energy, 3x faster accumulation on touch)

### Booking modal
- **Center modal** with spring animation, not a bottom sheet
- **Shimmer border** — animated conic gradient on a wrapper element (not the modal itself, to avoid `overflow` clipping on mobile)
- **Floating particles** — 24 particles drift upward inside the modal
- **Staggered field entrance** — each field fades/slides in with cascading delay
- **Button sparkles** — 30 on mobile, 16 on desktop, animated around the submit button
- **Contact field** — single field for email or phone, auto-detected by `@` presence
- **Modal dismissal** — X button always works; tap-outside only works after successful submission
- **Thank-you celebration** — glowing orb with orbiting dots, 30 burst sparks, 14 floating embers, staggered text entrance. Stays visible until user taps outside.

### Mobile
- `user-select: none` + `-webkit-touch-callout: none` prevents text selection / blue highlight
- Touch energy multiplier doubled (8x vs 4x desktop)
- Trail deposits every frame on touch (vs every 3rd frame on desktop)
- `touchActive` interpolation faster on touch (0.06 vs 0.012)
- Date input: `color-scheme: dark`, explicit `min-height: 40px`
- Form inputs explicitly re-enable `user-select: text`

### SEO
- Title: "Vera Oracle — Tarot Readings & Spiritual Guidance in Richmond, London"
- Meta description targeting: tarot, spiritual guidance, oracle, Richmond, Southwest London
- Keywords meta covering: tarot reader, psychic reading, clairvoyant, energy healing, aura reading, holistic healing across Richmond/London variations
- JSON-LD `LocalBusiness` schema with service types and area served
- Open Graph tags with `en_GB` locale
- Geo meta tags for Richmond upon Thames
- Hidden crawlable text content (screen-reader accessible) with natural keyword-rich copy since the visible page is mostly canvas
- Canonical URL set to `https://veraoracle.com`

### Typography
- Apple: SF Pro Display / SF Pro Text (system font)
- Windows/Android: Inter (loaded from Google Fonts CDN, weights 400/500/600)
- Fallback chain: `-apple-system, 'SF Pro Display', 'SF Pro Text', Inter, 'Helvetica Neue', Helvetica, Arial, sans-serif`

## Development

No build step. Edit `index.html`, push to `main`, Vercel auto-deploys.

```bash
# Local preview
open index.html

# Deploy
git add index.html
git commit -m "description"
git push origin main
# Vercel deploys automatically
```

### Supabase migrations
Migrations were pushed via a temp Supabase project linked to ref `odqnbtnbbvqktaqejuxh`:
- `20260313000000_booking_requests.sql` — initial table + RLS + rate limit
- `20260313010000_booking_phone.sql` — made email nullable, added phone column, updated rate limit
