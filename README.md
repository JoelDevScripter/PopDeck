PopDeck — README (English)

PopDeck is a collectible card Discord bot (backend in Go) that mints music/artist cards, lets users open packs, trade, sell, and collect eras and rarities. This README documents the full project scope, architecture, DB schemas, commands, integration details, admin flows, storage, monetization plans, and a week-by-week development roadmap so you — or future collaborators — can rebuild the project from scratch.

Table of contents

High-level summary

Goals & features

Architecture and folders

Database schemas (recommended SQL)

Image & storage strategy (URLs, CDN, Discord, S3)

Artist / Era / Image ingestion & heuristics (keywords)

Card model: templates vs user-owned copies

Rarety business rules & classifier logic

Commands list (slash & prefix) — full

Admin commands & admin.json format

Economy & monetization (coins, multiple currencies, real money flows)

Cooldowns, memberships, anti-abuse & rate limiting

Events, packs, marketplace, transfers

Dev setup & step-by-step: run locally and seed DB

Reset / reinitialise DB safely (SQL)

How to propagate a rarity change to users' holdings

Roadmap & weekly detailed schedule (full)

Notes on security, moderation and legal

Appendix: example env, admin.json, SQL snippets, prompts (short)

High-level summary

PopDeck is a Discord bot written in Go. Users open packs and draw collectible artist cards. Cards are built from real images (cover photos, photoshoots, concert shots, MV stills) associated with an artist and an era (release / promotion window). Each card has a rarity and is rendered into a final image (PNG) with a rarity frame.

Important design principles:

Keep the DB for metadata and URLs, not binary images.

Use a remote object store (S3 / Supabase / MinIO) or Discord message attachments + CDN for final rendered PNGs.

Use reproducible card templates so multiple users can own the same template_id (same card artwork/ID).

Provide robust admin tools to correct metadata, remove bad images, and change rarity (propagating to owners).

Design for scale: caching (Redis), worker pools for image downloads/composition, background sync jobs.

Goals & features

Minimum Viable Product (MVP):

/openpack or !openpack gives 3 cards; user keeps one (choose one of the three). Each card comes as a flip interaction (back -> front).

/mycards or !mycards list the user's cards (paginated).

Initial ingestion of Top Pop artists and their eras (Spotify/Last.fm/MusicBrainz).

Cards stored as templates and user inventory referencing templates.

Basic economy (Notes) and /daily.

Admin-only commands for ingest, set rarity, remove cards.

Full planned feature set (long-term):

Multiple currencies: Notes (freeish), Discs (mid), Gems/Gold (premium). All or some can be purchasable with real money.

Packs with configurable drop rates, event-limited packs, premium packs.

Marketplace: server-only listings and global trade; purchases use in-game currencies.

Trading flow with escrow & confirmations.

Events, leaderboards (per-server + global), missions (daily/weekly), achievements.

Background sync (weekly) to expand artist base via related artists and top charts.

Image classifier & heuristics to assign image type (concert, promo, MV, cover) and map to rarity.

Admin panel/commands and a job system for pre-rendering and pre-caching images to cut latency.

Architecture and folders

Recommended repo layout (you already use this — keep it):

project-root/
│── cmd/
│   └── bot/
│       └── main.go           # app entrypoint
│
│── internal/
│   ├── commands/             # discord command handlers (slash + prefix)
│   ├── cards/                # card generation & pack logic
│   ├── artists/              # ingestion from Last.fm/Spotify/MusicBrainz
│   ├── eras/                 # era resolution, MusicBrainz interactions
│   ├── images/               # search, download, classifier, dedupe
│   ├── render/               # rendering pipeline (fogleman/gg)
│   ├── economy/              # coins, wallets, transactions
│   ├── market/               # marketplace & sales logic
│   ├── events/               # event & pack definitions
│   └── jobs/                 # cron and workers for background tasks
│
│── pkg/
│   ├── db/                   # db connection & migration helpers
│   └── logger/               # structured logging
│
│── assets/                    # frames, fonts, templates for cards
│
│── db/                       # SQL migrations
│
│── .env                      # secrets and keys (never commit)
│── admin.json                # list of admin Discord IDs (local file)
│── go.mod
│── README.md                 # you are reading it


