# Vera Oracle — veraoracle.com

Complete technical documentation for reproducing the site.

## Architecture

```
veraoracle.com (static HTML on Vercel)
  ├─ Booking form → Supabase `booking_requests` table (INSERT only, anon)
  ├─ Page views   → Supabase `page_views` table (INSERT only, anon)
  └─ Email notify → Supabase Edge Function `notify-booking` → Resend → Pari's Gmail
                            ↓
parinameh.veraoracle.com (Vite+React on Vercel)
  ├─ Auto-imports bookings → creates clients + sessions in Parinameh
  └─ Reads page_views → shows Stats (today/week/month, sources, countries, search terms)
```

Both sites share one Supabase project but use different tables. The public form can only INSERT. Pari's authenticated session can SELECT and UPDATE.

## File structure

```
veraoracle/
├── index.html         # The entire site (HTML + CSS + WebGL + JS + Supabase)
├── robots.txt         # Allows all crawlers
├── sitemap.xml        # Single-page sitemap for Google
├── render.html        # Postcard generator (25 color variants, 127x177.8mm at 600 DPI)
├── render-cards.html  # Business card generator (25 variants, 70x28mm at 600 DPI)
├── CLAUDE.md          # This file
├── .gitignore         # Ignores .vercel/
└── .vercel/           # Vercel project config (not committed)
```

## Infrastructure

### Hosting
- **Veraoracle**: Vercel project `veraoracle` (ID `prj_vdbysj5BGJIigboPP9Q1FRY22n6S`)
- **Parinameh**: Vercel project `parinameh` (ID `prj_xE4g3t7zXkWbz7Ljovo6pL4Yd6Rd`)
- Both under Vercel org `team_z4yKKtSNu3IueF6SWJ6Yqdsh` (account: `timoarnall`)
- Both auto-deploy from GitHub on push to `main`
- No build step for Veraoracle — Vercel serves static files directly

### GitHub repos
- `github.com/timoarnall/veraoracle` → deploys to `veraoracle.com`
- `github.com/timoarnall/parinameh` → deploys to `parinameh.veraoracle.com`

### DNS (Porkbun — veraoracle.com)
| Type | Host | Value |
|---|---|---|
| A | veraoracle.com | 76.76.21.21 (Vercel) |
| CNAME | www.veraoracle.com | cname.vercel-dns.com |
| A | parinameh.veraoracle.com | 76.76.21.21 (Vercel) |
| MX | veraoracle.com | fwd1.porkbun.com (priority 10) |
| MX | veraoracle.com | fwd2.porkbun.com (priority 20) |
| TXT | veraoracle.com | v=spf1 include:_spf.porkbun.com ~all |
| TXT | _acme-challenge.veraoracle.com | (Vercel SSL verification tokens) |

### Supabase
- **Project**: parinameh, ref `odqnbtnbbvqktaqejuxh`, West Europe (London)
- **Anon key**: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Im9kcW5idG5iYnZxa3RhcWVqdXhoIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzM0MjI3NDIsImV4cCI6MjA4ODk5ODc0Mn0.1D-imafIpPFdcBRO2BZAry2h3cQs60dfvOcRJyVYxqA`
- **Supabase JS**: loaded from CDN `https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.min.js`

### Edge Function: notify-booking
- **Endpoint**: `https://odqnbtnbbvqktaqejuxh.supabase.co/functions/v1/notify-booking`
- **Purpose**: emails Pari (vatini.p@gmail.com) when a booking is submitted
- **Email service**: Resend (API key stored as Supabase secret `RESEND_API_KEY`)
- **From address**: `bookings@veraoracle.com` (requires Resend domain verification)
- **Sends**: formatted HTML email with name, contact, date, time, reading type, message
- **Called**: fire-and-forget from client JS after successful booking insert

---

## Database tables

### booking_requests

```sql
create table booking_requests (
  id              uuid        default gen_random_uuid() primary key,
  name            text        not null,
  email           text,       -- nullable, either email or phone required
  phone           text,       -- nullable
  preferred_date  date,
  preferred_time  text,       -- 15-min slots: "08:00", "08:15" ... "22:00"
  reading_type    text,       -- one of 12 reading types (see below)
  message         text,       -- passes to client Notes in Parinameh
  status          text        not null default 'pending'
                              check (status in ('pending','accepted','dismissed','imported')),
  created_at      timestamptz default now()
);
```

