# Troubleshooting

Start with the CLI, then use the owning checkout's README for stack details:

```bash
emobase status
emobase doctor --live
emobase domain plan
```

For updates, URL changes and cleanup, see [OPERATIONS.md](OPERATIONS.md). For
backups before a destructive fix, see [BACKUP.md](BACKUP.md).

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
