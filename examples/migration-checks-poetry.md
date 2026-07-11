# Migration checks (Poetry)

## Using [migration-checks-poetry.yml](../.github/workflows/migration-checks-poetry.yml) in callers

The Alembic sibling of [migration-checks-npm.yml](../.github/workflows/migration-checks.md). Runs three jobs against a Poetry + Alembic project on `pull_request`:

- **lint-migrations** — merged revisions are immutable (no modify/rename/delete of a version file already on the base branch), and the revision graph must have exactly **one head**. Alembic orders revisions by `down_revision`, not filename, so the failure mode equivalent to a knex "reorder" is two revisions branching from the same parent and producing multiple heads; `alembic upgrade head` then refuses to run.
- **migrate-roundtrip** — `alembic upgrade head` → `alembic downgrade base` → `alembic upgrade head` against a fresh database, exercising every `downgrade()`.
- **seeded-upgrade** — upgrade and seed at the **base ref**, then apply only the PR's new revisions on top, reproducing the deploy-time upgrade against existing data.

`contents: read` is all that is required. The job must run on a `pull_request` event; the lint and seeded-upgrade jobs compare against `github.base_ref` and self-skip when it is empty.

By default Postgres is started from the caller's own `docker-compose.yml` `postgres` service, so the CI database version matches what the application runs.

The seeded-upgrade job no-ops when the base ref has no revisions yet, so the PR that introduces a repo's first revision passes cleanly; the job becomes active once at least one revision is on the base branch.

### Minimal

```yaml
jobs:
  migration-checks:
    uses: digicatapult/shared-workflows/.github/workflows/migration-checks-poetry.yml@main
    permissions:
      contents: read
```

### With a pgvector service and SQL seeds

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
          PGPASSWORD=postgres psql -h localhost -U postgres -d spec-rag -v ON_ERROR_STOP=1 -f "$f"
        done
```