**RLS policies:**
- Anon: INSERT only (cannot read or delete)
- Authenticated: SELECT + UPDATE

**Rate limit trigger**: `check_booking_rate_limit()` — max 3 submissions per email or phone per hour.

**Status flow**: `pending` → `imported` (auto-imported into Parinameh) or `dismissed` (Pari manually dismisses)

### page_views

```sql
create table page_views (
  id         bigint      generated always as identity primary key,
  path       text        not null default '/',
  referrer   text,
  country    text,       -- 2-letter country code from api.country.is
  created_at timestamptz default now()
);
```

**RLS policies:**
- Anon: INSERT only
- Authenticated: SELECT only

### parinameh_data (used by Parinameh only)
- Single row per user, stores all financial + client data as JSON
- Not accessed by Veraoracle

---

## Reading types (12 — shared between both apps)

Career & purpose, Love & relationships, Family & home, Money & abundance, Grief & loss, Spiritual growth, Shadow work, Year ahead, Past life, Decision & crossroads, Creativity & self-expression, Travel & new beginnings

---

## Booking form → Parinameh flow

1. Visitor fills form (name, email/phone, date, time, reading type, message)
2. Contact field auto-detects email vs phone (checks for `@`)
3. Form INSERTs into `booking_requests` with status `pending`
4. Edge Function `notify-booking` fires (sends email to Pari)
5. Parinameh auto-imports on load: fetches all `pending`/`accepted` bookings
6. For each booking:
   - If client exists (matched by email, phone, or name) → adds session, appends message to notes
   - If new client → creates client entry with contact info, notes from message, and a session
7. All imported bookings marked `status: 'imported'`

---

## Page view analytics & funnel tracking

**Tracking method:** Direct REST API calls to Supabase (`fetch` to `/rest/v1/page_views`), independent of the Supabase JS client. Uses a reusable `logView(data)` helper function.

**Three events tracked (visitor funnel):**

| Path | When | Purpose |
|---|---|---|
| `/` | Page load | Visitor landed on site |
| `/book-click` | "Book a Reading" button clicked | Opened the booking modal |
| `/book-submit` | Booking form successfully submitted | Completed a booking |

This creates a conversion funnel: visits → opened form → completed booking.

**Country detection:** On page load, fetches country code from `https://api.country.is` (returns 2-letter ISO code). Attached to the page load event. Fire-and-forget — doesn't block rendering.

**Referrer:** Captured from `document.referrer` on all events.

**In Parinameh (overview tab, bottom):**
- Fetches counts for today/this week/this month using `select('id', { count: 'exact', head: true })`
- Aggregates top 5 referrer hostnames from referrer URLs
- Extracts search terms from `q=`/`query=`/`p=` parameters in referrer URLs (Google, Bing, DuckDuckGo, Yahoo)
- Aggregates top 5 countries
- Displays as: `✦ Stats · X today · Y this week · Z this month` with Sources, Search terms, Countries lists below

---

## WebGL aura shader

### Uniforms
| Uniform | Type | Purpose |
|---|---|---|
| `u_resolution` | vec2 | Canvas pixel dimensions |
| `u_time` | float | Elapsed seconds |
| `u_touch` | vec2 | Pointer position (0–1 normalized) |
| `u_touch_active` | float | Touch/hover intensity (0–1) |
| `u_scroll` | float | Movement phase accumulator (drives color shifting) |
| `u_energy` | float | Energy level from movement speed |
| `u_trail[16]` | vec3[] | Trail point positions + intensity |

### Noise functions
- **Simplex noise** (`snoise`): 3D simplex noise, standard implementation
- **FBM** (`fbm`): 5 octaves, frequency multiplier 2.1x per octave, amplitude decay 0.5x
- **Hash** (`hash`): cheap 2D hash for film grain

### Color palette (`auraColor`)
Three colors cycled with a phase parameter:
- c1: `vec3(0.25, 0.05, 0.45)` — deep purple
- c2: `vec3(0.55, 0.08, 0.40)` — magenta/rose
- c3: `vec3(0.15, 0.25, 0.50)` — blue
- Energy boost: `vec3(0.15, 0.08, 0.02) * energy²`

### Coordinate offset
`p.y -= 0.06 * (u_resolution.y / u_resolution.x)` — shifts aura center up to align with the "O" in Oracle. Aspect-responsive: more shift on tall screens (mobile), less on wide (desktop).

