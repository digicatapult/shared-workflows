# Migration checks (Poetry)

## Using [migration-checks-poetry.yml](../.github/workflows/migration-checks-poetry.yml) in callers

The Alembic sibling of [migration-checks-npm.yml](../.github/workflows/migration-checks-npm.yml). Runs three jobs against a Poetry + Alembic project:

- **lint-migrations** — merged revisions are immutable (no modify/rename/delete of a version file already on the base branch), and the revision graph must have exactly **one head**. Alembic orders revisions by `down_revision`, not filename, so the failure mode equivalent to a knex "reorder" is two revisions branching from the same parent and producing multiple heads; `alembic upgrade head` then refuses to run.
- **migrate-roundtrip** — `alembic upgrade head` → `alembic downgrade base` → `alembic upgrade head` against a fresh database, exercising every `downgrade()`.
- **seeded-upgrade** — upgrade and seed at the **base ref**, then apply only the PR's new revisions on top, reproducing the deploy-time upgrade against existing data.

`contents: read` is all that is required. The workflow runs on both `pull_request` and `push` callers: on `pull_request` it compares the PR base against the PR head, and on `push` (for example a `release.yml` on `main`) it compares the commit before the push against the pushed commit — so seeded-upgrade becomes a final "upgrade the previously deployed commit to this one" check. The single-head lint and `migrate-roundtrip` run on any event; the immutability lint and seeded-upgrade need a base to compare against.

By default Postgres is started from the caller's own `docker-compose.yml` `postgres` service, so the CI database version matches what the application runs.

The seeded-upgrade job no-ops when there is no base to compare against or when the base commit has no revisions yet, so the change that introduces a repo's first revision passes cleanly; the job becomes active once at least one revision is on the base.

### Minimal

```yaml
jobs:
  migration-checks:
    uses: digicatapult/shared-workflows/.github/workflows/migration-checks-poetry.yml@main
    permissions:
      contents: read
```

### With SQL seeds

Seeds run after `alembic upgrade head` at the base ref, so they must only
target tables that the migrations create. Data for tables created at
application runtime (for example a vector store table auto-created by an ORM
or embedding library) will not exist yet and should not be seeded here.

```yaml
jobs:
  migration-checks:
    uses: digicatapult/shared-workflows/.github/workflows/migration-checks-poetry.yml@main
    permissions:
      contents: read
    with:
      postgres_service: postgres
      seed_command: |
        for f in seeds/seed_*.sql; do
          PGPASSWORD=postgres psql -h localhost -U postgres -d my-app -v ON_ERROR_STOP=1 -f "$f"
        done
```
