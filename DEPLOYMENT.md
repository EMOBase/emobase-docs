# EMOBase deployment guide

This guide takes an empty Linux server to a working portal and species site
using the `emobase` command. Follow the five deployment steps in order.

`emobase` orchestrates three repositories:

| Repository | What it runs | Default host port |
| --- | --- | --- |
| [`emobase-infra`](https://github.com/EMOBase/emobase-infra) | MySQL, Keycloak and Elasticsearch for all stacks on this server | `8181` (Keycloak), `9202` (Elasticsearch, loopback only) |
| [`emobase-portal-deploy`](https://github.com/EMOBase/emobase-portal-deploy) | Portal UI, genomics API/workers, JBrowse2, BLAST | `9080` |
| [`emobase-deploy`](https://github.com/EMOBase/emobase-deploy/tree/dev) | Single-species UI and Directus (`dev` branch) | `9090` |

It clones the checkouts, calls each repository's own `init.sh`, and runs their
Compose commands in the right order. Every checkout keeps its own `.env` and
secrets; the CLI never stores or copies them.

On an empty server, prepare with [What you need](#what-you-need) and
[Install the CLI](#install-the-cli), then follow
[Deploy on a server](#deploy-on-a-server).

- Only want a local development environment? Use [DEVELOPMENT.md](DEVELOPMENT.md).
- Do not want the CLI? Use the repository references in [Without the CLI](#without-the-cli).

## Before you start

### What you need

- Linux server with Docker Engine 24+ and Compose v2
- `git` and `python3` (the Keycloak helpers use it)
- Outbound HTTPS to `ghcr.io`, Docker Hub and `quay.io`
- DNS names (or one name plus ports) for the portal, each species site and
  Keycloak
- For production, a TLS reverse proxy in front of the Keycloak, portal and
  species ports, forwarding `Host` and `X-Forwarded-*`

Check Docker:

```bash
docker version
docker compose version
```

### Install the CLI

Install the latest stable release:

```bash
curl -fsSL https://raw.githubusercontent.com/EMOBase/emobase-cli/main/install.sh | sh
emobase --version
```

The script detects Linux/macOS and `amd64`/`arm64`, verifies the checksum and
installs `emobase` in `/usr/local/bin` (using `sudo` only when needed). For a
pinned version, a per-user install, the rolling `main` build or a source build
with Go 1.25+, see
[`emobase-cli/INSTALL.md`](https://github.com/EMOBase/emobase-cli/blob/main/INSTALL.md).

## Deploy on a server

The steps below are the same for one server that runs the portal and species,
and for a species-only server (skip step 2).

Each `create` command prompts for its values and shows the default; press Enter
to accept it. The species code is the only required argument.

Run the steps from a workspace directory you can write to; it will hold the
checkouts. Any path works.

### Step 1: Create the infrastructure

```bash
emobase create infra
```

Clones the `emobase-infra` repository into `infra/` and writes its `.env`,
including generated MySQL root and Keycloak admin passwords. It asks for the
shared network, the host ports and the browser-facing Keycloak URL. The URL
defaults to the detected host IP; press Enter to accept it, or enter the public
domain or IP when the server is behind NAT.

Review the `.env` before the first start: this file is the authority for the
shared secrets, and every other stack reads them from it. No realm, database or
index is created here.

### Step 2: Create the portal (skip on a species-only server)

```bash
emobase create portal
```

`emobase create portal` runs the portal `init.sh`, which reads the Keycloak URLs
and MySQL password from the sibling `infra/` checkout. The prompts ask for the
nginx port, the public portal URL, the realm, the client ID and the seed user.
The public URL defaults to the detected host IP; press Enter to accept it, or
enter the public domain or IP.

When infra is running, the CLI then creates the portal database, realm, client
and seed user. Review `PUBLIC_SERVER` and `HTTP_PORT`, and pin
`EMOBASE_GENOMICS_IMAGE` / `EMOBASE_PORTAL_WEB_IMAGE` to a release tag rather
than a moving branch tag.

`PUBLIC_SERVER` must be the exact URL users open **and** a host the web app
accepts: `security.allowedDomains` in `emobase-portal-web` currently allows
`**.uni-goettingen.de`. Any other domain fails login with
`403 Cross-site POST form submissions are forbidden` until it is added there and
a new image is published.

### Step 3: Create the species

```bash
emobase create species Tcas
```

The species code is required. It copies `PORTAL_SERVER` from the sibling portal
checkout and derives `PORTAL_INTERNAL_URL`, then asks for the web UI port, the
public species URL, the Directus database, the container and index prefixes and
the realm. The public URL defaults to the detected host IP.

On a species-only server there is no sibling portal, so it asks for the portal
origin as well. Alternatively, pass
`--portal-server https://portal.example.org` and answer the other prompts.
The species nginx proxies `/blast/` to the portal, so `PORTAL_SERVER` must be
reachable from this server.

At the realm menu, choose the portal realm to share users, or create a dedicated
realm. Non-interactive forms: `--keycloak-realm join:emobase-portal` and
`--keycloak-realm new:emobase-tcas`. The resulting variables are listed in
[What `create` writes](https://github.com/EMOBase/emobase-cli/blob/main/COMMANDS.md#what-create-writes).

### Step 4: Bootstrap

```bash
emobase bootstrap
```

Runs infra → portal (JBrowse2 setup, migrations, shared realm, smoke test) →
each species (Directus database, realm, initialise, start). `--dry-run` prints
every command; `--only infra|portal|species|<stack>`, `--no-pull`,
`--skip-setup`, `--skip-migrations` and `--skip-smoke` narrow it. A stack can be
named by species code, checkout name or Compose project, for example
`emobase bootstrap --only Tcas`.

The portal migrations must run before the UI is used; without them the API
returns `500`.

### Step 5: Verify

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

## Before you go live

- [ ] `emobase doctor` reports no errors, and `doctor --live` passes
- [ ] `KEYCLOAK_PUBLIC_URL` and every site URL are final (TLS proxy in place)
- [ ] Image tags are pinned to releases, not moving branch tags
- [ ] `.env` files and the data volumes are covered by a backup plan
- [ ] JBrowse2 and BLAST work after the first release
- [ ] Sign-in works on the portal and on each species site

For updates, URL changes, more species and cleanup, see
[OPERATIONS.md](OPERATIONS.md). For backups, see [BACKUP.md](BACKUP.md).

## Without the CLI

Each repository remains independently deployable. If you prefer to run its
scripts directly, use that repository's own reference instead of this guide:

- [Infra init reference](https://github.com/EMOBase/emobase-infra/blob/main/INIT.md)
- [Portal init reference](https://github.com/EMOBase/emobase-portal-deploy/blob/main/INIT.md)
- [Species init reference](https://github.com/EMOBase/emobase-deploy/blob/dev/INIT.md)
