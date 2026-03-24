# Vera Oracle

A tarot reading booking website for Parinameh — a single-page static site with a procedural WebGL aura background, integrated booking system, and visitor analytics.

Live at [veraoracle.com](https://veraoracle.com)

## What it does
- Full-screen animated aura (WebGL shaders: simplex noise, FBM, spiral tendrils, sparkles, pointer-reactive energy)
- Booking modal with glassmorphism, floating particles, 12 reading types
- Personalised thank-you screen with 3D orbiting orb and burst particles
- Analytics: page views, booking funnel, traffic sources, search terms, geographic data
- Email notifications via Supabase Edge Functions + Resend

## Tools
- `orb-compare.html` — 6 shader variations for the thank-you orb (R&D)
- `render.html` — 25 postcard variants at 600 DPI for print
- `render-cards.html` — 25 business card variants at 600 DPI for print

## Tech
Zero dependencies — pure HTML/CSS/JS with inline WebGL shaders. Supabase for bookings + analytics. Deployed on Vercel (auto-deploy from GitHub, no build step).
