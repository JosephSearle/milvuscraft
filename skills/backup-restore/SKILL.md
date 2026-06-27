---
name: backup-restore
description: >
  Back up and restore Milvus collections. Load this skill whenever the user mentions: backup
  Milvus, restore Milvus, disaster recovery, milvus-backup CLI, data migration, cross-environment
  copy, promote from test to staging, promote to production, copy a collection between
  environments, RPO, RTO, etcd snapshot, or what happens if Milvus goes down. Always load
  milvuscraft-context first.
---

# Milvuscraft Backup & Restore Skill

Collection backup and restore using the `milvus-backup` open-source CLI, cross-environment
promotion patterns, and disaster-recovery discipline. No MCP tools exist for backup — all
operations use CLI or PyMilvus SDK.

---

## Core Philosophy

Untested backups are not backups. Every backup procedure ends with a test restore to a
throwaway collection and a sample search to confirm data integrity. Schedule quarterly
restore tests even when nothing has changed.

`milvus-backup` is **full-only** — there is no incremental option. For near-real-time RPO,
pair it with **Milvus-CDC** for incremental data synchronisation.

---

## Step 1 — Check for managed-service backup

Check milvuscraft-context → Deployment overrides for any native backup tooling provided by
your cloud provider. If a managed backup API is available, prefer it over the open-source
CLI. If no override exists, use the `milvus-backup` CLI as documented below.

---

## Step 2 — Understand what milvus-backup covers (and what it does not)

`milvus-backup` copies collection metadata and segment data to object storage. It does
**not** snapshot **etcd**, which holds cluster runtime metadata (collection load state,
index state, topic offsets).

A complete DR runbook requires **both**:

1. **etcd snapshot** — run separately before or after each milvus-backup:
   ```bash
   etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M).db \
     --endpoints=<etcd-endpoint>
   ```
   Restoring via milvus-backup alone cannot rebuild etcd-dependent runtime states.

2. **milvus-backup** — covers collection data (Step 3 below)

Store both artefacts together, versioned, off-site.

---

## Step 3 — Install and configure milvus-backup CLI

Pin to **≥v0.5.11** (fixes a FlushAll metadata corruption bug affecting Milvus ≥2.6.9;
safe to pin even on earlier Milvus versions).

```bash
go install github.com/zilliztech/milvus-backup@v0.5.11
```

Configure `backup.yaml` — see `references/backup-yaml-template.md` for a fully-commented
template covering Milvus gRPC endpoint and all object storage backends (MinIO, S3, GCS,
Azure Blob).

---

## Step 4 — Create a backup

```bash
# Backup a single collection
./milvus-backup create -c my_collection -n my_backup_$(date +%Y%m%d)

# Backup all collections in a database
./milvus-backup create -n full_backup_$(date +%Y%m%d)

# Metadata-only backup (fast; schema recovery only)
./milvus-backup create -c my_collection -n meta_backup --meta_only
```

Verify the backup was created successfully:

```bash
./milvus-backup list
```

Confirm the backup name appears with `status: success`.

---

## Step 5 — Test restore (mandatory after every backup)

```bash
# Restore with a suffix to avoid name collision with the live collection
./milvus-backup restore -n my_backup_20260520 -s _test
```

Load and search the restored collection to confirm data integrity:

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

## Step 6 — Restore in production

**Same-instance restore:**

```bash
./milvus-backup restore -n my_backup_20260520 -s _restored
```

Appends suffix to avoid collision with the live collection. Load and alias-swap
(see milvuscraft-collection-lifecycle) when ready.

**Cross-collection rename:**

```bash
./milvus-backup restore -n my_backup_20260520 \
  -r source_db.source_coll:target_db.target_coll
```

**Restore with index** (avoids a separate create_index call after restore):

```bash
./milvus-backup restore -n my_backup_20260520 -s _restored --restore_index
```

---

## Step 7 — Cross-environment promotion (test → staging → production)

1. Back up from the source environment (Step 4)
2. Copy backup files to the target environment's object storage bucket
3. Point `backup.yaml` at the target Milvus instance
4. Run restore in the target environment (Steps 5–6)

This is the standard path for separate test/staging/prod Milvus instances regardless of
deployment type.

---

## Step 8 — Near-real-time RPO with Milvus-CDC

`milvus-backup` RPO is bounded by backup frequency (full-only). For lower RPO, run
**Milvus-CDC** alongside scheduled backups:

- Milvus-CDC captures and synchronises **incremental** data between Milvus instances in
  near-real-time
- Use it to maintain a warm standby for DR; fall back to `milvus-backup` for point-in-time
  recovery or cross-environment promotion
- milvus-backup + Milvus-CDC + etcd snapshots together form a complete DR strategy

---

## Step 9 — RTO guidance

| Action | Typical time | Notes |
|--------|-------------|-------|
| Restore metadata only (`--meta_only`) | Seconds | Schema only; no data |
| Restore collection data | Minutes–hours | Scales with collection size |
| Build index after restore | Minutes–hours | Avoid with `--restore_index` flag |
| Load collection after restore | Seconds–minutes | Scales with segment count |

Reduce RTO by: using `--restore_index`, pre-loading restored collection, and keeping
regular smaller backups alongside periodic full backups.

---

## Step 10 — Backup discipline checklist

1. **Quarterly restore test**: run a full tested restore to a throwaway collection; verify
   `row_count` and search quality match the source
2. **etcd snapshot**: always pair with milvus-backup — run `etcdctl snapshot save` before
   or immediately after each backup run
3. **Before major schema changes**: backup first — schema is largely immutable
4. **Before Milvus version upgrades**: trigger a manual backup
5. **Retention**: keep at least **3 backup generations**; backups restore only to the same
   or newer Milvus version (restore supported into Milvus 2.4+)
6. **`--restore_index`**: include to rebuild indexes after restore; slower but immediately
   searchable without a separate `create_index` call

---

## Reference Files

- `references/backup-yaml-template.md` — Fully-commented `backup.yaml` template covering
  Milvus gRPC endpoint, MinIO/S3/GCS/Azure Blob storage configuration, and credential
  injection patterns for each backend
