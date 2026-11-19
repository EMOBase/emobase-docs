# EMOBase deployment runbook

This is the full empty-server → working-site procedure using the `emobase`
command. The CLI is the control point for the three deployment repositories:

| Repository | What it runs | Default host port |
| --- | --- | --- |
| [`emobase-infra`](https://github.com/EMOBase/emobase-infra) | MySQL, Keycloak and Elasticsearch for all stacks on this server | `8181` (Keycloak), `9202` (Elasticsearch, loopback only) |
| [`emobase-portal-deploy`](https://github.com/EMOBase/emobase-portal-deploy) | Portal UI, genomics API/workers, JBrowse2, BLAST | `9080` |
| [`emobase-deploy`](https://github.com/EMOBase/emobase-deploy/tree/dev) | Single-species UI and Directus (`dev` branch) | `9090` |

`emobase` only orchestrates: it clones checkouts, calls each repository's own
`init.sh`, and runs their Compose commands in the right order. Every checkout
keeps its own `.env` and secrets. The CLI never stores secrets and never copies
them between checkouts.

Each repository is still deployable on its own. If you do not want to use the
CLI, see [Without the CLI](#without-the-cli) for the repository references.

## How the CLI is organised

| Command                           | What it does                                                                                                                       |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `emobase create infra`            | Clone `emobase-infra` if needed and scaffold its `.env`                                                                            |
| `emobase create portal`           | Clone `emobase-portal-deploy` if needed and scaffold its `.env`; creates the portal database and realm if infra is already running |
| `emobase create species`          | Clone `emobase-deploy` (branch `dev`) into `<species>` and scaffold its `.env`; registers the realm if Keycloak is already running |
| `emobase bootstrap`               | Pull images, start infra, create databases and realms, run JBrowse2 setup and migrations, initialise Directus, start everything    |
| `emobase up` / `down`             | Start or stop the stacks in dependency order                                                                                       |
| `emobase status`                  | Show containers, databases, realms and configured URLs                                                                             |
| `emobase doctor`                  | Check the cross-repository configuration (`--live` also checks the running services)                                               |
| `emobase domain`                  | Inspect or change the browser-facing URLs and Keycloak issuer                                                                      |
| `emobase update`                  | `git pull`, pull images, migrate and restart in the documented order                                                               |
| `emobase sync`                    | Create missing databases and register missing realm origins after manual edits                                                     |
| `emobase species list` / `remove` | List species checkouts, or stop and purge one                                                                                      |
| `emobase prune`                   | Remove databases, realms and index prefixes recorded in the ownership ledger whose checkout is gone                                |

`create` writes `.env` and, when infra is already running, also creates the
portal database and registers realms. It never starts containers. On a fresh
server, use `bootstrap` for the first start; `up` is for restarts once the
databases and realms exist.

## Workspace model

The CLI works on a **workspace**: one parent directory that contains the
checkouts.

```text
/srv/emobase/
├── infra/
├── portal/
├── Tcas/
├── Dmel/                      # one emobase-deploy clone per species
└── .emobase/state.json        # ownership ledger (databases, realms, index prefixes)
```

`emobase create` uses these names and prompts for the directory, so a checkout
can be renamed per deployment without breaking discovery.

Run `emobase` from the workspace, or pass `--workspace DIR`. Commands discover
the checkouts from the current directory or its parent.

`create` asks before cloning a missing checkout; `--dir` places it elsewhere.
Use the same parent for all checkouts so the repositories can read each other's
`.env`.

The ownership ledger records which databases, realms and Elasticsearch index
prefixes each checkout created. `emobase prune` uses it to clean up resources of
deleted stacks without guessing from names. Copy it only together with the
workspace; do not reuse it on another server.

## Prerequisites

- Linux server with Docker Engine 24+ and Compose v2
- `git` and `python3` (the Keycloak helpers use it)
- Outbound HTTPS to `ghcr.io`, Docker Hub and `quay.io`
- DNS names (or one name plus ports) for the portal, each species site and
  Keycloak
- Enough RAM/disk for one local MySQL/Keycloak/Elasticsearch trio, and the
  application containers. Sharing them saves roughly 2-3 GiB at four species on
  one server, but 8 GB is only enough for a portal plus one or two species; plan
  24-32 GB for four to five species on a server.
- Ports reachable by the browser: `8181` for Keycloak, `9080` for the portal and
  `9090` for the species site when using the built-in services directly

Check Docker:

```bash
docker version
docker compose version
```

### Install the CLI

Download a release (`amd64`/`arm64`, Linux/macOS) from the
[emobase-cli releases page](https://github.com/EMOBase/emobase-cli/releases), or
build from source with Go 1.25+:

```bash
git clone https://github.com/EMOBase/emobase-cli.git
cd emobase-cli
sudo make install
emobase --version
```

## Choose the public URLs

Decide the URLs **before** the first start. `KEYCLOAK_PUBLIC_URL` becomes the
OIDC issuer; the portal and species URLs become redirect URIs and public links.
Changing them later requires `emobase domain` and invalidates existing login
sessions.

Without `--local`, each `create` command offers the detected host IPv4 address
as the default URL. That is the address used for the default route, not
necessarily a NAT public address; enter the public IP or domain when the server
is behind NAT.

Every server needs its own Keycloak URL, and each portal/species site needs its
own origin — a hostname or a port. They serve their UI at `/`, so do not put
them under path prefixes of the same origin.

### A. Direct ports (simple, no TLS)

| Site | URL |
| --- | --- |
| Keycloak | `http://SERVER_IP:8181/keycloak` |
| Portal | `http://SERVER_IP:9080` |
| Species | `http://SERVER_IP:9090` |

Use the server's LAN/public IP or a DNS name for the browser-facing URLs.

### B. TLS reverse proxy (production)

Example with three DNS names:

| Site | Public URL | Proxy target |
| --- | --- | --- |
| Keycloak | `https://auth.example.org/keycloak` | `http://127.0.0.1:8181` |
| Portal | `https://portal.example.org` | `http://127.0.0.1:9080` |
| Species | `https://tcas.example.org` | `http://127.0.0.1:9090` |

A minimal Caddy configuration:

```caddyfile
auth.example.org {
    reverse_proxy 127.0.0.1:8181
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
Keycloak and MySQL are **not** shared across servers: each server runs its own
infra stack.

## Quick start: local trial

Use this to try the whole stack on a laptop or desktop before any DNS/TLS
exists. Portal, Keycloak and species use `localhost` over plain HTTP on ports
`9080` (portal), `8181` (Keycloak) and `9090` (species).

```bash
mkdir -p ~/emobase && cd ~/emobase

emobase create infra --local
emobase create portal --local
emobase create species --species Tcas --local

emobase bootstrap
emobase status
```

What to expect:

- `create infra --local` writes `KEYCLOAK_PUBLIC_URL=http://localhost:8181/keycloak`.
  The container-facing Keycloak URL is fixed to `http://keycloak:8080/keycloak`.
- `create portal --local` reads the Keycloak URLs and MySQL password from
  `../infra/.env` and creates the portal realm credentials.
- `create species --species Tcas --local` copies the portal URL from the
  sibling portal checkout. `PORTAL_INTERNAL_URL` is derived as
  `http://host.docker.internal:9080`, so BLAST/SSR traffic stays internal.
  Docker Desktop provides `host.docker.internal`; on Linux the compose files
  map it with `host-gateway`.
- `bootstrap` pulls images, starts infra, creates the databases and realms,
  sets up JBrowse2, runs the portal migrations, initialises Directus and
  starts every stack.

Open `http://localhost:9080` for the portal and `http://localhost:9090` for the
species site. Keycloak admin is at `http://localhost:8181/keycloak/admin/`.

On Apple Silicon, enable Rosetta for `linux/amd64` first, or BLAST will not
start (see the portal README).

To run the whole scaffolding unattended, add `-y` to each `create` command.
`bootstrap` has no questions; use `--dry-run` to print the exact commands.

### Verify the local trial

```bash
emobase status
emobase doctor --live
```

Sign in with the seeded user: `emobase credential show portal` prints it for one
stack, while `emobase credential show` lists every realm's user without the
password. `emobase credential show infra` prints the Keycloak admin login for the
console. The seed password is temporary, so Keycloak asks for a new one on the
first sign-in. Then confirm that JBrowse2/BLAST links point at the portal URL.
BLAST returns `502` until the first release has built databases.

### Access from another machine (optional)

For access from another machine on the LAN, use non-local public URLs from the
start. Because Keycloak's issuer and the realm origins are fixed at creation,
it is cleaner to reset and re-create than to edit them afterwards:

```bash
emobase down
# On each checkout: docker compose --profile '*' down -v (destroys data)
cd ~/emobase

emobase create infra --keycloak-public-url http://192.168.1.10:8181/keycloak --force
emobase create portal --public-server http://192.168.1.10:9080 --force
emobase create species --species Tcas \
  --public-server http://192.168.1.10:9090 \
  --force

emobase bootstrap
```

Resetting is destructive; only do it while the deployment has no data worth
keeping.

## Deploy on a server

The four steps below are the same for one server that runs the portal and
species, and for a species-only server (skip the portal step).

Every `create` command can also run interactively: omit the flags and it prompts
for each value, including the checkout directory (`infra`, `portal`, `<Species>`)
— press Enter to keep the default.

### 1. Create the infrastructure

```bash
mkdir -p /srv/emobase && cd /srv/emobase

emobase create infra --keycloak-public-url https://auth-a.example.org/keycloak
```

Clones the `emobase-infra` repository into `infra/` and writes its `.env`,
including generated MySQL root and Keycloak admin passwords. Review it before the
first start: this file is the authority for the shared secrets, and every other
stack reads them from it. No realm, database or index is created here.

### 2. Create the portal (skip on a species-only server)

```bash
emobase create portal --public-server https://portal.example.org
```

Reads the Keycloak URLs and MySQL password from `../infra/.env` and
creates the `emobase-portal` realm, client and seed user. Review `PUBLIC_SERVER`
and `HTTP_PORT`, and pin `EMOBASE_GENOMICS_IMAGE` / `EMOBASE_PORTAL_WEB_IMAGE` to
a release tag rather than a moving branch tag.

`PUBLIC_SERVER` must be the exact URL users open **and** a host the web app
accepts: `security.allowedDomains` in `emobase-portal-web` currently allows
`**.uni-goettingen.de`. Any other domain fails login with
`403 Cross-site POST form submissions are forbidden` until it is added there and
a new image is published.

### 3. Create the species

```bash
emobase create species --species Tcas --public-server https://tcas.example.org
```

Copies `PORTAL_SERVER` from the sibling portal checkout and derives
`PORTAL_INTERNAL_URL`. On a species-only server, name the remote portal:

```bash
emobase create species \
  --species Tcas \
  --public-server https://tcas.example.org \
  --portal-server https://portal.example.org
```

The realm defaults to the portal realm when a portal checkout exists. Override
it with `--keycloak-realm join:emobase-portal` to share users, or
`--keycloak-realm new:emobase-tcas` for a dedicated realm. The resulting
variables are listed in [Reference: what `create` writes](#reference-what-create-writes).

### 4. Bootstrap

```bash
emobase bootstrap
```

Runs infra → portal (JBrowse2 setup, migrations, shared realm, smoke test) →
each species (Directus database, realm, initialise, start). `--dry-run` prints
every command; `--only infra|portal|species|all|<stack>`, `--no-pull`,
`--skip-setup`, `--skip-migrations` and `--skip-smoke` narrow it. A stack can
be named by species code, checkout name or Compose project, for example
`emobase bootstrap --only Tcas`.

The portal migrations must run before the UI is used; without them the API
returns `500`.

### Verify the deployment

```bash
emobase status
emobase doctor --live
```

Then in a browser: the portal opens and `/admin/versions` loads; sign-in works on
both sites; the species site loads genomics data from the portal; JBrowse2 opens
from a genome link; BLAST opens (it returns `502` until the first release has
built databases).

On a species-only server, confirm the remote portal answers:

```bash
curl -fsS https://portal.example.org/api/genomicsservice/v1/public/versions?status=READY >/dev/null
```

The portal never calls a species stack; the dependency only runs the other way.

## Add another species

Each species has its own checkout, `.env`, Directus database/data and
containers. Give each stack a distinct HTTP port:

```bash
emobase create species \
  --species Dmel \
  --public-server https://dmel.example.org \
  --http-port 9091

emobase bootstrap --only species
```

`--directus-db`, `--container-prefix` and `--es-index-prefix` default to
values derived from the species code, so the defaults are unique per species.
Run `emobase doctor` after creating a species to confirm there are no duplicate
ports, databases, index prefixes or URLs.

`bootstrap --only species` expects the local infra to be running; start it with
`emobase up --only infra` first if it is stopped.

On a species-only server, add `--portal-server https://portal.example.org`.

## Day-2 operations

### Inspect

```bash
emobase status          # inventory: what runs; which databases/realms exist
emobase species list
emobase doctor          # verdict on the cross-repo config (no Docker needed)
emobase doctor --live   # verdict on the running system against that config
```

`emobase status` shows whether each stack's containers are running, which
databases exist in the local MySQL, which realms exist, and the URLs each
checkout is configured to use.

`doctor` does not list state — it checks the state against what the configuration
expects, prints findings with hints and exits non-zero on errors, which is why
acceptance checks use it rather than `status`. `--live` adds the runtime probes:
container health, MySQL databases, Keycloak realm/client/redirect URIs and the
portal's HTTP response.

`emobase doctor --fix` applies safe fixes to `.env` files (a `.env.bak` is
kept). `--fix --live` can also create a missing database and register a
missing redirect URI. It never performs destructive issuer/origin changes.

### Update a deployment

```bash
emobase update
```

This pulls each checkout and its images, runs the portal migrations, restarts
the stacks and (unless `--skip-smoke`) runs the portal smoke test. The order is
infra → portal → species, because the portal may add database columns or API
fields that the species UI expects.

Useful flags: `--only infra|portal|species|all|<stack>`, `--no-git`,
`--skip-migrations`, `--skip-smoke`, `--dry-run`. For example,
`emobase update --only Tcas` updates one species stack.

A plain `up` does not re-pull a tag that already exists locally; `update` (or
`up --pull`) does. To roll back, set the previous image tag in the relevant
`.env` and run `emobase up` — or `emobase update --no-git` after pinning.

### Change a public URL

```bash
emobase domain
```

It shows each current URL, lets you edit them one at a time (current value as the
default), prints the resulting plan, and asks before applying. It updates every
affected `.env` plus the Keycloak realm origins, then restarts those stacks.

For automation or a review-first workflow, use the non-interactive form:
`emobase domain status`, `emobase domain plan --auth …`, then
`emobase domain apply --auth …`.

Changing the Keycloak URL signs everyone out: it is the OIDC issuer. On a
species-only server the portal is not asked about, and `PORTAL_SERVER` is left as
it is.

Old realm origins are left in place; remove them in the Keycloak admin console
if the cutover is permanent.

### Reconcile after manual edits

```bash
emobase sync --dry-run
emobase sync
```

`sync` creates missing portal/species databases, registers missing realm
origins, and records them in the ownership ledger. It does not pull images,
run migrations or start containers.

### Stop and start

```bash
emobase down
emobase up
emobase up --pull
```

`down` stops species → portal → infra; `up` starts infra → portal → species.

### Remove a species

```bash
emobase species remove Tcas
```

Without `--purge` this only prints the commands it would run. With
`--purge --yes` it stops the stack, deletes its Compose volumes, drops the
Directus database, deletes its `<prefix>-*` Elasticsearch indices and removes
the checkout directory (keep it with `--keep-dir`). A dedicated realm is
cleaned up by `emobase prune`, which only touches resources recorded in the
ownership ledger and skips shared realms that are still in use.

## Backup essentials

Back up:

- `infra/.env`, `portal/.env` and every `<species>/.env` (secrets)
- `portal/data/public/` (version files, assembly images)
- `<species>/data/`, `<species>/images/`
- The local MySQL instance (portal genomics, Keycloak realms/users and every
  Directus database)
- Elasticsearch data from the shared `infra` stack
- Directus uploads (`directus-data` volume per species)

Example MySQL dump (host-level, outside the CLI):

```bash
cd /srv/emobase/infra
docker compose exec -T mysql \
  sh -c 'exec mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" --all-databases' \
  > emobase-mysql-$(date +%F).sql
```

For Elasticsearch, use the snapshot procedure in the `emobase-infra` README.
The instance listens on `127.0.0.1:9202` for administration and snapshots; it
is not reachable from the LAN. Snapshots are written to
`infra/es_bk/`.

## Troubleshooting

Start with the CLI, then use the owning checkout's README for stack details:

```bash
emobase status
emobase doctor --live
emobase domain plan
```

| Symptom | Check |
| --- | --- |
| A stack is not running | `emobase status`; then `cd` to that checkout and read its README for log commands |
| Shared services unreachable | `emobase doctor --live`; start infra with `emobase up --only infra` |
| Login redirect loop | `PUBLIC_SERVER` must exactly match the browser URL; `KEYCLOAK_PUBLIC_URL` must be the URL the browser uses for Keycloak. `emobase doctor --live` reports mismatched origins. |
| Login POST answers `403 Cross-site POST form submissions are forbidden` | The browser's `Origin` does not match the URL the web container derived from the forwarded headers. Check that `PUBLIC_SERVER` equals the address in the address bar and that the TLS proxy sends `X-Forwarded-Proto: https`. Fix `.env` and restart that stack; a host outside `**.uni-goettingen.de` must be added to `security.allowedDomains` in `emobase-portal-web` / `emobase-web` and a new image published. |
| Keycloak redirects to the wrong host | `emobase domain status`; the configured Keycloak URL must match the URL exposed by the TLS proxy |
| Species login says invalid redirect URI | `emobase doctor --fix --live` registers the missing origin, or run `emobase domain apply` after a URL change |
| Species site has no data | Portal needs a READY release; species `PORTAL_SERVER` must match the portal URL |
| Diff between checkouts | `emobase doctor` lists the mismatched keys; `--fix --yes` copies the authoritative infra values |
| BLAST `502` | Expected until a release has built BLAST databases; check `/blast/searchdata.json` |
| JBrowse2 empty | The release's JBrowse2 jobs must have completed; check worker logs |
| `setup-jbrowse2-web` fails with `HTTP 403` | Update the portal checkout; setup downloads the pinned `JBROWSE_DOWNLOAD_URL` release instead of querying the rate-limited GitHub API |
| `port is already allocated` | Change `HTTP_PORT` in that checkout (`--http-port` when creating) and run `emobase doctor` |
| Image pull denied | `docker login ghcr.io` with `read:packages`; or make the package public |
| Directus database missing | `emobase sync`, or `emobase doctor --fix --live` |
| Directus not initialised | `emobase bootstrap --only species` |
| Search returns no results after moving to shared Elasticsearch | Check the species `EMOBASE_ES_INDEX_PREFIX` and `docker compose logs es-proxy`; indices are `<prefix>-*` |
| Portal API `500` after first start | Migrations were skipped; `emobase bootstrap --only portal` (without `--skip-migrations`) |
| Changed a URL after first start | `emobase domain plan`, then `emobase domain apply`; the issuer change forces users to sign in again |

For a throwaway deployment where the URLs changed before any real data
existed, resetting is often simpler than migrating:

```bash
emobase down
# then, in each checkout: docker compose --profile '*' down -v
emobase bootstrap
```

Only do that if you can lose the data.

## Deployment checklist

- [ ] Docker + Compose available, `python3` and `git` installed
- [ ] `emobase --version` works on the server
- [ ] Public URLs chosen and DNS/TLS ready (or `--local` for a trial)
- [ ] Workspace created; `emobase create infra` completed and `.env` reviewed
- [ ] `KEYCLOAK_PUBLIC_URL` and `KEYCLOAK_HOST_PORT` are final
- [ ] Portal checkout created and `.env` reviewed (portal servers only)
- [ ] Image tags pinned for production
- [ ] Each species checkout created with a unique HTTP port, database and
      Elasticsearch index prefix
- [ ] Realm choice made (joined vs dedicated) for every species
- [ ] `emobase doctor` reports no errors
- [ ] `emobase bootstrap` completed
- [ ] `emobase status` shows every expected container, database and realm
- [ ] `emobase doctor --live` passes
- [ ] Portal and species sites open; Keycloak sign-in works
- [ ] JBrowse2/BLAST verified after the first release
- [ ] `.env` files, MySQL dump and data volumes backed up

## Reference: what `create` writes

`create` only scaffolds `.env`; it never starts containers. These are the values
operators most often review or change.

**`infra/.env`** — `KEYCLOAK_PUBLIC_URL`, `KEYCLOAK_HOST_PORT`, the fixed
`KEYCLOAK_INTERNAL_URL=http://keycloak:8080/keycloak`, `ELASTICSEARCH_HOST_PORT`,
`ELASTICSEARCH_JAVA_OPTS`, `ELASTICSEARCH_MEM_LIMIT`, the shared Docker network
name, and the generated MySQL root and Keycloak admin passwords.

Elasticsearch is shared by the portal and every species: the portal writes
`genomics-*` indices, each species writes `<prefix>-*` under its own
`EMOBASE_ES_INDEX_PREFIX`. `ELASTICSEARCH_HOST_PORT` binds to `127.0.0.1` only,
for administration and snapshots.

**Portal** — `PUBLIC_SERVER` and `HTTP_PORT` are the browser-facing values;
`EMOBASE_PORTAL_WEB_IMAGE` and `EMOBASE_GENOMICS_IMAGE` select the images.
JBrowse2 gene links come from each assembly's `infoUrl` (set in the admin UI),
not from an environment variable. Local development on `localhost` is outside
the `**.uni-goettingen.de` allowlist: start the dev server with
`DISABLE_ORIGIN_CHECK=true`.

**Species `.env`**

| Variable | Meaning | Value |
| --- | --- | --- |
| `PUBLIC_SERVER` | The URL users open for this species | e.g. `https://tcas.example.org` |
| `PORTAL_SERVER` | Public portal URL used by browser links | the portal's `PUBLIC_SERVER` |
| `PORTAL_INTERNAL_URL` | Portal URL inside containers | `http://host.docker.internal:<portal port>` for a local portal; unset for a remote portal |
| `KEYCLOAK_PUBLIC_URL` | This server's Keycloak URL | from `infra/.env` |
| `KEYCLOAK_REALM` | Joined realm or a dedicated species realm | e.g. `emobase-portal`, `emobase-tcas` |
| `DIRECTUS_DB_DATABASE` | This species' database | e.g. `directus_tcas` |
| `EMOBASE_ES_INDEX_PREFIX` | This species' slice of the shared Elasticsearch | e.g. `tcas`; must be unique and not `genomics` |
| `COMPOSE_FILE` | Compose files for this stack | `compose.yml`; Tcas adds `:compose.tcas.yml` |

A checkout created before Elasticsearch was shared has no
`EMOBASE_ES_INDEX_PREFIX`. Add `EMOBASE_ES_INDEX_PREFIX=<species>` and re-import
the data (see the [species README](https://github.com/EMOBase/emobase-deploy/blob/dev/README.md#elasticsearch)).

## Without the CLI

Each repository remains independently deployable. If you prefer to run its
scripts directly, use that repository's own reference instead of this runbook:

- [Infra init reference](https://github.com/EMOBase/emobase-infra/blob/main/INIT.md)
- [Portal init reference](https://github.com/EMOBase/emobase-portal-deploy/blob/main/INIT.md)
- [Species init reference](https://github.com/EMOBase/emobase-deploy/blob/dev/INIT.md)
