# milvus-backup — backup.yaml Template

Fully-commented configuration file for the `milvus-backup` CLI.
Replace all `<placeholder>` values before use. Credentials should be injected from
environment variables or a secrets manager — never commit secrets to source control.

```yaml
# ─── Milvus connection ──────────────────────────────────────────────────────
milvus:
  address: <host>           # e.g. "localhost" or "milvus.internal"
  port: 19530               # gRPC port (default 19530; IBM watsonx.data: read from Infrastructure Manager)
  authorizationEnabled: true
  # Choose ONE auth method:
  # username: <user>
  # password: <password>
  # — or —
  token: <token>            # For managed cloud; format: see milvuscraft-context Deployment overrides
  tlsMode: 0                # 0=disabled, 1=server-side TLS, 2=mutual TLS
  # serverName: ""          # SNI override; required when tlsMode=1 or 2

# ─── Backup storage ─────────────────────────────────────────────────────────
# Choose ONE storage backend section below and remove the others.

# MinIO (self-hosted)
minio:
  address: <minio-host>
  port: 9000
  accessKeyID: <access-key>
  secretAccessKey: <secret-key>
  useSSL: false
  bucketName: milvus-backup
  rootPath: backup/

# AWS S3
# minio:
#   address: s3.amazonaws.com
#   port: 443
#   accessKeyID: <AWS_ACCESS_KEY_ID>
#   secretAccessKey: <AWS_SECRET_ACCESS_KEY>
#   useSSL: true
#   bucketName: my-milvus-backup-bucket
#   rootPath: backup/
#   cloudProvider: aws
#   region: us-east-1

# Google Cloud Storage
# minio:
#   address: storage.googleapis.com
#   port: 443
#   accessKeyID: ""           # Leave empty; use GOOGLE_APPLICATION_CREDENTIALS env var
#   secretAccessKey: ""
#   useSSL: true
#   bucketName: my-milvus-backup-bucket
#   rootPath: backup/
#   cloudProvider: gcp

# Azure Blob Storage
# minio:
#   address: <account>.blob.core.windows.net
#   port: 443
#   accessKeyID: <account-name>
#   secretAccessKey: <account-key>
#   useSSL: true
#   bucketName: milvus-backup-container
#   rootPath: backup/
#   cloudProvider: azure

# IBM Cloud Object Storage (S3-compatible)
# minio:
#   address: s3.<region>.cloud-object-storage.appdomain.cloud
#   port: 443
#   accessKeyID: <cos-hmac-access-key>
#   secretAccessKey: <cos-hmac-secret-key>
#   useSSL: true
#   bucketName: my-milvus-backups
#   rootPath: backups/
#   cloudProvider: aws     # COS uses S3-compatible API
#   crossStorage: true     # Required when source store differs from backup target

# ─── Backup behaviour ────────────────────────────────────────────────────────
backup:
  maxSegmentGroupSize: 2G   # Max segment group size per backup shard
  parallelism:
    backupCollection: 4     # Concurrent collections backed up simultaneously
    copydata: 128           # Concurrent segment files copied
    restoreCollection: 2    # Concurrent collections restored simultaneously

# ─── HTTP API (optional) ─────────────────────────────────────────────────────
http:
  enabled: false            # Set true to expose REST API on port 8080
  simpleResponse: false
```

---

## Credential injection patterns

### Environment variables (recommended)

```bash
export MILVUS_TOKEN="<token>"
export MINIO_ACCESS_KEY="<key>"
export MINIO_SECRET_KEY="<secret>"

# milvus-backup does not natively expand env vars in YAML — use envsubst:
envsubst < backup.yaml.template > backup.yaml
./milvus-backup create -c my_collection -n my_backup
```

### AWS IAM role (EC2 / EKS)

If running on AWS EC2 or EKS with an IAM role attached, leave `accessKeyID` and
`secretAccessKey` blank — the AWS SDK picks up credentials from the instance metadata
service automatically.

### GCP Workload Identity / Service Account

Set `GOOGLE_APPLICATION_CREDENTIALS` to the path of your service account JSON key file,
or use Workload Identity on GKE. Leave `accessKeyID` and `secretAccessKey` blank.

---

## crossStorage flag

Set `crossStorage: true` when the Milvus source storage and the backup target are different
object storage services (e.g., MinIO source → IBM COS target). With `crossStorage: true`,
data streams through the milvus-backup process rather than using the storage system's COPY
API. This is slower but works across incompatible storage backends.

`crossStorage` is auto-set when `storageType ≠ backupStorageType`. Set it manually when
both are the same type (e.g., both S3) but different service accounts or buckets.
