---
name: schema-healer
description: Fix orphaned Rails migrations and schema.rb / structure.sql pollution from abandoned branches. Use when db/schema.rb or db/structure.sql shows tables, columns or indexes nobody added on this branch, when "git diff db/schema.rb" shows ghost changes that reappear after db:migrate, or when db:migrate:status lists migrations as "NO FILE".
---

# Schema Healer

Fix schema pollution from abandoned branches: their migrations ran against the local database, then the branch (and migration files) went away without rolling back.

## Step 1: Find the orphaned versions

```bash
git status db/
git diff db/schema.rb
bin/rails db:migrate:status | grep "NO FILE"
```

Each `NO FILE` line (`up  20260123145226  ********** NO FILE **********`) is an orphaned version. Work only from this list: a change in the diff that doesn't trace back to one of these versions may come from a real, uncommitted migration — leave it alone.

If the app uses `config.active_record.schema_format = :sql`, the polluted file is `db/structure.sql`; use it wherever this skill says `schema.rb`.

## Step 2: Check it's safe

1. `bin/rails runner "puts Rails.env"` must print `development`.
2. `DATABASE_URL`, if set, must not point at a shared database.
3. Show the user the orphaned versions and the schema changes you believe each one made. Get explicit confirmation before changing anything.

## Step 3: Undo each orphaned version

### Preferred: recover the migration file and roll it back

A deleted branch's commits are often still on the remote or in the reflog. Look for the file:

```bash
git log --all --reflog --diff-filter=A --name-only --format=%h -- 'db/migrate/<VERSION>_*'
```

If found, restore it, roll it back, and delete it again:

```bash
git show <commit>:db/migrate/<VERSION>_<name>.rb > db/migrate/<VERSION>_<name>.rb
bin/rails db:migrate:down VERSION=<VERSION>
rm db/migrate/<VERSION>_<name>.rb
```

`db:migrate:down` reverses exactly what the migration did and removes its `schema_migrations` row, so skip to Step 4. If `down` raises `ActiveRecord::IrreversibleMigration`, fall through to the manual path.

### Fallback: reverse it by hand

Use this only when the file can't be recovered, and only for changes you can see in the schema diff:

| Diff shows | Undo with |
|---|---|
| `create_table "x"` | `drop_table(:x)` |
| `t.<type> "col"` added to an existing table | `remove_column(:table, :col)` |
| `t.index` / `add_index` | `remove_index(:table, name: "index_name")` |
| `add_foreign_key` | `remove_foreign_key(:from_table, :to_table)` |
| `enable_extension "x"` | `disable_extension("x")` — check nothing else uses it |

Run each through the connection:

```bash
bin/rails runner 'ActiveRecord::Base.connection.drop_table(:ghost_table)'
```

If `drop_table` fails because another table has a foreign key to it, remove that foreign key first. Use `drop_table(:x, force: :cascade)` only after confirming the referencing table is orphaned too.

**Stop and ask the user** if the diff shows a changed column type, default, null constraint, rename, or anything else that modified something that already existed. The original state can't be inferred from the diff alone, and the database may hold data the migration changed. Offer the reset in "Last resort" instead.

Then delete the orphaned versions — the exact versions from Step 1, never a `LIKE` pattern:

```bash
bin/rails runner "ActiveRecord::Base.connection.execute(\"DELETE FROM schema_migrations WHERE version IN ('20260123145226', '20260123145301')\")"
```

## Step 4: Regenerate and verify

```bash
bin/rails db:schema:dump
git diff db/schema.rb
bin/rails db:migrate:status | grep "NO FILE"
```

Both should come back empty, apart from changes from this branch's own migrations.

## Multiple databases

Rails apps with several databases (e.g. solid_queue, solid_cache, solid_cable) have one schema file and one `schema_migrations` table per database: `db/queue_schema.rb`, `db/cache_schema.rb`, etc., with migrations in `db/queue_migrate/` and so on. Run the workflow once per polluted file, using the database's name from `config/database.yml`:

- `bin/rails db:migrate:status:queue`, `bin/rails db:migrate:down:queue VERSION=…`, `bin/rails db:schema:dump:queue`
- Search the right migrations folder when recovering files: `-- 'db/queue_migrate/<VERSION>_*'`
- For manual steps, connect to that database first:

  ```bash
  bin/rails runner 'ActiveRecord::Base.establish_connection(:queue); ActiveRecord::Base.connection.drop_table(:ghost_table)'
  ```

## Last resort: rebuild from the committed schema

When the pollution is too tangled to undo, and with the user's OK — this wipes all local data in that database:

```bash
git checkout db/schema.rb
bin/rails db:schema:load
```

Restore the schema file first: `db:reset` and `db:schema:load` load whatever schema file is on disk, so running them on the polluted file recreates the ghost tables. Afterwards, re-run `bin/rails db:migrate` for this branch's own migrations and re-seed if needed.