File naming guidelines:

Each command: internal/commands/<name>.go (e.g. openpack.go, mycards.go, draw.go).

Cards logic: internal/cards/manager.go, draw.go, packs.go, render.go.

Artists ingestion: internal/artists/lastfm.go, spotify.go, musicbrainz.go.

Image classification: internal/images/classifier.go.

Migration files: db/migrations/000_init.sql, 001_more_tables.sql, etc.

Database schemas (recommended SQL)

This schema is designed for the "template" approach: card_templates store the canonical card (image, rarity, era, artist); user_cards link users to template IDs. This allows multiple users to own copies of the same card template.

Run migrations with your preferred tool (psql, pgx, migrate, golang-migrate). Below are example CREATE TABLE statements.

-- users (discord)
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  discord_id TEXT UNIQUE NOT NULL,
  coins BIGINT DEFAULT 0,          -- Notes (free)
  discs BIGINT DEFAULT 0,          -- premium mid-tier currency
  gems BIGINT DEFAULT 0,           -- premium top-tier currency
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  last_daily TIMESTAMP WITH TIME ZONE
);

-- artists + groups + artist_membership (for groups vs individuals)
CREATE TABLE artists (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  spotify_id TEXT,
  mbid TEXT,
  is_group BOOLEAN DEFAULT false,
  metadata JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

CREATE TABLE artist_members (
  id SERIAL PRIMARY KEY,
  artist_id INT REFERENCES artists(id) ON DELETE CASCADE,
  member_artist_id INT REFERENCES artists(id) ON DELETE CASCADE,
  role TEXT
);

-- eras (release groups / promo windows)
CREATE TABLE eras (
  id SERIAL PRIMARY KEY,
  artist_id INT REFERENCES artists(id) NOT NULL,
  name TEXT NOT NULL,
  start_date DATE,
  end_date DATE,
  search_keywords TEXT[],    -- seeds used to search images
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- raw images for era (stored source metadata + storage URL)
CREATE TABLE era_images (
  id BIGSERIAL PRIMARY KEY,
  era_id INT REFERENCES eras(id),
  source_url TEXT,
  storage_url TEXT,       -- final stored raw file (S3, Supabase, or discord message URL)
  kind TEXT,              -- 'concert'|'mv'|'promo'|'cover' etc.
  width INT,
  height INT,
  phash TEXT,             -- perceptual hash for dedupe
  license JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- card templates (final rendered templates or canonical card files)
CREATE TABLE card_templates (
  id BIGSERIAL PRIMARY KEY,
  public_id TEXT UNIQUE,      -- visible id TMC-1989-L-00X or pattern AAA_BB#01
  artist_id INT REFERENCES artists(id),
  era_id INT REFERENCES eras(id),
  rarity TEXT CHECK (rarity IN ('common','rare','epic','legendary','mythic')),
  image_url TEXT NOT NULL,    -- final rendered PNG (S3/CDN/discord)
  source_image_id BIGINT REFERENCES era_images(id),
  copies_created BIGINT DEFAULT 0,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- user inventory (each row = ownership of one copy)
CREATE TABLE user_cards (
  id BIGSERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id) NOT NULL,
  template_id BIGINT REFERENCES card_templates(id) NOT NULL,
  obtained_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  is_locked BOOLEAN DEFAULT false, -- prevents sale/trade
  is_temporary BOOLEAN DEFAULT false,
  expires_at TIMESTAMP WITH TIME ZONE NULL
);

-- transactions / economy log
CREATE TABLE transactions (
  id BIGSERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id),
  type TEXT, -- 'daily','buy','sell','market_fee','purchase_real'
  amount BIGINT,
  currency TEXT, -- 'notes','discs','gems'
  meta JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- marketplace (server-limited listings)
CREATE TABLE marketplace_listings (
  id BIGSERIAL PRIMARY KEY,
  seller_user_id INT REFERENCES users(id),
  template_id BIGINT REFERENCES card_templates(id),
  price BIGINT,
  currency TEXT, -- 'notes','discs','gems'
  server_id TEXT, -- guild id (for server-limited listings)
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  expires_at TIMESTAMP WITH TIME ZONE NULL,
  is_active BOOLEAN DEFAULT true
);

-- events / temporary cards
CREATE TABLE events (
  id SERIAL PRIMARY KEY,
  name TEXT,
  description TEXT,
  start_at TIMESTAMP WITH TIME ZONE,
  end_at TIMESTAMP WITH TIME ZONE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- temporary (event-only) cards (can be created by admins)
CREATE TABLE temp_card_templates (
  id BIGSERIAL PRIMARY KEY,
  public_id TEXT UNIQUE,
  artist_id INT,
  era_id INT,
  rarity TEXT,
  image_url TEXT,
  event_id INT REFERENCES events(id),
  created_by INT, -- admin user id
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  expires_at TIMESTAMP WITH TIME ZONE
);


Indexes: create indices on user_cards(user_id), card_templates(artist_id, rarity), era_images(phash) for fast lookup.

Image & storage strategy (URLs, CDN, Discord, S3)

Principles

Store metadata in DB; store actual files in object storage or Discord attachments (and serve via CDN). Keep the storage_url / image_url in DB.

Prefer S3-compatible storage (AWS S3, Backblaze, Supabase Storage, MinIO). Configure a CDN in front (Cloudflare, Cloudfront) for global, fast serving.

Fields

era_images.storage_url = raw downloaded image (S3 path).

card_templates.image_url = final PNG (S3/CDN/discord message URL), used directly in Discord embeds.

Why not store binary in DB?

DB grows quickly and becomes expensive / slow. Keep DB metadata-only.

Option: Discord as storage

For fast prototyping you can upload composed PNG to a dedicated Discord channel (bot uploads there) and store the message attachment URL. Pros: no extra infra. Cons: fragile, rate limits, and not ideal long-term.

CDN

Place a CDN in front of S3 bucket URLs. Store CDN URL as image_url (e.g., https://cdn.mydomain.com/cards/xxx.png). If using Supabase, use their CDN or proxy through Cloudflare.

Artist / Era / Image ingestion & heuristics (keywords)

When creating an era, generate search keywords to find era-specific images:

Example keywords (seed):

"{artist} {era_name} concert"
"{artist} {era_name} live"
"{artist} {era_name} photoshoot"
"{artist} {era_name} mv"
"{artist} {era_name} red carpet"
"{artist} {era_name} press photo"


Heuristics to associate image → era

If image source (Flickr / Wikimedia) includes a date metadata and that date falls within era's start/end window → assign era.

If image source page contains the album/era text or mentions the release → strong match.

If not, fallback to the era used in the search query (the era you searched for).

Use phash and reverse image match to avoid duplicates.

Image type → suggested rarity mapping

concert → Common or Rare (depending on closeness, stage shot quality)

cover (official album cover) → Rare

mv (music video still) → Rare / Epic

red carpet / awards / iconic promotional shot → Epic / Legendary

promo/iconic → Epic / Legendary

Quality filters

Minimum resolution (e.g., width >= 800).

Non-watermarked (heuristic).

Check license metadata if available.

Classifier

Build a small classifier that tags images (concert, mv, promo, cover) using:

URL patterns (youtube thumbnails, flickr tags),

Alt text / page text,

Simple image heuristics (presence of stage, crowd, or red carpets via color & face detectors).

Card model: templates vs user-owned copies

card_templates represent canonical card items (artwork + public_id + rarity). Multiple users can own the same template_id.

user_cards reference template_id. This supports counting copies (copies_created), market listings referencing specific template copies, and mass updates (if rarity changes, update card_templates.rarity and the listing will reflect it).

When a card is generated freshly (rendered), you may either:

create a card_template and then give the user a user_cards row referencing it, or

reuse an existing card_template (increase copies_created and add user_cards reference).

Public ID format
User requested: TEXT(4)_TEXT(2)#(2digits) e.g. ABCD_XY#01 — you can generate an ID scheme like <ART4>_<ER2>#<NN> where <ART4> is an artist code, <ER2> era code, and NN a two-digit counter. Ensure public_id uniqueness.

Rarety business rules & classifier logic

Rarities: common, rare, epic, legendary, mythic

Base probabilities (configurable)

default: common 62%, rare 23%, epic 10%, legendary 4%, mythic 1%

Rarity assignment flow

Determine base rarity by sampling the distribution.

Reweight according to image kind (e.g., red carpet increases weight to epic/legendary).

Query era_images stock for (era, rarity). Use re-use policy:

if stock < min_stock_per_rarity → create (download a fresh image)

else sample Bernoulli with reuse_bias to decide reuse vs create

When creating new, run classifier to map image type to a likely rarity and optionally override the base random draw.

Manual override

Admins can change rarity for a template via /setrarity. The change affects card_templates.rarity and will reflect in /mycards outputs (no per-user copy changes required because user references the template).

Commands list (slash & prefix) — full

All commands are registered as slash commands; many have prefix equivalents (e.g. !openpack). Slash commands are preferred for discoverability.

Player-facing commands

/openpack (or !openpack) — open a pack (default 3 cards). Returns three card backs; player picks one to keep. (Cooldown, see below.)

/draw (or !draw) — draw a single random card (could be paywalled).

/mycards (or !mycards) — list cards owned by user (paginated).

/inventory [@user] — same as mycards but view other's inventory (depending permissions).

/viewcard <public_id> — show full card info (rendered image, artist, era, rarity, copies owned).

/profile (or !profile) — user profile & balances.

/balance [@user] (or !balance) — show currency balances.

/daily (or !daily) — claim daily Notes.

/shop — open shop to buy packs or currencies.

/market — show marketplace listings (with filters).

/sell <card_instance_id> <price> <currency> (or !sell) — list a card for sale (server-only if desired).

/buy <listing_id> — buy a listed card.

/trade @user <template_id> — initiate trade.

/transfer_data <target_discord_id> — transfer your account data to another account (admin-approved flow or user-initiated if target account empty).

/leaderboard [global|server] — top collectors / coins.

/events — show active events and event packs.

/missions — show daily/weekly missions and progress.

/claim_event_pack <pack_id> — claim event reward if eligible.

/cooldowns — show your cooldowns.

/help — help text.

Admin-only commands (must check admin.json OR DB permissions)

/admin ingest artist <artist_name> — force ingest artist + eras/images.

/admin sync — run weekly sync jobs immediately.

/admin setrarity <template_public_id> <rarity> — set rarity for a card template; broadcasts update to owners.

/admin delete_template <template_public_id> [refund] — remove template and all instances; optionally refund sellers/owners (refund logic documented below).

/admin create_temp_card — create event-only card template.

/admin give_currency <user> <amount> <currency> — grant currency to user.

/admin clear_db — dangerous: wipes tables / resets; should be disabled by default.

/admin purge_artist <artist_id> — remove an artist and related data (with checks).

Bot owner commands (hard-coded IDs in admin.json)

Commands above plus shutdown, reseed, force_render_job.

Admin commands & admin.json format

admin.json (local file) example:

{
  "admins": [
    "123456789012345678",  // your discord id
    "987654321098765432"
  ]
}


The bot reads this file (or a DB table) on startup. Admin commands check whether i.Member.User.ID is in that list.

For deploy flexibility, prefer storing admin list in DB or in an environment-backed secret; but admin.json is fine for local dev and private repos.

Economy & monetization

Currencies

Notes (free, common): awarded from daily, some commands, selling basic cards.

Discs (mid): harder to earn, used in special packs and marketplace.

Gems / Gold (premium): very rare, purchasable with real money, used for premium packs, event packs, large discounts.

Monetization channels

Stripe / PayPal integration (server backend) to let users buy Gems. Provide /buypack and store transactions in transactions table.

Ko-Fi / Patreon perks (manual grants or API).

Boost-based perks (server boosters get discounts / extra draws).

Pricing & UX

Free daily draws / limited free draws per day (e.g., 3 free pack openings per day).

Additional openings cost Notes or Discs. Premium discount for Gems.

Market fees: 5% sale fee (to server/bot bank).

Real-money purchases should map to Gems/Gold; never sell cards directly (sell currencies instead).

Multiple currency uses

Notes for common actions, small trades.

Discs for mid-tier packs and event storefront.

Gems for premium limited-time packs and major discounts.

Anti pay-to-win

Limit premium advantages to convenience and cosmetic boosts: smaller cooldowns, a few extra guaranteed premium drops, not absolute guaranteed legendaries.

Maintain free acquisition routes for premium currency (very rare missions/events) so paying is attractive but not strictly required.

Cooldowns, memberships & anti-abuse

Cooldowns

openpack default cooldown: 7 minutes (user-level).

draw default: 2 minutes.

order (single paid draw) default: 2 minutes.

Admins & special membership tiers reduce cooldown (configurable; e.g., booster tiers: -50%).

Persist cooldowns in Redis with TTL or in Postgres user_cooldowns(user_id, command, expires_at).

Rate limiting & concurrency

Use Redis to throttle per-user and global concurrency.

Worker pool for image downloads (limit concurrency to avoid API throttling).

Anti-abuse

Detect alt-accounts via IP/behavior patterns; ban/mute using admin enforcement rules.

Prevent mass-gifting exploit: track gifts per day and flag accounts with suspicious patterns.

Events, packs, marketplace, transfers

Events

Events can be global (bot server) or server-specific. Event cards are temporary (use temp_card_templates), and expire after event end.

Events can have special drop rates: e.g., summer event +20% legendary on event packs.

Packs

Packs are config-driven. Example:

standard-3 (3 cards): cost 0 Notes (free daily allotment) or 200 Notes / 1 Disc.

premium-5 : higher legendary chance, costs Discs or Gems.

Marketplace logic

Server-only listings by default; global marketplace optional.

When buyer buys from listing: transfer user_cards row -> new user_id, set is_active=false in listing, create transactions log, charge marketplace fee.

If listing expired or seller revoked listing -> show errors.

Transfer data

/transfer_data <target>: user-initiated transfer allowed only if target has no account (or user confirms merge behavior).

If target has existing data and both accounts consent, offer merge or replace. Cost / checks apply.

Dev setup & step-by-step: run locally and seed DB

Install Go (>= 1.20 recommended). go env GOPATH configured.

PostgreSQL local instance (or Docker container).

Clone repo, cd project-root.

Copy .env.example to .env and populate:

DATABASE_URL=postgres://user:pass@localhost:5432/popdeck
DISCORD_TOKEN=your_discord_token
LASTFM_API_KEY=...
SPOTIFY_CLIENT_ID=...
SPOTIFY_CLIENT_SECRET=...
S3_ENDPOINT=...
S3_BUCKET=...
S3_KEY=...
S3_SECRET=...
CDN_URL=https://cdn.example.com


Run DB migrations:

psql $DATABASE_URL -f db/migrations/000_init.sql

Run seed command:

go run cmd/seed/main.go (this should ingest top artists and seed card templates or era_images).

Run bot locally:

go run cmd/bot/main.go (or use air/reflex for live reload).

Reset / reinitialise DB safely (SQL)

When you want to wipe and start from zero:

-- WARNING: destructive. Run on dev only or after backup.
DROP SCHEMA public CASCADE;
CREATE SCHEMA public;

-- Recreate migration tables, then run migrations files in db/migrations...


Or drop specific tables:

DROP TABLE IF EXISTS user_cards, card_templates, era_images, eras, artists, users, transactions, marketplace_listings, events, temp_card_templates CASCADE;
-- Then run the migration SQL to recreate them.


Important: Always back up production DB before destructive operations.

How to propagate a rarity change to users' holdings

If you change card_templates.rarity (via /admin setrarity), the simplest approach to "propagate" is:

Update card_templates row:

UPDATE card_templates SET rarity = $new WHERE public_id = $id;


Because user_cards references the template, any SELECT that reads user_cards JOIN card_templates will reflect the new rarity automatically (no per-user mutation needed).

If you store cached per-user snapshots or precomputed rarity totals, re-compute those aggregates or schedule a background job to update caches.

If you have per-user custom metadata (e.g., stored displayed rarity inside a JSON blob), you must run an update job:

UPDATE user_cards uc
SET meta = jsonb_set(uc.meta, '{rarity}', to_jsonb(ct.rarity))
FROM card_templates ct
WHERE uc.template_id = ct.id AND ct.public_id = $id;


Also notify affected users via DM or server announcement, and update marketplace listings.

Roadmap & weekly detailed schedule (full)

You asked for an absolutely detailed week-by-week plan to rebuild from scratch. Below is a 12-week plan with specific deliverables per week. You said your weekly availability is ~1h weekdays and 2–3h weekends; this schedule assumes ~8–10h/week and prioritizes core features first.

Week 0 (prep) — create repo, set up .env template, decide infra (Postgres, S3/Supabase, Redis).
(If you want to compress weeks, combine work.)

Week 1 — Project skeleton & basic bot

Create repository and branch structure.

Implement cmd/bot/main.go bootstrapping:

Load .env, connect to DB (pkg/db), logger.

Create admin.json loader.

Initialize Discord session and register handlers.

Add /ping and !ping.

Add README (this file).

Tests: lint & gofmt.

Deliverables: repo skeleton, ping command, README.

Week 2 — DB & user registration

Implement PostgreSQL migrations (db/migrations/000_init.sql) with users table.

pkg/db with pool connection using DATABASE_URL.

Implement /profile and /register (auto-register on /profile).

Implement GetUserCoins helper.

Deliverables: migrations, user registration, profile embed.

Week 3 — Economy basics

Add transactions table and wallet helpers.

Implement /daily with cooldown (24h) and reward.

Implement !daily prefix.

Add balance command.

Logging & transactions insert.

Deliverables: economy core, transactions.

Week 4 — Ingest artists & eras (MVP)

Implement internal/artists/lastfm.go to fetch Top Pop artists (using your Last.fm API key).

Implement internal/eras/musicbrainz.go to resolve release-groups (eras) for an artist.

Migrations: artists, eras, era_images.

Implement cmd/seed/main.go to ingest top 100 artists and associated eras (seed command).

Provide one-off seed run instructions.

Deliverables: DB seeded with artists & eras.

Week 5 — Image ingestion & templates

Implement image search & downloader wrapper (images/search.go, downloader.go) using Bing or Last.fm images.

Implement era_images creation + phash dedupe.

Implement card_templates table and pipeline to create one sample template per era using album cover.

Render simple card template (overlay frame + text) with internal/render/compose.go.

Deliverables: era_images and card_templates seeded with cover-based card(s).

Week 6 — Packs & pack-opening flow

Implement internal/cards/packs.go to select templates at random with rarities.

Implement /openpack and !openpack:

Returns three backs with an interaction to pick one.

Insert user_cards row on selection.

Add cooldown logic (Redis or DB).

Implement /mycards listing.

Deliverables: opening packs + pick interaction + persisted ownership.

Week 7 — Rarity & classifier, creation vs reuse

Implement rarity policies config (reuse_bias, min_stock_per_rarity).

Implement internal/images/classifier.go to map image type → rarity probabilities.

Update pack logic: if stock low, fetch/build new image; else reuse templates.

Implement admin /admin setrarity command.

Deliverables: rarity policies + admin setrarity.

Week 8 — Marketplace & trading

Create marketplace_listings table.

Implement /sell, /buy flows with checks (ownership, locked).

Implement trade flow (request, accept, confirm).

Implement transactions logs for each sale.

Deliverables: marketplace & trade.

Week 9 — Events & temporary cards

Implement events table and temp_card_templates.

Admin /admin create_temp_card to add event-only card.

UI command /events and /claim_event_pack.

Precache event packs & background job.

Deliverables: events and temp cards.

Week 10 — Leaderboards, pagination & UX improvements

Implement per-server and global leaderboards (/leaderboard).

Improve /mycards pagination and filters (by artist, era, rarity).

Add /viewcard for details and copy count.

Add embed pagination (reactions or buttons).

Deliverables: leaderboards, filters, pagination.

Week 11 — Monetization & premium flows

Implement payments backend (Stripe / PayPal) or provide webhook handlers and transactions insert for real purchases.

Implement shop for buying Gems/Discs, and purchase flows.

Implement booster / membership benefits (cooldown reduction, extra free draws).

Deliverables: payment purchase -> credit user, premium tier benefits.

Week 12 — Hardening, scaling & admin panels

Add cleanup & hygiene jobs: monthly artist pruning (check Spotify monthly listeners threshold).

Add cron job service for weekly ingestion of related artists.

Add Redis caching for API responses & cooldowns.

Add admin command suite: delete_template, purge_artist, force_render_jobs.

Write detailed docs and finalize README.

Deliverables: production readiness, background jobs, docs.

Weeks 13+ — polish and extended features

P2P marketplace improvements, escrow, dispute resolution.

Image quality improvements, pHash dedupe, better classifier.

CDN & S3 integration and migration.

Tests, CI, and deployment to a hosting service (Docker + Kubernetes optional).

Notes on security, moderation and legal

Respect image copyright. Prefer public domain, CC-BY or press images with proper credits. Store license info in era_images.license and make decisions accordingly.

Avoid hotlinking. Download and store sources or use licensed sources.

Sensitive operations (admin delete, refunds) should require confirmation and an audit log.

Appendix — Examples & snippets
.env example
DATABASE_URL=postgres://popdeck:password@127.0.0.1:5432/popdeck
DISCORD_TOKEN=Bot_xxx
LASTFM_API_KEY=xxx
SPOTIFY_CLIENT_ID=xxx
SPOTIFY_CLIENT_SECRET=xxx
S3_ENDPOINT=https://s3.example.com
S3_BUCKET=popdeck
S3_KEY=...
S3_SECRET=...
CDN_URL=https://cdn.popdeck.example
REDIS_URL=redis://localhost:6379

admin.json example
{
  "admins": ["123456789012345678"]
}

Example /admin setrarity SQL (Go/pseudo)
UPDATE card_templates SET rarity = $1 WHERE public_id = $2;
-- Optionally: notify affected users or update cache.

Commands (Full list recap)

Player commands

/openpack, !openpack

/draw

/mycards, !mycards

/inventory [@user]

/viewcard <public_id>

/profile

/balance [@user]

/daily

/shop

/market

/sell

/buy

/trade @user <template_id>

/leaderboard [global|server]

/events

/missions

/claim_event_pack

/cooldowns

/help

/transfer_data <target>

Admin commands

/admin ingest artist <artist_name>

/admin sync

/admin setrarity <public_id> <rarity>

/admin delete_template <public_id> [refund]

/admin create_temp_card

/admin give_currency <user> <amount> <currency>

/admin clear_db (DESCTRUCTIVE — restrict heavily)

/admin purge_artist <artist_id>