### Layer 1 — Deep background nebula
- Scale: `p * 1.2`
- Two FBM samples combined, `smoothstep(-0.1, 0.8, n * n2 + 0.3)`
- Multiplier: 0.4

### Layer 2 — Mid volumetric light
- Animated center: `vec2(sin(t*0.3)*0.15, cos(t*0.25)*0.1)`, blended toward touch position
- Exponential distance falloff: `exp(-dist * 1.5)`
- Boosted by touch influence and energy
- Multiplier: 1.5x

### Layer 3 — Tendrils
- Spiral count: `3.0 + sin(scrollPhase*0.5) * 1.5`
- FBM-perturbed spiral: `sin(angle * tendrilCount - t*1.5 + fbm(...)*4.0)`
- Exponential radius falloff: `exp(-radius * 2.5)`
- Multiplier: 1.8x

### Layer 4 — Sparkles
- High-frequency noise: `p * 4.0`
- Power function for sharp sparkle: `pow(max(n * n2, 0.0), 3.0) * 2.0`
- Color: `vec3(0.7, 0.5, 0.9)` (lavender)
- Boosted by energy: `1.0 + energy * 2.0`

### Layer 5 — Halo
- Exponential glow from center
- Size oscillates with movement phase: `0.8 + sin(scrollPhase*0.3) * 0.3`
- Color oscillates between c1 and c3
- Multiplier: 0.15

### Trail (volumetric light deposits)
- 16 trail points, each with x, y, intensity
- Distance cull at 0.7 (skip if too far)
- Single snoise per point (not FBM, for performance)
- Three falloff types: glow (`exp(-d*8.0)`), scatter (`exp(-d*3.0)*0.25`), ambient (`exp(-d*1.5)*0.08`)
- Warm core highlight: `vec3(0.9, 0.8, 1.0) * exp(-d*12.0)`

### Energy burst
- Radial glow from center, scaled by energy level
- `energy * exp(-burstRadius * 1.5) * 0.4`
- snoise perturbation for organic feel

### Post-processing
- Vignette: `smoothstep(0.4, 1.4, length(p))`
- Tonemap: `color / (color + 0.5)`
- Color grading: red × 1.05, blue × 1.02, gamma 0.9
- Film grain: hash-based, amplitude 0.02

### Adaptive resolution
- DPR capped at 1.5
- Scale factor: mobile (<480px) 0.35, tablet (<768px) 0.45, desktop 0.55
- CSS scales up the low-res canvas — dark/glowy scenes hide the scaling perfectly

---

## Interaction system

### Movement tracking
- Both mousemove and touchmove update pointer position
- Position interpolation per frame: `0.15` lerp factor (responsive tracking)
- Mouse hover also activates aura glow (not just click/touch)

### Energy
- Computed from pointer movement speed: `sqrt(dx² + dy²) * energyMult`
- Desktop multiplier: 4.0
- Touch multiplier: 8.0 (touch events fire less frequently)
- Energy cap: 1.5
- Energy decay per frame: `× 0.97`
- Minimum threshold: 0.001

### Movement phase (swirl speed)
- Accumulated from movement speed: `speed * 200.0`
- Drives scrollPhase in shader (uniform `u_scroll * 0.002`)
- Faster movement = faster color cycling, tendril morphing, halo oscillation

### Trail deposits
- Desktop: every 3rd frame
- Touch: every frame
- Intensity: 1.0 if pointer down, 0.5 on hover
- Trail decay per frame: `× 0.985`

### Touch active
- Desktop interpolation: `0.012` (slow fade)
- Touch interpolation: `0.06` (faster response)
- Desktop auto-decay when mouse stops: `targetTouchActive *= 0.96`
- On mouse move: `targetTouchActive = min(targetTouchActive + 0.15, 1.0)`

### "You were meant to find this" reveal
- Accumulates `energyLevel * 0.01` per frame (desktop) or `* 0.03` (touch)
- Triggers at threshold 8.0
- Fades in with `transition: opacity 1.5s ease`
- One-time — doesn't hide again

---

## Mobile lockdown

Prevents all scroll, bounce, pull-to-refresh, and text selection on the main page.

**CSS (html):** `height: 100%; overflow: hidden; overscroll-behavior: none; position: fixed; width: 100%`

