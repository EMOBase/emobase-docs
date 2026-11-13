# Keycloak realms

EMOBase separates the Keycloak **server** from the application **realms**.

- `emobase-infra` runs Keycloak and owns its `master` realm, admin credentials
  and the `keycloak` database.
- The portal and species stacks create and own all application realms.
- Infra has no default realm, client, seed user or application database.

## Portal realm

The portal owns the first realm. `emobase-portal-deploy/init.sh` defaults to:

```dotenv
KEYCLOAK_REALM=emobase-portal
KEYCLOAK_CLIENT_ID=emobase-portal-client
```

The portal also generates `KEYCLOAK_CLIENT_SECRET` and the `coco` seed user.
These values live only in the portal `.env`.

## Species realms

A species can either join an existing realm or create a new one.

`emobase-deploy/init.sh` scans the portal and sibling species checkouts and
offers:

```text
1) Realm 'emobase-portal' — used by portal
2) Realm 'emobase-tcas' — used by Tcas
3) Create a new independent realm [emobase-dmel]
```

### Join an existing realm

Joining means the species uses the same users, roles and login session as the
realm owner. The script copies:

- `KEYCLOAK_REALM`
- `KEYCLOAK_CLIENT_ID`
- `KEYCLOAK_CLIENT_SECRET`
- `COCO_USERNAME`
- `COCO_PASSWORD`

It writes `KEYCLOAK_REALM_MODE=joined` and registers the species origin on the
realm's client.

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

## CLI flags

```sh
# Join the portal realm
emobase create species --keycloak-realm join:emobase-portal

# Create a new realm
emobase create species --keycloak-realm new:emobase-dmel
```

`--realm` is an alias for `--keycloak-realm` in `emobase create species`.

When `--yes` is used and no realm is passed, a species joins the first
discovered realm. On a species-only server with no existing realm, it creates a
new realm.

## Creating the realm

Realm creation is deferred until Keycloak is healthy. It is performed by:

- `emobase create portal` / `emobase create species`, when infra is running
- `emobase up`
- `emobase bootstrap`
- or manually with `emobase-infra/scripts/create-realm.sh`

`create-realm.sh` is idempotent. It creates or updates the realm, client,
origin and seed user, and keeps an existing seed password.

```bash
cd emobase-infra
./scripts/create-realm.sh \
  --realm emobase-tcas \
  --origin https://tcas.example.org \
  --client-id emobase-tcas-client \
  --client-secret "<KEYCLOAK_CLIENT_SECRET from the species .env>" \
  --coco-username coco \
  --coco-password "<COCO_PASSWORD from the species .env>" \
  --keep-password --quiet
```

`allow-origin.sh` only adds another origin to an existing client:

```bash
./scripts/allow-origin.sh \
  --realm emobase-portal \
  --client-id emobase-portal-client \
  https://dmel.example.org
```

## Credential ownership

The first checkout that creates a realm is its owner. Joining species copy the
owner's client credentials into their own `.env`.

`emobase doctor` reports a `realm.client_mismatch` or
`realm.secret_mismatch` when two checkouts claim the same realm but use
different credentials.

## Changing the Keycloak URL

`KEYCLOAK_PUBLIC_URL` is the OIDC issuer. Changing it changes the issuer for
every realm. Use `emobase domain` to update all dependent `.env` files and
restart the stacks. Existing login sessions become invalid.

## Migration from the old layout

Older deployments stored the realm/client/seed values in
`emobase-infra/.env`. To migrate:

1. Copy `KEYCLOAK_REALM`, `KEYCLOAK_CLIENT_ID`, `KEYCLOAK_CLIENT_SECRET`,
   `COCO_USERNAME` and `COCO_PASSWORD` from `emobase-infra/.env` into the portal
   `.env` (or the first species `.env` when there is no portal).
2. Add `KEYCLOAK_REALM_MODE=joined` to species that share the portal realm, or
   `KEYCLOAK_REALM_MODE=dedicated` to species with their own realm.
3. Remove those realm keys from `emobase-infra/.env`.
4. Keep the existing Keycloak database and realms; no realm recreation is
   required.
5. Run `emobase doctor`.

## Related references

- [Infra init reference](https://github.com/EMOBase/emobase-infra/blob/main/INIT.md)
- [Portal init reference](https://github.com/EMOBase/emobase-portal-deploy/blob/main/INIT.md)
- [Species init reference](https://github.com/EMOBase/emobase-deploy/blob/dev/INIT.md)
- [Full deployment runbook](DEPLOYMENT.md)
