# Backup and restore

A deployment is not one database. A complete backup has several independent
pieces, and a restore puts each one back:

| Layer | What it contains | Where it lives |
| --- | --- | --- |
| Configuration | `.env` files: secrets, URLs, database and realm names | each checkout |
| Shared data | Keycloak, portal genomics, Directus databases | MySQL in `infra` |
| Search data | Portal and species indices | Elasticsearch in `infra` |
| Files | Uploads, version files, species data and images | Bind mounts in the checkouts |
| Volumes | Directus uploads, JBrowse2 tracks, BLAST databases | Docker volumes |

MySQL rows and Elasticsearch documents point at files on disk, and the `.env`
files name the databases and realms. Restore all layers from the same backup and
keep the paths and database names unchanged.

JBrowse2 tracks and BLAST databases are derived from the uploaded files and
releases. Back them up to restore faster, or rebuild them.

## Back up

Stop the stacks first so nothing writes during the backup; then start only the
two shared databases:

```bash
emobase down
cd infra
docker compose up -d --wait mysql elasticsearch
```

1. Keep every `.env` file: `infra/.env`, `portal/.env` and each
   `<species>/.env`. They hold the secrets and the database/realm names the
   restore must match.

2. Take the MySQL dump and the Elasticsearch snapshot exactly as described in
   [`emobase-infra/BACKUP.md`](https://github.com/EMOBase/emobase-infra/blob/main/BACKUP.md).
   Both are point-in-time exports; after they finish you can start the stacks
   again.

3. Return to the workspace root and archive the bind-mounted files:

   ```bash
   cd ..
   tar czf portal-files.tar.gz -C portal data
   tar czf tcas-files.tar.gz -C Tcas data images
   ```

4. Archive the Docker volumes. The names are prefixed with the Compose project;
   list them with `docker volume ls`:

   ```bash
   # Repeat for blast-data and each directus-data volume.
   docker run --rm \
     -v emobase-portal_jbrowse2-data:/data \
     -v "$PWD":/backup \
     alpine sh -c 'cd /data && tar czf /backup/jbrowse2-data.tar.gz .'
   ```

5. Start everything again with `emobase up`.

## Restore

Restore is the reverse of each layer, from the same backup. The order matters:
the `.env` files name what the databases contain, and the database rows point at
the files and volumes restored afterwards.

1. Stop everything:

   ```bash
   emobase down
   ```

2. Restore the `.env` files first.

3. Restore MySQL and Elasticsearch with
   [`emobase-infra/BACKUP.md`](https://github.com/EMOBase/emobase-infra/blob/main/BACKUP.md).
   Run both its MySQL and Elasticsearch sections; they start only those
   services and leave the application stacks stopped.

4. From the workspace root, restore the bind-mounted files:

   ```bash
   cd ..
   tar xzf portal-files.tar.gz -C portal
   tar xzf tcas-files.tar.gz -C Tcas
   ```

5. Restore the Docker volumes before starting the stacks that use them:

   ```bash
   # Repeat for blast-data and each directus-data volume.
   docker run --rm \
     -v emobase-portal_jbrowse2-data:/data \
     -v "$PWD":/backup \
     alpine sh -c 'cd /data && tar xzf /backup/jbrowse2-data.tar.gz'
   ```

6. Start in dependency order and verify:

   ```bash
   emobase up
   emobase doctor --live
   ```

## Consistency and versions

- Use the MySQL dump, Elasticsearch snapshot and file archives from the same
  backup. A newer MySQL with an older Elasticsearch leaves dangling references.
- The realm, Directus and client secrets in the `.env` files must match the
  restored Keycloak and Directus data; restore them from the same backup.
- Restore with the same image tags as the backup. If you need to upgrade, do it
  after the restore with `emobase update`.
- Test a restore on a throwaway server before relying on it.