**CSS (body):** `height: 100vh; overflow: hidden; overscroll-behavior: none; position: fixed; -webkit-user-select: none; user-select: none; -webkit-touch-callout: none; -webkit-tap-highlight-color: transparent`

**JS:** `document.addEventListener('touchmove', e => { if (!e.target.closest('.booking-modal')) e.preventDefault(); }, { passive: false })` — blocks touchmove everywhere except inside the booking modal form.

**Form inputs exception:** `.booking-modal input, .booking-modal textarea { -webkit-user-select: text; user-select: text; }` — re-enables text selection in form fields only.

---

## Booking modal

### Layout
- **Overlay**: fixed fullscreen, `background: rgba(0,0,0,0.6)`, centers the modal
- **Wrapper** (`.modal-wrap`): `max-width: 400px`, `width: calc(100% - 32px)`, holds the shimmer border pseudo-element
- **Modal** (`.booking-modal`): `border-radius: 24px`, `padding: 28px 24px 24px`, `background: rgba(12,8,20,0.88)`, `backdrop-filter: blur(60px) saturate(1.4)`, `max-height: 88vh`, scrollable

### Opening animation
- Wrapper: `scale(0.92) translateY(20px)` → `scale(1) translateY(0)` with `0.5s cubic-bezier(0.16,1,0.3,1)`
- Opacity: `0` → `1` with `0.4s ease`

### Shimmer border
- On `.modal-wrap::before`, not on the modal (avoids overflow clipping)
- `border-radius: 25px`, `inset: -1px`, `padding: 1px`
- Animated conic gradient: `rgba(200,160,255,0.35)` and `rgba(140,180,255,0.25)` highlights
- Uses `@property --shimmer-angle` for smooth CSS custom property animation
- Rotation: `4s linear infinite`
- Mask technique: `mask-composite: exclude` creates the border-only effect

### Floating particles
- 24 particles inside `.modal-particles`
- Size: 1.5–3.5px with purple glow
- Animation: drift upward (`particleFloat`), varying duration (3–7s), staggered delays

### Field entrance stagger
- Each `.field` starts at `opacity: 0; translateY(12px)`
- Transitions in with cascading delays: 0.08s, 0.13s, 0.18s, 0.23s, 0.28s, 0.33s, 0.38s, 0.43s

### Form fields
1. **Name** — `<input type="text">`, required
2. **Email or phone** — `<input type="text">`, required, auto-detects email vs phone by `@`
3. **Preferred date/time** — row layout, `<input type="date">` + `<select>` with 15-min slots 08:00–22:00
4. **Type of reading** — `<select>` with 12 options, marked optional
5. **Message** — `<textarea>`, marked optional
6. **Submit button** — pill-shaped, gradient background, sparkles

### Input styling
- Background: `rgba(255,255,255,0.06)`, border: `rgba(255,255,255,0.1)`
- Focus: border `rgba(180,140,255,0.35)`, background `rgba(255,255,255,0.09)`, glow `0 0 20px rgba(160,120,255,0.1)`
- Border-radius: 12px, padding: 9px 12px
- Labels: `rgba(255,255,255,0.55)`, 0.75rem, capitalised words (not uppercase)
- Placeholders: `rgba(255,255,255,0.3)`
- Date input: `color-scheme: dark; min-height: 40px`

### Submit button
- `border-radius: 980px` (fully round pill)
- `background: linear-gradient(135deg, rgba(160,120,255,0.1), rgba(120,160,255,0.08))`
- `border: 1px solid rgba(200,160,255,0.15)`
- Color: `rgba(255,255,255,0.85)`
- Hover glow pseudo-element expands from center
- **Sparkles**: 30 on touch devices, 16 on desktop. Size 1.5–4px, cycle 0.8–2.3s, peak opacity 0.5–1.0, purple glow

### Dismissal behavior
- X button (`.close-btn`): always works, closes and resets form
- Tap outside overlay: only works after successful booking submission (`bookingSubmitted` flag)

### Validation
- Name required
- Email or phone required (shows contextual error message in button text for 2s)

---

## Thank-you celebration

Shown after successful booking submission. Stays visible until user taps outside or clicks X.

### Personalised heading
`"Thank you [first name]"` — extracts first word from the name field.

