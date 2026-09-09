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
