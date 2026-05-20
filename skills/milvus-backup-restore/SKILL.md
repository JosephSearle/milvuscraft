---
name: milvus-backup-restore
description: >
  Back up and restore Milvus collections. Load this skill whenever the user mentions: backup
  Milvus, restore Milvus, disaster recovery, milvus-backup CLI, data migration, cross-environment
  copy, promote from test to staging, promote to production, copy a collection between
  environments, or what happens if Milvus goes down. Always load milvus-context first.
---

# Milvus Backup & Restore Skill

Collection backup and restore using the `milvus-backup` open-source CLI, cross-environment
promotion patterns, and disaster-recovery discipline. No MCP tools exist for backup — all
operations use CLI or PyMilvus SDK.

---

## Core Philosophy

Untested backups are not backups. Every backup procedure ends with a test restore to a
throwaway collection and a sample search to confirm data integrity. Schedule quarterly
restore tests even when nothing has changed.

---

## Step 1 — Check for managed-service backup

Check milvus-context → Deployment overrides for any native backup tooling provided by your
cloud provider. If a managed backup API is available, prefer it over the open-source CLI.
If no override exists, use the `milvus-backup` CLI as documented below.

---

## Step 2 — Install and configure milvus-backup CLI

Pin to **≥v0.5.11** (fixes a FlushAll metadata corruption bug affecting Milvus ≥2.6.9;
safe to pin even on earlier Milvus versions).

```bash
go install github.com/zilliztech/milvus-backup@v0.5.11
```

Configure `backup.yaml` — see `references/backup-yaml-template.md` for a fully-commented
template covering Milvus gRPC endpoint and all object storage backends (MinIO, S3, GCS,
Azure Blob).

---

## Step 3 — Create a backup

```bash
# Backup a single collection
./milvus-backup create -c my_collection -n my_backup_20260520

# Backup all collections in a database
./milvus-backup create -n full_backup_20260520

# Metadata-only backup (fast; schema recovery only)
./milvus-backup create -c my_collection -n meta_backup --meta_only
```

Verify the backup was created successfully:

```bash
./milvus-backup list
```

Confirm the backup name appears with `status: success`.

---

## Step 4 — Test restore (mandatory after every backup)

```bash
# Restore with a suffix to avoid name collision with the live collection
./milvus-backup restore -n my_backup_20260520 -s _test
```

Then load and search the restored collection:

```python
client.load_collection("my_collection_test")
results = client.search(
    "my_collection_test",
    data=[<known_query_vector>],
    limit=1,
    output_fields=["id"],
)
assert results[0][0]["id"] == <expected_id>
client.release_collection("my_collection_test")
client.drop_collection("my_collection_test")
```

---

## Step 5 — Restore in production

**Same-instance restore:**

```bash
./milvus-backup restore -n my_backup_20260520 -s _restored
```

Appends suffix to avoid collision with the live collection. Load and alias-swap
(see milvus-collection-lifecycle) when ready.

**Cross-collection rename:**

```bash
./milvus-backup restore -n my_backup_20260520 \
  -r source_db.source_coll:target_db.target_coll
```

---

## Step 6 — Cross-environment promotion (test → staging → production)

1. Back up from the source environment (Step 3)
2. Copy backup files to the target environment's object storage bucket
3. Point `backup.yaml` at the target Milvus instance
4. Run restore in the target environment (Step 4–5)

This is the standard path for separate test/staging/prod Milvus instances regardless of
deployment type.

---

## Step 7 — Backup discipline checklist

1. **Quarterly restore test**: run a full tested restore to a throwaway collection; verify
   `row_count` and search quality match the source
2. **Before major schema changes**: backup first — schema is immutable and a bad change
   requires a restore
3. **Before Milvus version upgrades**: trigger a manual backup
4. **Retention**: keep at least **3 backup generations**
5. **`--restore_index` flag**: include to rebuild indexes after restore; slower but
   immediately searchable without a separate `create_index` call

---

## Reference Files

- `references/backup-yaml-template.md` — Fully-commented `backup.yaml` template covering
  Milvus gRPC endpoint, MinIO/S3/GCS/Azure Blob storage configuration, and credential
  injection patterns for each backend