### Glowing orb (`.ty-orb`)
- Size: 80×80px
- Entrance: `scale(0.3)` → `scale(1)` over 1s with 0.2s delay
- **Ambient glow** (`::before`): `inset: -20px`, radial gradient `rgba(180,140,255,0.25)`, breathes between `scale(1)` and `scale(1.15)` over 3s
- **Specular highlight** (`::after`): radial gradient with offset highlight at 40% 35%, box-shadow pulses between 40px/80px and 60px/100px glow over 2s
- Both pseudo-elements at `z-index: 2` (orbit dots pass behind and in front)

### Live aura canvas inside orb
- 128×128 WebGL canvas, clipped to circle via CSS `border-radius: 50%`
- Simplified shader with 2 snoise layers (scales 0.8x and 0.6x, speeds 0.12 and 0.09)
- Darker palette: c1 `(0.18,0.04,0.35)`, c2 `(0.35,0.06,0.30)`, c3 `(0.10,0.18,0.38)`
- Subtle sparkle: `vec3(0.5,0.35,0.7) * pow(max(n1*n2,0.0),2.0) * 0.3`
- Edge falloff: `smoothstep(1.0,0.3,r) * 0.7`, alpha: `smoothstep(1.0,0.8,r) * 0.85`
- Transparent background, alpha blending enabled

### 3D orbiting dots
- **Count**: 7 dots
- **3D orbital mechanics**: each dot orbits on its own tilted plane around the orb, simulating electrons around a nucleus
- **Per-dot parameters**:
  - Phase: `(i/7) * 2π + random * 0.5`
  - Speed: 40% chance of fast (0.8–1.5), 60% slow (0.25–0.55)
  - Radius: 46–56px
  - Tilt X: 0.4–1.6 radians (23°–92°, polar to equatorial)
  - Tilt Z: 0–2π (full rotation spread)
  - Wobble speed: 0.1–0.3, wobble amount: 0.15–0.4
- **Depth simulation**: dots in front (z > 0) get `z-index: 3`, full opacity (0.4–0.9), glow shadow. Dots behind (z < 0) get `z-index: 1`, dimmed (0.15–0.3), no glow.
- **Wobble**: tilt angles drift with `sin(t * wobbleSpeed) * wobbleAmt`, creating organic non-circular paths
- Animated via requestAnimationFrame, cleaned up on modal close

### Burst sparks (`.ty-spark`)
- **Count**: 30
- Originate from center of orb (modal width/2, 100px from top)
- Fly outward in random directions, distance 60–180px
- Size: 2–5px, purple tones with glow
- Duration: 0.8–1.6s, delay: 0–0.3s
- Animation: appear at 15% of timeline, fade out by 100% while shrinking to 30%

### Floating embers (`.ty-ember`)
- **Count**: 14
- Position: random across bottom 5–25% of container
- Drift upward 60–160px continuously (infinite animation)
- Size: 1.5–4px, purple glow
- Duration: 2.5–5.5s, delay: 0.5–2.5s
- Fade in at 20%, sustain to 80%, fade out

### Text entrance
- "Thank you [name]": fades in at 0.6s delay
- Message + note: fades in at 0.9s delay
- Both: `translateY(15px)` → `translateY(0)` over 0.7s
- Message: `rgba(255,255,255,0.4)`, 0.95rem, font-weight 500, centered
- Note ("Your booking is not confirmed until we contact you"): `rgba(255,255,255,0.2)`, 0.95rem

---

## SEO

### Meta tags
- **Title**: "Vera Oracle — Tarot Readings & Spiritual Guidance in Richmond, London"
- **Description**: targeting tarot, spiritual guidance, oracle, Richmond, Southwest London, in-person and online
- **Keywords**: 15+ terms covering tarot reader, psychic reading, clairvoyant, energy healing, aura reading, holistic healing across Richmond/London
- **Canonical**: `https://veraoracle.com`
- **Open Graph**: title, description, url, type=website, locale=en_GB
- **Geo**: region=GB-RIC, placename=Richmond upon Thames, London

### JSON-LD structured data
```json
{
  "@type": "LocalBusiness",
  "name": "Vera Oracle",
  "url": "https://veraoracle.com",
  "areaServed": ["Richmond upon Thames", "London"],
  "address": { "addressLocality": "Richmond upon Thames", "addressRegion": "London", "addressCountry": "GB" },
  "serviceType": ["Tarot Reading", "Oracle Card Reading", "Spiritual Guidance", "Intuitive Counseling", "Energy Healing"],
  "knowsAbout": ["Tarot", "Oracle Cards", "Spiritual Guidance", "Intuitive Reading", "Energy Healing", "Chakra Reading", "Aura Reading"]
}
```

