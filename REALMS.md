# Keycloak realms

EMOBase separates the Keycloak **server** from the application **realms**.

- `emobase-infra` runs Keycloak and owns its `master` realm, admin credentials
  and the `keycloak` database.
- The portal and species stacks create and own all application realms.
- Infra has no default realm, client, seed user or application database.

`emobase create` scaffolds `.env` files and, when infra is already running, also
creates the portal database and registers realms. On a fresh server nothing is
created yet; `emobase bootstrap`, `emobase up` and `emobase sync` create or
extend the realms later. All of them use the credentials each checkout wrote to
its own `.env`.

## Portal realm

The portal owns the first realm. `emobase create portal` writes:

```dotenv
KEYCLOAK_REALM=emobase-portal
KEYCLOAK_CLIENT_ID=emobase-portal-client
```

It also generates `KEYCLOAK_CLIENT_SECRET` and the `coco` seed user. These
values live only in the portal `.env`. When the realm is created — by
`emobase create portal` with infra already running, or later by `emobase
bootstrap`, `up` or `sync` — the portal's `PUBLIC_SERVER` becomes its first
origin.

## Species realms

A species can either join an existing realm or create a new one.

```bash
# Join the portal realm: shared users and login
emobase create species --species Tcas --keycloak-realm join:emobase-portal

# Create a dedicated realm
emobase create species --species Tcas --keycloak-realm new:emobase-tcas
```

`--realm` is an alias for `--keycloak-realm`. Without either flag,
`emobase create species` asks with a menu built from the portal and sibling
species checkouts:

```text
🔐 Keycloak realm for this species:
   1) Realm 'emobase-portal' — used by portal
   2) Realm 'emobase-tcas' — used by Tcas
   3) Create a new independent realm [emobase-dmel]
```

With `--yes` and no realm flag, the species joins the first discovered realm.
On a species-only server with no existing realm, it creates a new one.

### Join an existing realm

Joining means the species uses the same users, roles and login session as the
realm owner. The CLI copies from the owner's `.env`:

- `KEYCLOAK_REALM`
- `KEYCLOAK_CLIENT_ID`
- `KEYCLOAK_CLIENT_SECRET`
- `COCO_USERNAME`
- `COCO_PASSWORD`

It writes `KEYCLOAK_REALM_MODE=joined`. When the realm is created (or already
exists), the species' `PUBLIC_SERVER` is registered as an additional origin on
the realm's client.

Only realms with locally available credentials can be joined. A realm that
exists only in Keycloak cannot be joined automatically because its client
secret and seed credentials are not available.

### Create a new realm

A new realm defaults to:

```dotenv
KEYCLOAK_REALM=emobase-<species>
KEYCLOAK_CLIENT_ID=emobase-<species>-client
KEYCLOAK_REALM_MODE=dedicated
```

The species generates its own client secret and seed password.

## When realms are created

| Command | Realm work |
| --- | --- |
| `emobase create portal` | Scaffold `.env`; create the portal database and realm if infra is already running |
| `emobase create species` | Scaffold `.env`; register the realm if Keycloak is already running |
| `emobase create species --apply` | Create the Directus database, register the realm origin, initialise Directus, start |
| `emobase bootstrap` | Create the portal realm, then create or extend each species realm |
| `emobase up` | Create or extend realms before starting each stack |
| `emobase sync` | Create missing databases and register missing origins without starting containers |

Realm creation is idempotent: it creates or updates the realm, client, origin
and seed user, and keeps an existing seed password. `emobase sync` never
deletes resources; `emobase prune` handles cleanup.

## Credential ownership

The first checkout that creates a realm is its owner. Joining species copy the
owner's client credentials into their own `.env`.

```bash
emobase doctor
```

reports `realm.client_mismatch` or `realm.secret_mismatch` when two checkouts
claim the same realm but use different credentials.

The CLI keeps a workspace ownership ledger at `<workspace>/.emobase/state.json`.
It records which databases and realms were created by which checkout.
`emobase prune` uses that ledger to clean up resources from removed stacks
without guessing from names. It skips shared realms that are still in use.

## Changing the Keycloak URL

`KEYCLOAK_PUBLIC_URL` is the OIDC issuer. Changing it changes the issuer for
every realm.

```bash
emobase domain plan  --auth https://auth.example.org/keycloak
emobase domain apply --auth https://auth.example.org/keycloak
```

`domain` updates every dependent `.env`, registers the new origins and restarts
the affected stacks. Existing login sessions become invalid. Old origins are
left in place; remove them in the Keycloak admin console if the cutover is
permanent.

On a species-only server, `domain` manages the local Keycloak and species URLs
and leaves `PORTAL_SERVER` untouched.

## Related references

- [Full deployment runbook](DEPLOYMENT.md)
- [Infra init reference](https://github.com/EMOBase/emobase-infra/blob/main/INIT.md)
- [Portal init reference](https://github.com/EMOBase/emobase-portal-deploy/blob/main/INIT.md)
- [Species init reference](https://github.com/EMOBase/emobase-deploy/blob/dev/INIT.md)
