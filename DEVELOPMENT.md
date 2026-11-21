# Local development environment

Run the whole EMOBase stack on one machine for local development, before any
DNS or TLS exists. Published images are used by default; [Build from source](#build-from-source)
explains how to replace one or more images with local builds.

Portal, Keycloak and species use `localhost` over plain HTTP on ports `9080`
(portal), `8181` (Keycloak) and `9090` (species).

## Start the stack

From an empty workspace directory:

```bash
emobase create infra --local
emobase create portal --local
emobase create species Tcas --local

emobase bootstrap
emobase status
```

`--local` sets every URL to `localhost`. The portal reads the Keycloak URLs and
MySQL password from the sibling `infra/` checkout; the species copies the portal
URL and derives `PORTAL_INTERNAL_URL` as `http://host.docker.internal:9080`, so
BLAST/SSR traffic stays internal. Docker Desktop provides `host.docker.internal`;
on Linux the compose files map it with `host-gateway`.

## Verify

Open `http://localhost:9080` for the portal and `http://localhost:9090` for the
species site. Keycloak admin is at `http://localhost:8181/keycloak/admin/`.

```bash
emobase doctor --live
emobase credential show portal
```

The seed password is temporary, so Keycloak asks for a new one on the first
sign-in. BLAST returns `502` until the first release has built databases.

## Build from source

The source repositories usually sit next to the deployment checkouts:

```text
workspace/
├── infra/
├── portal/
├── Tcas/
├── emobase-portal-web/
├── emobase-genomics/
└── emobase-web/
```

`infra/`, `portal/` and `Tcas/` come from the `--local` setup; the `emobase-*`
directories are source clones.

Each image has its own variable, so you can replace only the image you are
working on and keep the rest from `ghcr.io`.

| Working on | Image | Variable | Set in | Used by |
| --- | --- | --- | --- | --- |
| `emobase-genomics` | `emobase-genomics:local` | `EMOBASE_GENOMICS_IMAGE` | `portal/.env` | API, workers, one-shot jobs |
| `emobase-portal-web` | `emobase-portal-web:local` | `EMOBASE_PORTAL_WEB_IMAGE` | `portal/.env` | Portal UI |
| `emobase-web` | `emobase-web:local` | `EMOBASE_WEB_IMAGE` | `<species>/.env` | Species UI |

The pattern is the same for every image; only the source checkout, variable and
affected services differ.

### Replace one image

Run each recipe from the workspace root. Build the image, set the variable,
then recreate the services that use it.

#### Genomics

```bash
cd emobase-genomics
docker build -t emobase-genomics:local .
```

Set `EMOBASE_GENOMICS_IMAGE=emobase-genomics:local` in `portal/.env`, then:

```bash
cd ../portal
docker compose up -d genomicsservice genomicsworker-1 genomicsworker-2
docker compose --profile update run --rm migrate-genomics-db
docker compose --profile update run --rm migrate-genomics-es
```

The same image also runs `setup-jbrowse2-web`; rerun it with
`docker compose --profile setup run --rm setup-jbrowse2-web` if you changed the
JBrowse2 setup.

#### Portal UI

```bash
cd emobase-portal-web
docker build -t emobase-portal-web:local .
```

Set `EMOBASE_PORTAL_WEB_IMAGE=emobase-portal-web:local` in `portal/.env`, then:

```bash
cd ../portal
docker compose up -d fe
```

#### Species UI

```bash
cd emobase-web
docker build -t emobase-web:local .
```

Set `EMOBASE_WEB_IMAGE=emobase-web:local` in `Tcas/.env`, then:

```bash
cd ../Tcas
docker compose up -d fe
```

Rebuild the image and rerun the matching `docker compose up -d` command after a
source change. To replace both portal images, run the genomics and portal UI
recipes.

To return an image to the published version, remove its variable from `.env`
and run `emobase up --only portal` (or `emobase up --only Tcas`). While a local
tag is set, `emobase update` tries to pull it and fails; use `emobase up` or the
compose commands instead. `emobase status` and `emobase doctor` see the locally
built containers either way.

## Notes

- On Apple Silicon, enable Rosetta for `linux/amd64` before starting BLAST;
  otherwise the SequenceServer container never binds its port. See the
  [portal development guide](https://github.com/EMOBase/emobase-portal-deploy/blob/main/DEVELOPMENT.md).
- To run the scaffolding unattended, add `--yes` to each `create` command.
- For a local environment reachable from other machines, use non-local URLs when
  running `create`.

## Related documentation

- [Deployment guide](DEPLOYMENT.md) — deploy on a real server.
- [Operations](OPERATIONS.md) — add species, update, change URLs, remove.
- [Troubleshooting](TROUBLESHOOTING.md) — BLAST, JBrowse2 and login checks.
- [Portal development](https://github.com/EMOBase/emobase-portal-deploy/blob/main/DEVELOPMENT.md) — local image builds for the portal stack.
- [Species local development](https://github.com/EMOBase/emobase-deploy/blob/dev/README.md#local-development) — local image builds for the species stack.