### Hidden crawlable content
Screen-reader accessible div (`aria-hidden="false"`, positioned offscreen) with keyword-rich natural copy covering: all reading types, nearby areas (Twickenham, Kingston), service descriptions. Necessary because the visible page is mostly canvas/WebGL.

### Supporting files
- `robots.txt`: allows all crawlers, references sitemap
- `sitemap.xml`: single URL entry for `https://veraoracle.com`

---

## Typography

- **Apple devices**: SF Pro Display / SF Pro Text (system font, `-apple-system`)
- **Windows/Android**: Inter (Google Fonts CDN, weights 400/500/600, preconnected)
- **Fallback chain**: `-apple-system, 'SF Pro Display', 'SF Pro Text', Inter, 'Helvetica Neue', Helvetica, Arial, sans-serif`
- **Title** ("Vera Oracle"): `clamp(3.5rem, 14vw, 9rem)`, weight 600, letter-spacing -0.02em, text-shadow with white and purple glow
- **Book button**: `clamp(0.85rem, 2.5vw, 1rem)`, weight 500, letter-spacing 0.04em, pill border-radius 980px
- **Reveal text**: `clamp(1.2rem, 5vw, 2.5rem)`, weight 600

### Favicon
Inline SVG data URI — purple radial gradient circle (no external file needed):
`data:image/svg+xml,<svg xmlns='...' viewBox='0 0 32 32'><defs><radialGradient id='g'><stop offset='0%' stop-color='#c8a0ff'/><stop offset='100%' stop-color='#402060'/></radialGradient></defs><circle cx='16' cy='16' r='14' fill='url(#g)'/></svg>`

---

## Instagram link

- Position: fixed, bottom 2rem, centered
- SVG: official Instagram glyph, `fill="currentColor"`, `viewBox="0 0 24 24"`, rendered at 30×30px
- Color: `rgba(255,255,255,0.25)`, hover: `rgba(255,255,255,0.6)`
- Link: `https://www.instagram.com/veraoracle/`, `target="_blank"`, `rel="noopener"`
- Fade-in: 3s ease-out with 2.5s delay

---

## Render tools (local only, not deployed)

### render.html — Postcard generator
- 25 color palette variants of the aura shader
- Dimensions: 3000×4200px (127×177.8mm at 600 DPI, portrait)
- Text: "You were meant / to find this." — SF Pro Medium 260px, white, top-left with 300px/280px margin
- Palettes: 25 hand-tuned complementary schemes (purple/rose, indigo/teal, midnight/gold, etc.)
- Each at a different time point and scroll phase for visual variety
- Click to download individual JPGs, or "download all" link

### render-cards.html — Business card generator
- 25 variants matching postcard palettes
- Dimensions: 1654×661px (70×28mm at 600 DPI, landscape)
- Text: "Vera Oracle" — SF Pro Medium 100px, white, centered
- Same download behavior

---

## Supabase migrations

Pushed via temp project linked to ref `odqnbtnbbvqktaqejuxh`:

1. `20260313000000_booking_requests.sql` — initial table + RLS + rate limit trigger
2. `20260313010000_booking_phone.sql` — made email nullable, added phone column, updated rate limit for both
3. `20260315000000_page_views.sql` — page view tracking table + RLS

---

## Development workflow

No build step. Edit `index.html`, push, Vercel auto-deploys.

```bash
# Edit
vim index.html

# Deploy
git add index.html
git commit -m "description"
git push origin main
# Live in ~5 seconds

# Run Supabase migration
cd /tmp && supabase init && supabase link --project-ref odqnbtnbbvqktaqejuxh
# Create migration in supabase/migrations/
supabase db push

# Deploy Edge Function
supabase functions deploy notify-booking --project-ref odqnbtnbbvqktaqejuxh

# Set secrets
supabase secrets set RESEND_API_KEY=re_xxxxx --project-ref odqnbtnbbvqktaqejuxh
```

### Git config
```
user.name = Timo Arnall
user.email = timo.arnall@gmail.com
remote: https://github.com/timoarnall/veraoracle.git
branch: main
```
