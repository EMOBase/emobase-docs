# EMOBase documentation

Cross-repository documentation for EMOBase deployments. Each server runs its
own `emobase-infra` for the portal and species stacks deployed there.

| Document | Contents |
| --- | --- |
| [Deployment runbook](DEPLOYMENT.md) | Full empty-server → working-site procedure for `emobase-infra`, `emobase-portal-deploy` and `emobase-deploy` |

## Repositories

| Repository | Purpose |
| --- | --- |
| [`emobase-cli`](https://github.com/EMOBase/emobase-cli) | Optional `emobase` command that orchestrates the deployment repositories |
| [`emobase-infra`](https://github.com/EMOBase/emobase-infra) | MySQL and Keycloak, one stack per server |
| [`emobase-portal-deploy`](https://github.com/EMOBase/emobase-portal-deploy) | Multi-species portal, genomics API, JBrowse2 and BLAST |
| [`emobase-deploy`](https://github.com/EMOBase/emobase-deploy/tree/dev) | Single-species UI, Directus and species services (`dev` branch) |
| [`emobase-portal-web`](https://github.com/EMOBase/emobase-portal-web) | Portal web UI image source |
| [`emobase-web`](https://github.com/EMOBase/emobase-web) | Species web UI image source |
| [`emobase-genomics`](https://github.com/EMOBase/emobase-genomics) | Genomics API and worker image source |

Each deployment repository keeps its own README for stack-specific details. Use
this repository for the shared deployment order, URL hand-offs and Keycloak
realm strategy.

The deployment runbook uses the repositories' own `init.sh` and Docker Compose
commands directly. For the optional `emobase` CLI wrapper, see the
[emobase-cli README](https://github.com/EMOBase/emobase-cli#readme).
