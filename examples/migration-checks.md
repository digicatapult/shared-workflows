# Migration checks

## Using [migration-checks-npm.yml](../.github/workflows/migration-checks-npm.yml) in callers

Runs three jobs against a knex project on `pull_request`:

- **lint-migrations** — merged migrations are immutable (no modify/rename/delete of a file already on the base branch), and any new migration must sort after the newest existing one.
- **migrate-roundtrip** — `migrate:latest` → `migrate:rollback` → `migrate:latest` against a fresh database, so broken or asymmetric `down()` functions surface here.
- **seeded-upgrade** — migrate and seed the database at the **base ref**, then apply only the PR's new migrations on top. This reproduces what deployment does (migrating a database that already has data), which an empty-database CI run never exercises.

`contents: read` is all that is required. The job must run on a `pull_request` event: the lint and seeded-upgrade jobs compare against `github.base_ref`, and self-skip when it is empty.

By default Postgres is started from the caller's own `docker-compose.yml` `postgres` service, so the CI database version matches what the application runs (and is kept current by renovate).

The seeded-upgrade job no-ops when the base ref has no migrations yet, so the PR that introduces a repo's first migration passes cleanly; the job becomes active once at least one migration is on the base branch.

### Minimal

```yaml
jobs:
  migration-checks:
    uses: digicatapult/shared-workflows/.github/workflows/migration-checks-npm.yml@main
    permissions:
      contents: read
```

### Custom compose service and seed

Point the workflow at a specific compose service and provide smoke seed data so the seeded-upgrade job runs against representative rows.

```yaml
jobs:
  migration-checks:
    uses: digicatapult/shared-workflows/.github/workflows/migration-checks-npm.yml@main
    permissions:
      contents: read
    with:
      migrations_dir: db/migrations
      postgres_service: postgres
      seed_command: npm run db:seed
```

### Image fallback

For a repo with no postgres service in its compose file, provision a service container from an image instead. The CI version then no longer tracks the app, so prefer the compose service where one exists.

```yaml
jobs:
  migration-checks:
    uses: digicatapult/shared-workflows/.github/workflows/migration-checks-npm.yml@main
    permissions:
      contents: read
    with:
      postgres_image: pgvector/pgvector:pg18
      db_name: my-app
```
