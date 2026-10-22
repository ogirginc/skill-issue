---
name: schema-healer
description: Fix orphaned database migrations and schema.rb changes from abandoned branches. Use when schema.rb shows unwanted tables/columns from deleted migration files, or when "git diff db/schema.rb" shows ghost changes that keep reappearing after db:migrate.
---

# Schema Healer

Fix schema.rb pollution from abandoned branches where migrations were run but never rolled back before the branch was deleted.

## When to Use

- `git diff db/schema.rb` shows unwanted tables or columns
- Running `db:migrate` keeps adding ghost changes to schema.rb
- You abandoned a branch without rolling back its migrations
- Schema.rb has tables from gems you uninstalled

## Diagnosis Flow

```
git diff db/schema.rb shows changes
           │
    Identify added tables/columns
           │
    Check if migration files exist
    for those changes
           │
     ┌─────┴─────┐
   Exist      Don't exist
     │            │
  Normal      Orphaned!
  changes     Need healing
```

## Step 1: Identify Orphaned Changes

First, check what's different in schema.rb:

```bash
git diff db/schema.rb
```

Look for `create_table` blocks or `add_column` lines that were added (marked with `+`).

## Step 2: Find Orphaned Migration Versions

Rails already flags migrations that exist in the database but have no file on disk:

```bash
bin/rails db:migrate:status | grep "NO FILE"
```

Each line is an orphaned version, shown as `up  20260123145226  ********** NO FILE **********`.

## Step 3: Confirm Before Removing Anything

The next steps are destructive. Before running them:

1. Check the environment is local development — `bin/rails runner "puts Rails.env"`
   must print `development`, and check `DATABASE_URL` (if set) does not point at a
   shared database.
2. Cross-check Steps 1 and 2: only drop tables/columns whose changes correspond to
   the orphaned versions from Step 2. A table in the diff could come from a real,
   not-yet-committed migration — leave those alone.
3. Show the user the exact list of tables, columns, and migration versions to be
   removed, and get explicit confirmation before proceeding.

## Step 4: Drop Orphaned Tables

For each confirmed orphaned table:

```bash
bin/rails runner "ActiveRecord::Migration.drop_table(:table_name)"
```

## Step 5: Remove Orphaned Columns

If only columns were added (not full tables):

```bash
bin/rails runner "ActiveRecord::Migration.remove_column(:table_name, :column_name)"
```

## Step 6: Clean Up schema_migrations

Remove the orphaned migration version records:

```bash
bin/rails runner "ActiveRecord::Base.connection.execute(\"DELETE FROM schema_migrations WHERE version IN ('VERSION1', 'VERSION2')\")"
```

## Step 7: Regenerate Schema

Dump a fresh schema from the cleaned database:

```bash
bin/rails db:schema:dump
```

## Step 8: Verify

Confirm schema.rb is now clean:

```bash
git diff db/schema.rb
git status db/schema.rb
```

Should show no changes.

## Example: Full Workflow

```bash
# 1. See what's polluting schema.rb
git diff db/schema.rb
# Output shows: + create_table "ghost_table" ...

# 2. Find orphaned migrations
bin/rails db:migrate:status | grep "NO FILE"
# Output: up  20260123145226  ********** NO FILE **********

# 3. Confirm environment and get user sign-off, then drop the ghost table
bin/rails runner "ActiveRecord::Migration.drop_table(:ghost_table)"

# 4. Remove orphaned migration record
bin/rails runner "ActiveRecord::Base.connection.execute(\"DELETE FROM schema_migrations WHERE version = '20260123145226'\")"

# 5. Regenerate schema
bin/rails db:schema:dump

# 6. Verify clean
git diff db/schema.rb  # Should be empty
```

## Common Scenarios

### Abandoned gem migrations (e.g., console1984, devise)

```bash
# Drop all tables from the gem
bin/rails runner "ActiveRecord::Migration.drop_table(:console1984_sessions)"
bin/rails runner "ActiveRecord::Migration.drop_table(:console1984_users)"
# ... drop all related tables

# Clean up migration records
bin/rails runner "ActiveRecord::Base.connection.execute(\"DELETE FROM schema_migrations WHERE version LIKE '202601231%'\")"
```

### Single orphaned column

```bash
bin/rails runner "ActiveRecord::Migration.remove_column(:users, :abandoned_field)"
```

## Other Schema Formats and Multiple Databases

- If the app uses `config.active_record.schema_format = :sql`, the polluted file is
  `db/structure.sql` instead of `db/schema.rb`. The same workflow applies; diff and
  dump that file instead.
- Rails apps with multiple databases (e.g. Rails 8 with solid_queue / solid_cache /
  solid_cable) have separate schema files like `db/queue_schema.rb` and
  `db/cache_schema.rb`, each with its own `schema_migrations` table. Check
  `git status db/` for all changed schema files and run the workflow per database
  (`bin/rails db:migrate:status:queue`, etc.).

## Safety Notes

- Always check `git diff` first to understand what needs removal
- Only drop tables/columns whose migration versions are confirmed orphaned (Step 3)
- Never run this against a shared or production database — verify `Rails.env` and
  `DATABASE_URL` first, and get explicit user confirmation before dropping anything
- If unsure, `bin/rails db:reset` is the nuclear option — it wipes all local data, so ask first
