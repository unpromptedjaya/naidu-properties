# Naidu Properties

A single-file landing page for Naidu Properties — Ravi Kumar's property practice in Bengaluru:
rentals, resale, property management and paperwork (khata, registration, EC).

Everything lives in `index.html`: inline styles, inline script, a hash router over the
`home / rent / buy / management / services / contact` routes, a mobile nav panel and a
gallery bottom sheet. No build step and no dependencies.

## Running it

Open `index.html` directly, or serve the folder:

```sh
python3 -m http.server 8791
```

then visit http://localhost:8791.

## The CMS

`admin.html` is a small back office for the site — listings, reviews and the
site photo. Serve the folder and open `/admin.html`.

- **Listings** — add, edit, duplicate and delete properties in the rent and buy
  sections, attach photos, and hide a listing from the site without deleting it.
- **Reviews** — add a review, or publish one a visitor sent in. Reviews carry a
  state (waiting / published / hidden) and the pages they appear on.
- **Site images** — Ravi's portrait, plus a storage meter.

Data currently lives in the browser's own storage, seeded from `properties.json`
and `reviews.json`. **Publish** produces the exact block to paste back into
`index.html`. That interim goes away once Supabase is connected — see
[SUPABASE.md](SUPABASE.md), which carries the schema, the RLS policies and the
driver swap.

Visitors can leave a review from any reviews section on the site. Until the
backend exists it reaches Ravi over WhatsApp, and he puts it live from the CMS.
