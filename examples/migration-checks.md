# Migration checks

## Using [migration-checks-npm.yml](../.github/workflows/migration-checks-npm.yml) in callers

Runs three jobs against a knex project on `pull_request`:

- **lint-migrations** — merged migrations are immutable (no modify/rename/delete of a file already on the base branch), and any new migration must sort after the newest existing one.
- **migrate-roundtrip** — `migrate:latest` → `migrate:rollback` → `migrate:latest` against a fresh database, so broken or asymmetric `down()` functions surface here.
- **seeded-upgrade** — migrate and seed the database at the **base ref**, then apply only the PR's new migrations on top. This reproduces what deployment does (migrating a database that already has data), which an empty-database CI run never exercises.

`contents: read` is all that is required. The workflow runs on both `pull_request` and `push` callers: on `pull_request` it compares the PR base against the PR head, and on `push` (for example a `release.yml` on `main`) it compares the commit before the push against the pushed commit — so seeded-upgrade becomes a final "upgrade the previously deployed commit to this one" check. `migrate-roundtrip` runs on any event.

By default Postgres is started from the caller's own `docker-compose.yml` `postgres` service, so the CI database version matches what the application runs (and is kept current by renovate).

The seeded-upgrade job no-ops when there is no base to compare against (a first push to a new branch) or when the base commit has no migrations yet, so the change that introduces a repo's first migration passes cleanly; the job becomes active once at least one migration is on the base.

### Minimal (pull request)

```yaml
jobs:
  migration-checks:
    uses: digicatapult/shared-workflows/.github/workflows/migration-checks-npm.yml@main
    permissions:
      contents: read
```

### Final check on release (push to main)

Called from a `push`-triggered `release.yml`, seeded-upgrade migrates the previously deployed commit's schema and data, then applies the migrations introduced by this push on top — the same upgrade the deploy is about to perform.

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

For a repo with no postgres service in its compose file — or one whose compose file cannot be brought up in CI without extra bootstrap (for example bind-mounted TLS certs or required environment variables, since `docker compose up <service>` still validates the whole file) — provision a service container from an image instead. The CI version then no longer tracks the app, so pin it to match the compose image and prefer the compose service where one starts cleanly.

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
