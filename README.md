# OzPAX

OzPAX is a passenger and cargo movement emulator for the VATSIM network. It watches
real flight activity across Australian and Oceania airspace and turns that traffic
into a living population of individual passengers — each one planning a realistic
journey, connecting only through airports the network actually flies to, and
boarding real pilots' aircraft as they go.

**Status:** in development, targeting a **late 2026** launch.

> OzPAX is an independent community project and is not affiliated with or endorsed
> by VATSIM.

## Core concept

Every passenger (PAX) is its own individual with an origin, a destination, and a
plan — not an abstract statistic. A PAX doesn't just teleport from A to B; if no
direct traffic exists between their origin and destination, the system has to work
out a sensible connection, the same way a real passenger would. A PAX travelling
YPAD → YBBN would never be routed via YPDN or YPPH, but might reasonably connect
via YMML or YSSY, because that's what the network actually flies.

Because VATSIM connections are inherently random, OzPAX doesn't guess at
plausibility — it learns it from observed network traffic.

## How it works

### 1. Network data ingestion

OzPAX continuously polls the VATSIM data feed and stores snapshots of every pilot
session (identified by CID + callsign + logon time, so a single connection can be
tracked across updates even as position data changes).

A session is only recorded as a completed flight once it can be confirmed as
**landed** — groundspeed dropping to zero and altitude settling near field
elevation close to the filed arrival airport. If a pilot disconnects mid-flight
(anywhere before that point), the session is discarded and does not count towards
network traffic data.

### 2. Recording scope

- **Inbound** — any flight that lands at an Australian aerodrome, or anywhere
  within a FIR under Oceania vACC / VATPAC jurisdiction, regardless of departure
  point.
- **Outbound** — flights departing Australia toward an overseas destination,
  recorded as long as the pilot doesn't disconnect en route.

Only completed sessions are kept long-term; raw polling snapshots are pruned once
a session resolves.

### 3. The traffic graph

Completed flights build a weighted departure → arrival graph, with edge weight
based on observed frequency (recency-decayed, so routes that go quiet fade out
over time). This graph is the single source of truth for "what's a plausible
route on this network right now" — both for planning itineraries and for
deciding how much demand a given airport should generate.

### 4. Passenger generation

New PAX are generated organically from the traffic graph itself: airports and
routes that see more real traffic produce more demand. There's no external
schedule or seed data — the passenger population is a direct reflection of who's
actually flying.

### 5. Itinerary planning

A PAX's itinerary is computed as a weighted path search across the traffic graph
(favouring well-travelled connections over obscure ones), so routings stay
realistic without needing hand-coded geography rules.

### 6. Boarding

- If a real pilot's flight plan is a **direct match** to a PAX's final
  destination, the PAX boards as soon as that pilot connects.
- If the flight is only a **connecting leg** of a PAX's itinerary, the PAX
  doesn't lock in until that flight is within 10 minutes of its scheduled
  departure — avoiding early commitment to a flight that might not actually go.
- When more qualifying PAX exist than available seats, boarding is decided by
  **weighted random selection**, weighted by how long each PAX has been waiting.

### 7. Seat capacity

Each flight's passenger capacity is capped by the real-world seat count of the
filed aircraft type, looked up from a seeded ICAO-type → seat-count reference
table (with a sane fallback for unrecognised or unusual types).

### 8. Disconnects

If a PAX's boarded flight disconnects before landing, the PAX is returned to
their origin airport and re-enters the waiting pool to be re-planned onto a new
itinerary. Nothing is stranded mid-air.

### 9. Journey timeout & stranding feedback

Every PAX has **one week** from generation to reach their final destination. If a
PAX never manages to leave their departure field in that time, it's logged as a
**stranded** event against that airport. This feeds back into passenger
generation (§4) as a dampening signal — an airport that keeps stranding its PAX
has its future outbound demand reduced, so the population doesn't keep piling up
demand for connections the network never actually provides.

### 10. Maps

- **Passenger map** — airport icons showing waiting PAX counts and a breakdown of
  where they're headed.
- **Live network map** — active flights in transit with their current boarded-PAX
  manifest, for live network stats.

## Tech stack

Built on Laravel, with Tailwind CSS for the frontend.

## Development

Standard Laravel setup:

```bash
composer install
cp .env.example .env
php artisan key:generate
php artisan migrate
npm install && npm run dev
```

This readme.md file was generated with ClaudeAI after consolidating all plans into one location.