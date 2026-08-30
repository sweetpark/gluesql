# file-storage: Crash-safe data and schema writes

## Problem

`storages/file-storage` writes both row data (`.ron` files) and table schema
(`.sql` files) by calling `File::create` followed by `write_all`. This
overwrites the target file in place. If the process crashes, the disk fills
up, or the write otherwise fails partway through, the target file is left
truncated or containing partially written bytes. A subsequent read of that
table (schema parse or row deserialization) then fails or returns corrupt
data, with no way to recover the previous, valid contents.

This affects:

- `StoreMut::insert_data` (`storages/file-storage/src/store_mut.rs:58`)
- `StoreMut::append_data` (`storages/file-storage/src/store_mut.rs:44`)
- `StoreMut::insert_schema` (`storages/file-storage/src/store_mut.rs:17`)

`StoreMut::delete_data` is not affected: it only calls `fs::remove_file`,
and `unlink` on POSIX filesystems does not produce a partially-deleted file.

`storages/file-storage/src/migration.rs` already solves this same problem
for migration writes with `write_file_atomically`: write to a temp file,
`fsync`, back up any existing target, then `rename` the temp file into
place (rolling back the rename if a later step fails). This design reuses
that mechanism for the normal write path instead of only the migration path.

## Non-goals

- **Statement-level atomicity across multiple files.** `insert_data` can
  write several row files in one call; a failure partway through today (and
  after this change) can still leave some rows written and others not. Fixing
  this requires a journal/WAL and is tracked as a separate follow-up.
- **Startup recovery of leftover temp/backup files.** A crash between the
  temp-file write and the final rename can leave a stray `.tmp-*` or
  `.bak-*` file on disk. These files are inert (never read back) but not
  cleaned up. Cleanup on `FileStorage::new` is a separate follow-up.
- **csv-storage / json-storage.** These are independent crates with their
  own copies of the same `File::create` + `write_all` pattern. They are not
  touched here and get their own follow-up PRs.

## Approach

Extract the atomic-write primitive out of `migration.rs` into a new module,
`storages/file-storage/src/atomic_write.rs`:

- `write_file_atomically(path: &Path, data: &str) -> Result<()>`
- `temp_path_for(path: &Path) -> PathBuf`
- `backup_path_for(path: &Path) -> PathBuf`

These move verbatim (no behavior change). `migration.rs` imports them from
the new module instead of defining them locally.

`store_mut.rs` then calls `write_file_atomically` instead of
`File::create` + `write_all` in:

- `insert_schema` (schema `.sql` file)
- `append_data` (each row `.ron` file)
- `insert_data` (each row `.ron` file)

`delete_data` is left unchanged.

## Testing

Add tests (co-located with `store_mut.rs` or in `storages/file-storage/tests`,
matching existing test layout) covering:

1. A normal `insert_data` write succeeds and leaves no `.tmp-*` or `.bak-*`
   file behind.
2. Inserting a row whose file already exists (re-insert / overwrite path)
   replaces the content and leaves no stray temp/backup file after success.
3. Simulating a write failure (e.g. an unwritable target directory) leaves
   the original file's contents untouched when one already existed.

Also run per `AGENTS.md`: `cargo clippy --all-targets -- -D warnings`,
`cargo fmt --all`, and the file-storage test suite.

## Rollout

Single PR against `gluesql/gluesql`, opened directly (no precursor issue),
following the project's existing convention for self-contained, clearly
scoped fixes (e.g. PR #1980, #1972). Follow-up work (WAL for statement
atomicity, startup recovery, csv/json-storage) is proposed as separate
issues first, since those involve design tradeoffs the maintainers should
weigh in on before implementation.
