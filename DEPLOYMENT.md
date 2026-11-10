# EMOBase deployment runbook

This is the full “empty server → working site” procedure for the three
repositories:

| Repository | What it runs | Default host port |
| --- | --- | --- |
| [`emobase-infra`](https://github.com/EMOBase/emobase-infra) | MySQL and Keycloak for all stacks on this server | `8081` (Keycloak) |
| [`emobase-portal-deploy`](https://github.com/EMOBase/emobase-portal-deploy) | Portal UI, genomics API/workers, Elasticsearch, JBrowse2, BLAST | `9080` |
| [`emobase-deploy`](https://github.com/EMOBase/emobase-deploy/tree/dev) | Single-species UI, its own Elasticsearch and Directus (`dev` branch) | `9090` |

Each repository's own README documents its services, environment variables and
stack-specific maintenance. This guide covers the cross-repository deployment
order and hand-off.

**Run one `emobase-infra` on each server** that hosts the portal and/or species.
It provides one MySQL and one Keycloak for every stack on that server. Deploy
the local infrastructure first, then the portal (if present), then the species
stacks.

Keycloak has its own browser-facing URL (`KEYCLOAK_PUBLIC_URL`), so species
authentication does not depend on the portal. The species UI still reads
genomics, JBrowse2 and BLAST from a portal, so that portal must be reachable
and have a READY release.

Within one server, the portal and species stacks share the local
`emobase-infra` Docker network. Across servers, each server has its own
MySQL/Keycloak; only the portal URL is shared.

No domain or TLS yet? Jump to [Local setup](#local-setup-no-public-server).

```
Server A (portal + species)
  Browser ──► https://portal.example.org (portal :9080)
  Browser ──► https://auth-a.example.org (Keycloak :8081)
  portal/species ──► local MySQL + Keycloak

Server B (species only)
  Browser ──► https://tcas.example.org (species :9090)
  Browser ──► https://auth-b.example.org (Keycloak :8081)
  species ──► local MySQL + Keycloak
  species ──► Server A's portal for genomics/JBrowse2/BLAST
```

## Local setup (no public server)

Use this to try the whole stack on a laptop or desktop before any DNS/TLS
exists. The portal, Keycloak and species sites use `localhost` over plain HTTP
on ports `9080` (portal), `8081` (Keycloak) and `9090` (species). MySQL and
Keycloak run once in `emobase-infra`.

The browser and the containers need different views of the services:

- `PUBLIC_SERVER` / `PORTAL_SERVER` are the browser-facing portal and species
  URLs and can use `localhost`.
- `KEYCLOAK_PUBLIC_URL` is the browser-facing Keycloak URL. It is not the
  portal origin.
- `INTERNAL_KEYCLOAK_ISSUER` is how containers reach Keycloak.
- `PORTAL_INTERNAL_URL` is used by species nginx for the BLAST proxy.
- `INTERNAL_API_*` and `INTERNAL_DIRECTUS_URL` are used for server-side calls.

`init.sh --local` sets the internal Keycloak/portal URLs so containers can
reach them. Docker Desktop provides `host.docker.internal`; on Linux the
compose files map it with `host-gateway`.

Prerequisites: Docker Desktop or Docker Engine, all three deployment
repositories cloned side by side, `python3` for the Keycloak helper scripts,
and ports `8081`/`9080`/`9090` free. On Apple Silicon, enable Rosetta for
`linux/amd64` first, or BLAST will not start (see the portal README).

1. Local infrastructure:

   ```bash
   cd emobase-infra
   ./init.sh --local
   docker compose up -d --wait
   ```

   This writes `PORTAL_PUBLIC_SERVER=http://localhost:9080`,
   `KEYCLOAK_PUBLIC_URL=http://localhost:8081/keycloak` and the shared
   MySQL/Keycloak secrets. Keycloak admin is at
   `http://localhost:8081/keycloak/admin/`.

2. Portal. Its `init.sh` reads the shared secrets and Keycloak URLs from
   `../emobase-infra/.env`:

   ```bash
   cd ../emobase-portal-deploy
   ./init.sh --local
   docker compose --profile '*' pull
   docker compose --profile setup run --rm setup-jbrowse2-web
   docker compose --profile update run --rm migrate-genomics-db
   docker compose --profile update run --rm migrate-genomics-es
   docker compose up -d --wait
   ./scripts/smoke-test.sh
   ```

   Open `http://localhost:9080`.

3. Species site. `--local` uses localhost for the browser, copies the portal
   URL from the sibling checkout and uses the local Keycloak:

   ```bash
   cd ../emobase-deploy
   ./init.sh --local --species Tcas
   ../emobase-infra/scripts/create-database.sh directus_tcas
   # Only for the shared realm; a dedicated realm is configured by init.sh.
   ../emobase-infra/scripts/allow-origin.sh http://localhost:9090
   docker compose run --rm setup-directus
   docker compose up -d --wait
   ```

   This writes approximately:

   ```dotenv
   PUBLIC_SERVER=http://localhost:9090
   PORTAL_SERVER=http://localhost:9080
   PORTAL_INTERNAL_URL=http://host.docker.internal:9080
   KEYCLOAK_PUBLIC_URL=http://localhost:8081/keycloak
   DIRECTUS_DB_DATABASE=directus_tcas
   ```

   Open `http://localhost:9090`. Sign-in redirects to
   `http://localhost:8081/keycloak/...`, not to the portal.

   > **Note**
   >
   > If the portal is not a sibling checkout, pass
   > `--portal-server http://localhost:9080`. The internal URL is derived from
   > that port. Use `--portal-internal-url ...` only when it needs an explicit
   > override:
   >
   > ```bash
   > ./init.sh --local --species Tcas \
   >   --portal-server http://localhost:9080 \
   >   --portal-internal-url http://host.docker.internal:9080
   > ```

### Verify the local setup

Sign in with the seeded `coco` user from `emobase-infra/.env` and confirm that
JBrowse2/BLAST links point at the portal URL. BLAST returns `502` until the
first release has built databases.

### Access from another machine (optional)

For access from another machine on the LAN, use non-local public URLs. Because
Keycloak's public issuer is fixed at import time, the simplest path for a
throwaway local setup is to reset and recreate it with the LAN URLs:

```bash
cd emobase-infra
docker compose down -v
./init.sh \
  --portal-server http://192.168.1.10:9080 \
  --keycloak-public-url http://192.168.1.10:8081/keycloak -f
docker compose up -d --wait

cd ../emobase-portal-deploy
./init.sh --public-server http://192.168.1.10:9080 -f

cd ../emobase-deploy
./init.sh --public-server http://192.168.1.10:9090 \
  --portal-server http://192.168.1.10:9080 \
  --portal-internal-url http://host.docker.internal:9080 \
  --species Tcas -f

cd ../emobase-infra
./scripts/create-database.sh directus_tcas
# Only for the shared realm; a dedicated realm is configured by init.sh.
./scripts/allow-origin.sh http://192.168.1.10:9090
```

Keycloak imports the realm only once. `allow-origin.sh` adds shared-realm
species origins at any time, but changing the portal origin or
`KEYCLOAK_PUBLIC_URL` after the first start requires resetting Keycloak or
editing the realm in the admin console. Dedicated realms are created with their
origin already allowed.

### Reset the local stacks (optional)

To throw the local data away and start over:

```bash
cd emobase-infra && docker compose down -v
cd ../emobase-portal-deploy && docker compose --profile '*' down -v
cd ../emobase-deploy && docker compose --profile '*' down -v
```

## 1. Prepare the server

Requirements:

- Linux server with Docker Engine 24+ and Compose v2
- `git`
- Outbound HTTPS access to `ghcr.io`, Docker Hub and `quay.io`
- Enough RAM/disk for one local MySQL/Keycloak pair, one Elasticsearch per
  stack, and the application containers. Sharing MySQL/Keycloak saves roughly
  2-3 GiB at four species on one server, but 8 GB is only enough for a portal
  plus one or two species; plan 24-32 GB for four to five species on a server.
- DNS names (or one name plus ports) for the portal, each species site and
  Keycloak
- Ports reachable by the browser: `8081` for Keycloak, `9080` for the portal and
  `9090` for the species site when using the built-in services directly
- `python3` on the host for the Keycloak helper scripts

If you use a TLS reverse proxy in front (recommended), let it listen on 80/443,
point the auth host at `127.0.0.1:8081` and keep the portal/species nginx
containers on localhost ports.

Check Docker:

```bash
docker version
docker compose version
```

## 2. Clone all repositories

Use the same parent directory so the helpers can find each other's `.env`
automatically:

```bash
mkdir -p ~/emobase && cd ~/emobase
git clone https://github.com/EMOBase/emobase-infra.git
git clone https://github.com/EMOBase/emobase-portal-deploy.git
git clone -b dev https://github.com/EMOBase/emobase-deploy.git
```

SSH clone URLs work too:

```bash
git clone git@github.com:EMOBase/emobase-infra.git
git clone git@github.com:EMOBase/emobase-portal-deploy.git
git clone -b dev git@github.com:EMOBase/emobase-deploy.git
```

The species deployment changes are on its `dev` branch; portal and infra use
`main`.

The images are public, so no registry login is normally needed. If a package
is private, log in once with a token that has `read:packages`:

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u <github-user> --password-stdin
```

## 3. Choose the public URLs

Decide the URLs **before** the first start. `KEYCLOAK_PUBLIC_URL` becomes the
OIDC issuer; the portal and species URLs become redirect URIs and public links.
Changing them later requires extra work (see Troubleshooting).

Every server needs its own Keycloak URL, and each portal/species site needs its
own origin — a hostname or a port. They serve their UI at `/`, so do not put
them under path prefixes of the same origin.

Two common layouts:

### A. Direct ports (simple, no TLS)

| Site | URL | Env file |
| --- | --- | --- |
| Keycloak | `http://SERVER_IP:8081/keycloak` | `emobase-infra/.env`: `KEYCLOAK_PUBLIC_URL`, `KEYCLOAK_HOST_PORT=8081` |
| Portal | `http://SERVER_IP:9080` | `emobase-infra/.env`: `PORTAL_PUBLIC_SERVER`<br>`emobase-portal-deploy/.env`: `PUBLIC_SERVER`, `HTTP_PORT=9080` |
| Species | `http://SERVER_IP:9090` | `emobase-deploy/.env`: `PUBLIC_SERVER`, `HTTP_PORT=9090` |

Use the server's LAN/public IP or a DNS name for the browser-facing URLs. When
the portal and species stacks run on the same Docker host, also set the species
`PORTAL_INTERNAL_URL` to an internal address such as
`http://host.docker.internal:9080` (or another hostname reachable from the
species stack's default Docker network), so BLAST and SSR calls do not go
through the public URL. `localhost` is fine for browser URLs when the stacks
use the internal URL.

### B. TLS reverse proxy (production)

Example with three DNS names:

| Site | Public URL | Proxy target |
| --- | --- | --- |
| Keycloak | `https://auth.example.org/keycloak` | `http://127.0.0.1:8081` |
| Portal | `https://portal.example.org` | `http://127.0.0.1:9080` |
| Species | `https://tcas.example.org` | `http://127.0.0.1:9090` |

A minimal Caddy configuration:

```caddyfile
auth.example.org {
    reverse_proxy 127.0.0.1:8081
}

portal.example.org {
    reverse_proxy 127.0.0.1:9080
}

tcas.example.org {
    reverse_proxy 127.0.0.1:9090
}
```

With nginx, proxy to the same ports and forward `Host`, `X-Forwarded-Host`,
`X-Forwarded-For` and `X-Forwarded-Proto`; Keycloak and both stacks honour the
forwarded headers. Set `KEYCLOAK_PUBLIC_URL` to the public auth URL.

The portal URL must be reachable from a species server, not just from the
browser, because the species nginx proxies `/blast/` to the portal's `/blast/`.
Keycloak and
MySQL are **not** shared across servers: each server runs its own
`emobase-infra`. When both stacks are on the same server, set
`PORTAL_INTERNAL_URL` to `http://host.docker.internal:9080` (or another
hostname reachable from the species stack's default Docker network) to keep
BLAST traffic internal; otherwise it defaults to `PORTAL_SERVER`.

## 4. Deploy the local infrastructure

```bash
cd ~/emobase/emobase-infra
./init.sh \
  --portal-server https://portal.example.org \
  --keycloak-public-url https://auth.example.org/keycloak
docker compose up -d --wait
```

This creates the `emobase-infra` Docker network, initialises the MySQL
databases (`emobase_genomics`, `keycloak`, `directus`) and imports the shared
portal realm (`KEYCLOAK_REALM`, default `ibb`). It also generates
`MYSQL_ROOT_PASSWORD`, `KEYCLOAK_CLIENT_SECRET` and the Keycloak admin/`coco`
credentials. Review `.env` before continuing.

Check that both services are healthy:

```bash
docker compose ps
```

Keycloak is published on `KEYCLOAK_HOST_PORT` (default `8081`) and should be
reached by browsers at `KEYCLOAK_PUBLIC_URL`. The admin console is at
`https://auth.example.org/keycloak/admin/`. The portal no longer proxies
Keycloak.

## 5. Deploy the portal

```bash
cd ~/emobase/emobase-portal-deploy
./init.sh --public-server https://portal.example.org --http-port 9080
```

`init.sh` reads the MySQL/Keycloak secrets, Keycloak endpoints, realm and client
ID from `../emobase-infra/.env`, asks for anything not passed as a flag, and
generates the remaining secrets. The portal always uses the shared realm. It
writes `.env` and prints the next commands. Review `.env` before continuing,
especially:

- `PUBLIC_SERVER` — the exact URL users open; it must match
  `PORTAL_PUBLIC_SERVER` in `emobase-infra/.env`
- `HTTP_PORT` — the host port nginx binds
- `EMOBASE_GENOMICS_IMAGE`, `EMOBASE_PORTAL_WEB_IMAGE` — pin a release/SHA tag
  for production instead of a moving branch tag

JBrowse2 gene links are built from each assembly's `infoUrl` (set in the admin
UI), not from an environment variable.

Then pull images, initialise JBrowse2, run the migrations and start:

```bash
docker compose --profile '*' pull
docker compose --profile setup run --rm setup-jbrowse2-web
docker compose --profile update run --rm migrate-genomics-db
docker compose --profile update run --rm migrate-genomics-es
docker compose up -d --wait
```

> **Reusing a Docker host?** Compose volumes are global to the project name,
> not to the checkout directory. If `emobase-portal_jbrowse2-data` exists from
> an earlier run, `jbrowse create /web` fails because `/web` is not empty.
> Remove that volume (`docker volume rm emobase-portal_jbrowse2-data`) or use a
> fresh project name consistently (`-p` or `COMPOSE_PROJECT_NAME`).
> `docker compose --profile '*' down -v` also resets it, but destroys all
> portal data.

Verify:

```bash
./scripts/smoke-test.sh
```

Open `PUBLIC_SERVER` in a browser. Keycloak admin is at
`KEYCLOAK_PUBLIC_URL/admin/` using `KEYCLOAK_ADMIN` /
`KEYCLOAK_ADMIN_PASSWORD` from `../emobase-infra/.env`.

The migrations must run before the portal UI is used. Without them the API
returns `500` because tables such as `versions` do not exist yet.

`/blast/` may return `502` at this point. That is expected until the first
release has built BLAST databases; the smoke test only warns about it.

**The portal is now the source of `PORTAL_SERVER`.** Read it from its `.env`:

```bash
grep '^PUBLIC_SERVER=' ~/emobase/emobase-portal-deploy/.env
```

## 6. Deploy the species site

`emobase-deploy` needs two public URLs, an optional internal portal URL and a
per-species Directus database:

| Variable | Meaning | Value |
| --- | --- | --- |
| `PUBLIC_SERVER` | The URL users open for **this species site** | e.g. `https://tcas.example.org` |
| `PORTAL_SERVER` | Public portal URL used by browser links | the portal's `PUBLIC_SERVER`, e.g. `https://portal.example.org` |
| `PORTAL_INTERNAL_URL` | Portal URL used inside containers | e.g. `http://host.docker.internal:9080` on the same host |
| `KEYCLOAK_PUBLIC_URL` | Local Keycloak public URL (from infra) | e.g. `https://auth.example.org/keycloak` |
| `KEYCLOAK_REALM` | Shared portal realm or a dedicated species realm | `ibb` or `ibb_tcas` |
| `DIRECTUS_DB_DATABASE` | This species' database in the local MySQL | e.g. `directus_tcas` |

The species stack uses the `emobase-infra` on **its own server**, not the
portal server's MySQL or Keycloak. Keep the `emobase-infra` checkout in the
sibling directory (or pass `--infra-dir`).

Let `init.sh` copy the portal URL straight from the sibling checkout:

```bash
cd ~/emobase/emobase-deploy
./init.sh \
  --species Tcas \
  --public-server https://tcas.example.org \
  --portal-dir ../emobase-portal-deploy
```

`--portal-dir` reads `PUBLIC_SERVER` from that directory's `.env`. If the
directory is named `../emobase-portal-deploy`, the flag can be omitted:

```bash
./init.sh --species Tcas --public-server https://tcas.example.org
```

On the same Docker host, add the internal portal URL so BLAST/SSR traffic does
not leave through the public domain:

```bash
./init.sh \
  --species Tcas \
  --public-server https://tcas.example.org \
  --portal-dir ../emobase-portal-deploy \
  --portal-internal-url http://host.docker.internal:9080
```

On a separate species server, first run that server's own `emobase-infra`,
then leave `PORTAL_INTERNAL_URL` as the default (`PORTAL_SERVER`) and pass the
portal URL explicitly:

```bash
./init.sh \
  --species Tcas \
  --public-server https://tcas.example.org \
  --portal-server https://portal.example.org
```

During `init.sh` you choose the Keycloak realm model:

- **Shared realm** — reuse the portal's realm. One team manages users for the
  portal and every species. You must register the species origin with
  `allow-origin.sh`.
- **Dedicated realm** — `init.sh` creates `ibb_<species>` (or the realm name you
  pass) with its own client and `coco` administrator. Separate people can
  manage that site without access to the portal realm. No `allow-origin.sh`
  call is needed.

Create the Directus database, register the origin only for the shared realm,
then initialise Directus and start the stack:

```bash
cd ~/emobase/emobase-infra
./scripts/create-database.sh directus_tcas
# Only for the shared realm; a dedicated realm is configured by init.sh.
./scripts/allow-origin.sh https://tcas.example.org

cd ../emobase-deploy
docker compose run --rm setup-directus
docker compose up -d --wait
```

Open the species URL and sign in with `COCO_USERNAME` / `COCO_PASSWORD` from the
species `.env`. For a shared realm these are copied from
`../emobase-infra/.env`; for a dedicated realm they belong only to that species.
The password is temporary; Keycloak asks for a new one on first login.

To add another species later, clone `emobase-deploy` into another directory (or
another server) and repeat only this step with the matching `--species`,
`--public-server` and `--portal-server` / `--portal-dir`. Each species checkout
needs its own `.env`, Directus database/data and containers. On one Docker
host, also give each stack a distinct `--http-port` and `--container-prefix`.
Run `create-database.sh` once per species, and `allow-origin.sh` only when the
species reuses the shared realm.

## 7. Verify the whole setup

On the portal server:

```bash
cd ~/emobase/emobase-portal-deploy
./scripts/smoke-test.sh
```

On the species server:

```bash
cd ~/emobase/emobase-deploy
docker compose ps
curl -I "$(grep '^PUBLIC_SERVER=' .env | cut -d= -f2-)"
curl -I "$(grep '^KEYCLOAK_PUBLIC_URL=' .env | cut -d= -f2-)/realms/$(grep '^KEYCLOAK_REALM=' .env | cut -d= -f2-)"
```

Manual checks:

1. Portal opens and `/admin/versions` loads (no `versions` 500).
2. Keycloak opens at `KEYCLOAK_PUBLIC_URL` and sign-in works on both sites.
3. The species site loads genomics data from the portal.
4. JBrowse2 opens from a genome link.
5. BLAST opens in the species site. It returns `502` until BLAST databases have
   been built from a release.

## 8. Updating a deployment

Update `emobase-infra` first, then the portal, then the species UI. The portal
may add database columns or API fields that the species UI expects.

Infra:

```bash
cd ~/emobase/emobase-infra
git pull
docker compose pull
docker compose up -d --wait
```

Portal:

```bash
cd ~/emobase/emobase-portal-deploy
git pull
docker compose --profile '*' pull
docker compose --profile update run --rm migrate-genomics-db
docker compose --profile update run --rm migrate-genomics-es
docker compose up -d --wait
./scripts/smoke-test.sh
```

Species:

```bash
cd ~/emobase/emobase-deploy
git pull
docker compose pull
docker compose up -d --wait
```

A plain `docker compose up -d` does not re-pull a tag that already exists
locally; always `pull` first, or pin an immutable tag and change it explicitly.
To roll back, set the previous image tag in `.env` and repeat the same commands.

## 9. Backup essentials

At minimum back up:

- `emobase-infra/.env`, `emobase-portal-deploy/.env` and every
  `emobase-deploy/.env` (secrets)
- `emobase-portal-deploy/data/public/` (version files, assembly images)
- `emobase-deploy/data/`, `emobase-deploy/images/`
- The local MySQL instance (portal genomics, Keycloak realms/users and every
  Directus database)
- Elasticsearch data from each stack
- Directus uploads (`directus-data` volume per species)

Example MySQL dump:

```bash
cd ~/emobase/emobase-infra
docker compose exec -T mysql \
  sh -c 'exec mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" --all-databases' \
  > emobase-mysql-$(date +%F).sql
```

For Elasticsearch, use the snapshot procedure in
`emobase-deploy/README.md` for the species stack. The portal's Elasticsearch is
not exposed on a host port; snapshot it from inside the compose network or back
up the `es-data` volume.

## 10. Adding another server

Each server runs its own `emobase-infra`. MySQL and Keycloak are **not** shared
across servers; only the portal URL is shared.

To add a server:

1. Clone `emobase-infra` and `emobase-deploy` (and the portal if this server
   will run one) and give the server its own Keycloak DNS name or port.
2. Run `emobase-infra/init.sh` on that server with its own
   `--keycloak-public-url` and, if it has a portal, `--portal-server`.
3. Start the local infra. Deploy the local portal, if present.
4. Run `emobase-deploy/init.sh` against the local infra. If the portal lives on
   another server, pass `--portal-server https://portal.example.org`.
5. Create the Directus database and configure the realm on the local infra:
   `./scripts/create-database.sh directus_tcas` and
   `./scripts/allow-origin.sh https://tcas.example.org` (shared realm only).
6. Make sure the species server can reach the portal:

   ```bash
   curl -fsS https://portal.example.org/api/genomicsservice/v1/public/versions?status=READY >/dev/null
   ```

The reverse direction is **not** supported: the portal does not call the
species stack. The species stack consumes the portal for genomics/JBrowse2/
BLAST and its own local infra for MySQL/Keycloak.

## 11. Troubleshooting

| Symptom | Check |
| --- | --- |
| `502` on the portal | `docker compose ps` and `docker compose logs --tail=200 fe genomicsservice` |
| Shared services unreachable | `cd ~/emobase/emobase-infra && docker compose ps`; `docker compose logs --tail=200 mysql keycloak` |
| Login redirect loop | `PUBLIC_SERVER` must exactly match the browser URL and `emobase-infra/.env`'s `PORTAL_PUBLIC_SERVER`; `KEYCLOAK_PUBLIC_URL` must be the URL the browser uses for Keycloak |
| Keycloak redirects to the wrong host | `KEYCLOAK_PUBLIC_URL` does not match the URL exposed by the TLS proxy or host port |
| Species login says invalid redirect URI | Shared realm only: register the origin with `emobase-infra/scripts/allow-origin.sh https://tcas.example.org`. A dedicated realm is configured by `init.sh`. |
| Species site has no data | Portal needs a READY release; species `.env` `PORTAL_SERVER` must match the portal URL |
| BLAST `502` | Expected until a release has built BLAST databases; check `/blast/searchdata.json` |
| JBrowse2 empty | The release's JBrowse2 jobs must have completed; check worker logs |
| `port is already allocated` | Change `HTTP_PORT` (and `PUBLIC_SERVER` if it contains the port) |
| Image pull denied | `docker login ghcr.io` with `read:packages`; or make the package public |
| `setup-jbrowse2-web` says `/web` has existing files | A stale `emobase-portal_jbrowse2-data` volume exists from another checkout/project. Remove it (`docker volume rm emobase-portal_jbrowse2-data`) or use a fresh Compose project name (`-p` / `COMPOSE_PROJECT_NAME`). |
| Directus database missing | Run `emobase-infra/scripts/create-database.sh <DIRECTUS_DB_DATABASE>` |
| Directus not initialised | Re-run `docker compose run --rm setup-directus` |
| Portal API `500` after first start | Migrations were skipped; run both `migrate-genomics-*` services |
| Changed `PUBLIC_SERVER` or `KEYCLOAK_PUBLIC_URL` after first start | Update Keycloak's issuer and redirect URIs in the admin console, or reset Keycloak |

Changing `PUBLIC_SERVER` or `KEYCLOAK_PUBLIC_URL` after the first start is not
automatic for Keycloak: the realm is imported once, including its issuer and
redirect URIs. Dedicated realms are created during species init. For a
throwaway deployment the cleanest fix is to reset the infrastructure and re-run
the initialisers:

```bash
cd ~/emobase/emobase-infra && docker compose down -v   # destroys all shared data
cd ../emobase-portal-deploy && docker compose --profile '*' down -v
cd ../emobase-deploy && docker compose --profile '*' down -v
```

Only do that if you can lose the data.

## Deployment checklist

- [ ] Docker + Compose available (and `python3` on the host)
- [ ] All three repositories cloned into the same parent directory (or portal
      URL noted for a separate server)
- [ ] `emobase-infra/.env` reviewed; `KEYCLOAK_PUBLIC_URL`,
      `KEYCLOAK_HOST_PORT` and `PORTAL_PUBLIC_SERVER` are final; secrets
      generated
- [ ] Local MySQL and Keycloak healthy; Keycloak reachable at
      `KEYCLOAK_PUBLIC_URL`
- [ ] Portal `.env` reviewed; `PUBLIC_SERVER` matches the infra value
- [ ] Portal images pulled
- [ ] Portal JBrowse2 setup and both migrations completed
- [ ] Portal stack up; smoke test passes
- [ ] Species `.env` has `PUBLIC_SERVER` = species URL
- [ ] Species `.env` has `PORTAL_SERVER` = portal `PUBLIC_SERVER`
- [ ] Species `.env` uses the local `KEYCLOAK_PUBLIC_URL`
- [ ] Species `.env` has a unique `DIRECTUS_DB_DATABASE`
- [ ] Species Directus database created in the local MySQL
- [ ] Species Keycloak realm chosen (shared or dedicated)
- [ ] Shared realm: species origin registered with `allow-origin.sh`
- [ ] Dedicated realm: `KEYCLOAK_REALM`/client/user created and recorded
- [ ] Same-host deployments set `PORTAL_INTERNAL_URL` to an internal address
- [ ] `INTERNAL_API_*`, `INTERNAL_KEYCLOAK_ISSUER`, `INTERNAL_DIRECTUS_URL` are
      set or intentionally falling back to public URLs
- [ ] Directus setup completed
- [ ] Species stack up; both sites open in a browser
- [ ] BLAST/JBrowse2 verified after the first release
- [ ] `.env` files, local MySQL dump and data backed up
