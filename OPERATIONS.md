# Operations

Day-2 tasks for an existing `emobase` workspace. For the first deployment, see
[DEPLOYMENT.md](DEPLOYMENT.md). For the full command list, see the
[`emobase-cli` command reference](https://github.com/EMOBase/emobase-cli/blob/main/COMMANDS.md#commands).

Run `emobase` from the workspace, or pass `--workspace DIR`. Commands discover
the checkouts from the current directory or its parent.

## Inspect

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

## Add another species

Each species has its own checkout, `.env`, Directus database/data and
containers. Give each stack a distinct HTTP port:

```bash
emobase create species Dmel \
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

## Update a deployment

```bash
emobase update
```

This pulls each checkout and its images, runs the portal migrations, restarts
the stacks and (unless `--skip-smoke`) runs the portal smoke test. The order is
infra → portal → species, because the portal may add database columns or API
fields that the species UI expects.

Useful flags: `--only infra|portal|species|<stack>`, `--no-git`,
`--skip-migrations`, `--skip-smoke`, `--dry-run`. For example,
`emobase update --only Tcas` updates one species stack.

A plain `up` does not re-pull a tag that already exists locally; `update` (or
`up --pull`) does. To roll back, set the previous image tag in the relevant
`.env` and run `emobase up` — or `emobase update --no-git` after pinning.

## Change a public URL

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

## Reconcile after manual edits

```bash
emobase sync --dry-run
emobase sync
```

`sync` creates missing portal/species databases, registers missing realm
origins, and records them in the ownership ledger. It does not pull images,
run migrations or start containers.

## Stop and start

```bash
emobase down
emobase up
emobase up --pull
```

`down` stops species → portal → infra; `up` starts infra → portal → species.

## Remove a species

```bash
emobase species remove Tcas
```

Without `--purge` this only prints the commands it would run. With
`--purge --yes` it stops the stack, deletes its Compose volumes, drops the
Directus database, deletes its `<prefix>-*` Elasticsearch indices and removes
the checkout directory (keep it with `--keep-dir`). A dedicated realm is
cleaned up by `emobase prune`, which only touches resources recorded in the
ownership ledger and skips shared realms that are still in use.

## Prune removed stacks

```bash
emobase prune
emobase prune --yes
```

`prune` removes databases, dedicated realms and Elasticsearch index prefixes
that the ownership ledger records for checkouts that are gone. It never guesses
from names and never touches resources that are not recorded. Shared realms and
shared index prefixes are skipped while an active checkout still uses them. See
the [`emobase-cli` README](https://github.com/EMOBase/emobase-cli#ownership-ledger)
for the ledger contents.
