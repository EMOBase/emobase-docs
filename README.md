# EMOBase documentation

EMOBase is a multi-species genomics platform: one portal hosts database versions
and their per-species assemblies, and each species site serves its own UI and
Directus content. MySQL, Keycloak and Elasticsearch run once per server in
`emobase-infra`; the portal and the species stacks share them.

This repository holds the cross-repository documentation for EMOBase
deployments. Each repository keeps its own README and `INIT.md`; use those for
one stack without the CLI.

| Document | Contents |
| --- | --- |
| [Deployment guide](DEPLOYMENT.md) | Empty server → working site using the `emobase` command |
| [Local development](DEVELOPMENT.md) | Run the stack locally, with published or locally built images |
| [Operations](OPERATIONS.md) | Add a species, update, change URLs, sync, stop/start, remove, prune |
| [Backups](BACKUP.md) | What to back up across the stacks and the restore order |
| [Troubleshooting](TROUBLESHOOTING.md) | Symptoms, checks and reset caveats |
| [Realm model](REALMS.md) | Portal/species realm ownership, sharing and credential checks |
| [Infra init reference](https://github.com/EMOBase/emobase-infra/blob/main/INIT.md) | Standalone `emobase-infra/init.sh` prompts, flags and output |
| [Portal init reference](https://github.com/EMOBase/emobase-portal-deploy/blob/main/INIT.md) | Standalone `emobase-portal-deploy/init.sh` prompts, flags and output |
| [Species init reference](https://github.com/EMOBase/emobase-deploy/blob/dev/INIT.md) | Standalone `emobase-deploy/init.sh` prompts, realm menu and output |

## Repositories

| Repository | Purpose |
| --- | --- |
| [`emobase-cli`](https://github.com/EMOBase/emobase-cli) | `emobase` command that orchestrates the deployment repositories |
| [`emobase-infra`](https://github.com/EMOBase/emobase-infra) | MySQL, Keycloak and Elasticsearch, one stack per server |
| [`emobase-portal-deploy`](https://github.com/EMOBase/emobase-portal-deploy) | Multi-species portal, genomics API, JBrowse2 and BLAST |
| [`emobase-deploy`](https://github.com/EMOBase/emobase-deploy/tree/dev) | Single-species UI, Directus and species services (`dev` branch) |
| [`emobase-portal-web`](https://github.com/EMOBase/emobase-portal-web) | Portal web UI image source |
| [`emobase-web`](https://github.com/EMOBase/emobase-web) | Species web UI image source |
| [`emobase-genomics`](https://github.com/EMOBase/emobase-genomics) | Genomics API and worker image source |

The deployment guide uses `emobase` for the cross-repository flow. Each
deployment repository keeps its own README and `INIT.md`; use those when you
want to run a stack without the CLI.
