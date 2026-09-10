# Connecting Supabase

The CMS was built so that this is a swap, not a rewrite. Everything that reads or
writes data goes through the `Store` object at the top of `admin.html`'s script.
Nothing else in the CMS touches storage.

## What has to happen

1. **Create the project.** You do this — signing up and entering credentials is
   yours, not something to hand off. Note the project URL and the **anon** key.
   The `service_role` key never leaves the Supabase dashboard and must never be
   pasted into a client page or into a chat.
2. **Create the tables** (SQL below).
3. **Set the row level security policies** (also below) — without them the anon
   key can read and write everything.
4. **Point the CMS at it**: add the supabase-js script tag, fill in the URL and
   anon key, implement the `supabase` driver in `Store`, flip `DRIVER` to
   `"supabase"`.
5. **Point the site at it**: `index.html` fetches published rows instead of using
   its inline `PROPERTIES` / `REVIEWS` arrays, keeping the inline arrays as the
   offline fallback.

## Schema

```sql
create table properties (
  id             text primary key,          -- NP-1042
  deal           text not null check (deal in ('rent','sale')),
  title          text not null,
  type           text,                      -- Apartment / Villa / Plot / Commercial / Studio
  config         text,                      -- 3 BHK, Office, Site
  locality       text not null,
  area           text,
  sqft           integer,
  price          integer not null,          -- rupees; monthly rent or asking price
  deposit        integer,                   -- rent only
  furnish        text,
  floor          text,
  parking        text,
  status         text default 'available' check (status in ('available','offer','rented')),
  available_from text,
  khata          text,                      -- sale only
  age            text,                      -- sale only
  art            text default 'apartment',  -- fallback drawing when there are no photos
  photos         text[] default '{}',       -- Storage public URLs
  hidden         boolean default false,     -- kept in the CMS, off the site
  created_at     timestamptz default now(),
  updated_at     timestamptz default now()
);

create table reviews (
  id           uuid primary key default gen_random_uuid(),
  stars        smallint not null check (stars between 1 and 5),
  text         text not null,
  name         text not null,
  detail       text,
  tag          text[] default '{home}',     -- home / rent / buy / management / services
  status       text default 'pending' check (status in ('pending','published','hidden')),
  submitted_at timestamptz default now()
);

create table site (
  key   text primary key,                   -- 'portrait'
  value text                                -- Storage public URL
);
```

Note `available_from` — Postgres convention is snake_case, so the driver maps it
to the `availableFrom` the UI uses. Same for nothing else; the rest already match.

## Row level security

The point of the policies: the public site reads only what is published, the
public can *submit* a review but never publish one, and only a signed-in Ravi can
change anything.

```sql
alter table properties enable row level security;
alter table reviews    enable row level security;
alter table site       enable row level security;

-- anyone may read listings that are not hidden
create policy "public reads live listings" on properties
  for select using (hidden = false);

-- anyone may read published reviews
create policy "public reads published reviews" on reviews
  for select using (status = 'published');

-- anyone may submit a review, but only as pending
create policy "public submits pending reviews" on reviews
  for insert with check (status = 'pending');

-- anyone may read site images
create policy "public reads site" on site for select using (true);

-- signed-in admin does everything
create policy "admin all listings" on properties
  for all using (auth.role() = 'authenticated') with check (auth.role() = 'authenticated');
create policy "admin all reviews" on reviews
  for all using (auth.role() = 'authenticated') with check (auth.role() = 'authenticated');
create policy "admin all site" on site
  for all using (auth.role() = 'authenticated') with check (auth.role() = 'authenticated');
```

Create Ravi's login as a single user in Supabase Auth (email + password), and put
a sign-in gate in front of `admin.html`. Until that gate exists, `admin.html`
should not be published anywhere public with a live database behind it.

## Photos

Create a public Storage bucket `property-photos`. On upload the CMS keeps the same
downscale step it has now (1400px, JPEG q0.75) and then uploads instead of
inlining, storing the public URL in `photos[]`. That removes the browser's ~5 MB
localStorage ceiling, which is the only reason photos are currently capped.

## The driver

In `admin.html`, replace the commented block in `Store` with:

```js
const sb = supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

const remote = {
  label: "Supabase",
  properties: {
    async list(){
      const {data, error} = await sb.from("properties").select("*").order("created_at", {ascending:false});
      if(error) throw error;
      return data.map(fromRow);
    },
    async save(p){
      const {error} = await sb.from("properties").upsert(toRow(p));
      if(error) throw error;
      return p;
    },
    async remove(id){
      const {error} = await sb.from("properties").delete().eq("id", id);
      if(error) throw error;
    }
  },
  // reviews and site follow the same shape
};
```

`toRow` / `fromRow` only need to swap `availableFrom` ↔ `available_from`.

Then `DRIVER = "supabase"` and the CMS is live. The Publish button becomes
unnecessary at that point, because the site reads the same tables.
