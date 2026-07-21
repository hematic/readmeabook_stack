# ReadMeABook Stack (Synology NAS)

[ReadMeABook](https://github.com/kikootwo/ReadMeABook) — audiobook request and
acquisition engine (search → Prowlarr → qBittorrent/SABnzbd → import into
Audiobookshelf/Plex). Runs on the Synology (synology920, `192.168.1.90`) via
Komodo. Spiritual successor to AudioBookRequest (decommissioned 2026-07-18) —
this one drives the full download/import pipeline instead of just a request form.

| Service | Port | URL |
|---|---|---|
| ReadMeABook | 3030 | https://rmab.apps.hematic.net |

Postgres and Redis run *inside* this single image (not separate containers) —
no multi-service orchestration, unlike some similar tools.

## Path parity — the one thing that must be exact

RMAB's `/downloads` and `/media` must resolve to the **same physical
directories** qBittorrent and Audiobookshelf already use, or downloads will
appear to succeed but silently fail to import.

| RMAB path | Real share | Matches |
|---|---|---|
| `/downloads` | `/volume1/downloads` | qBittorrent's `${DOWNLOADS}` (`/mnt/downloads` on the ThinkCentre, NFS to this same share) |
| `/media` | `/volume1/audiobooks` | Audiobookshelf's `${AUDIOBOOKS}` (`/mnt/audiobooks` on the ThinkCentre, same share) |

Because this container runs directly on the NAS, these are plain local paths —
no NFS mount needed on this side, even though the ThinkCentre reaches the same
directories over the network.

## Environment (`stack.env`)

| Variable | Description |
|---|---|
| `PUID` / `PGID` | Match the NAS's usual values (other 920 stacks use `1024`) |
| `TZ` | Timezone |
| `RMAB_JWT_SECRET` | `openssl rand -hex 32` |
| `RMAB_JWT_REFRESH_SECRET` | `openssl rand -hex 32` |
| `RMAB_CONFIG_ENCRYPTION_KEY` | `openssl rand -hex 32` |

Set these explicitly (rather than letting the app auto-generate them) so a
redeploy doesn't rotate them out from under existing sessions/config.

Rename `rmab` before first deploy if you'd rather a different hostname — it's
baked into `PUBLIC_URL` here, plus Traefik and Cloudflare DNS.

## Traefik

`rmab.apps.hematic.net` → `http://192.168.1.90:3030` in
`traefik_stack/config/dynamic/combined-services.yml` (middlewares:
`default-headers, geoblock` — no forwardAuth; RMAB guards itself via its own
login + OIDC, same shape as paperless/komodo/komga/lubelogger).

## Setup

1. **Deploy** this stack via Komodo on synology920.
2. Visit `https://rmab.apps.hematic.net`, run the first-run **setup wizard** —
   create your local admin account, connect Prowlarr/qBittorrent (their real
   LAN addresses, e.g. `http://192.168.1.126:9696` / `:9107`), and point it at
   the Audiobookshelf library.
3. In the app's **Admin → OAuth/OIDC settings** (unlike other apps in this
   fleet, RMAB's OIDC is configured through its own UI, not env vars), enable
   OIDC. It will display the exact **Redirect URI** to give Pocket ID — copy
   it verbatim; don't guess it from another app's pattern.
4. **Pocket ID** (`id.apps.hematic.net`) → OIDC Clients → Add:
   - Name: `ReadMeABook`, launch URL `https://rmab.apps.hematic.net`
   - Callback URL: **exactly what RMAB's OAuth settings page showed you**
   - Public off, PKCE off (adjust if RMAB's wizard says otherwise), Skip
     Consent **on**
   - **Save**, then **grant a user group** on the client (Pocket ID v2
     restricts every new client by default — no grant means an instant
     `access_denied`, hit repeatedly elsewhere in this fleet)
   - Tick **Email verified** on your Pocket ID user if not already done
   - Copy the Client ID and secret back into RMAB's OAuth settings form
5. Test the OIDC login button.

## Notes

- If RMAB ever needs to match an *existing* local account by email rather
  than create a new one, check its wizard/docs for that toggle — behavior
  wasn't fully documented at the time this was set up; verify empirically.
- Config/Postgres/Redis all live in `/volume1/containers/readmeabook` — local
  NAS disk, fine for a database since this container runs on the NAS itself
  (the "databases never on NFS" rule is about a *different* host reaching
  storage over the network, not applicable here).
