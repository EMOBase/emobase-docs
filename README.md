# EMOBase documentation

Cross-repository documentation for EMOBase deployments.

| Document | Contents |
| --- | --- |
| [Deployment runbook](DEPLOYMENT.md) | Full empty-server → working-site procedure for `emobase-infra`, `emobase-portal-deploy` and `emobase-deploy` |

## Repositories

| Repository | Purpose |
| --- | --- |
| [`emobase-infra`](https://github.com/EMOBase/emobase-infra) | Shared MySQL and Keycloak |
| [`emobase-portal-deploy`](https://github.com/EMOBase/emobase-portal-deploy) | Multi-species portal, genomics API, JBrowse2 and BLAST |
| [`emobase-deploy`](https://github.com/EMOBase/emobase-deploy) | Single-species UI, Directus and species services (`dev` branch) |
| [`emobase-portal-web`](https://github.com/EMOBase/emobase-portal-web) | Portal web UI image source |
| [`emobase-web`](https://github.com/EMOBase/emobase-web) | Species web UI image source |
| [`emobase-genomics`](https://github.com/EMOBase/emobase-genomics) | Genomics API and worker image source |

Each deployment repository keeps its own README for stack-specific details. Use
this repository for the shared deployment order, URL hand-offs and Keycloak
realm strategy.
