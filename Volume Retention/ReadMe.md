# Databricks Unity Catalog Volume Backup & Point-in-Time Recovery

A policy-driven backup and recovery framework for files stored in Databricks Unity Catalog Volumes.

This project demonstrates how business-critical files stored inside Unity Catalog Volumes can be protected using:

- Policy-driven backup selection
- Incremental file backup
- Logical recovery snapshots
- Configurable retention
- Delta-based backup manifests
- Backup run auditing
- Dry-run recovery
- Staging-based restore validation
- Full-snapshot and single-file recovery

---

## Project Motivation

Delta tables provide strong historical data-management capabilities through transaction history and time travel.

Files stored inside Unity Catalog Volumes have different recovery requirements.

Volumes are commonly used to store artifacts such as:

- Application JAR files
- Configuration files
- Deployment packages
- Reference datasets
- Documents
- Machine learning artifacts
- Supporting binaries
- Application resources

For business-critical volume content, I wanted to build a framework that could provide a controlled file-level recovery capability.

The goal was to answer a simple question:

> Can a Unity Catalog Volume be protected with a lightweight, policy-driven backup framework that supports incremental backups and point-in-time recovery?

This project is my implementation of that idea.

---

# High-Level Architecture

```text
                 Unity Catalog Volumes
                          |
                          v
                volume_backup_policy
                          |
                          v
                02_volume_backup_job
                          |
              +-----------+-----------+
              |           |           |
              v           v           v
             NEW       MODIFIED    UNCHANGED
              |           |           |
              +------> Backup <--------+
                          |
                          v
                 UC Backup Volume
                          |
                          v
               volume_backup_manifest
                          |
                          v
                  Restore Engine
                          |
                 03_volume_restore
                          |
               +----------+----------+
               |                     |
               v                     v
            DRY RUN               RESTORE
                                     |
                           +---------+---------+
                           |                   |
                           v                   v
                        STAGING              SOURCE
```

---

# Repository Structure

```text
Volume Retention/
│
├── 01_volume_backup_setup.ipynb
│
├── 02_volume_backup_job.ipynb
│
├── 03_volume_restore.ipynb
│
└── README.md
```

Each notebook has a specific responsibility.

---

# 01 - Volume Backup Setup

`01_volume_backup_setup.ipynb` initializes the complete framework.

The notebook is designed to be rerunnable and creates the required Unity Catalog objects only when they do not already exist.

## Objects Created

### Catalog

```text
backup_admin
```

### Schema

```text
backup_admin.volume_protection
```

### Backup Volume

```text
/Volumes/backup_admin/volume_protection/backups
```

### Delta Control Tables

```text
backup_admin.volume_protection.volume_backup_policy

backup_admin.volume_protection.volume_backup_manifest

backup_admin.volume_protection.volume_backup_run
```

---

# Backup Policy

The `volume_backup_policy` Delta table determines which volumes should be protected.

Example policy used in the project:

```text
Policy ID        : POLICY_APP_ARTIFACTS

Source Catalog   : royalinsurance

Source Schema    : edw

Source Volume    : app_artifacts

Backup Enabled   : true

Priority         : CRITICAL

Retention        : 7 days

Frequency        : DAILY
```

This makes the solution configuration-driven.

Adding another protected volume does not require rewriting the backup engine. A new policy can be added instead.

---

# Backup Manifest

The `volume_backup_manifest` Delta table acts as the recovery catalog.

Each snapshot contains file-level metadata such as:

```text
snapshot_id
policy_id
snapshot_timestamp
source_catalog
source_schema
source_volume
source_relative_path
source_full_path
backup_full_path
file_name
file_size_bytes
source_modification_time
file_checksum
change_type
backup_status
error_message
created_at
```

The manifest connects a logical recovery point with the physical backup objects required to reconstruct it.

---

# Backup Run Audit

Every execution is recorded in:

```text
backup_admin.volume_protection.volume_backup_run
```

Metrics include:

```text
run_id
start_timestamp
end_timestamp
run_status

protected_volume_count

scanned_file_count
new_file_count
modified_file_count
unchanged_file_count
deleted_file_count

backed_up_file_count
failed_file_count

bytes_backed_up

error_message
```

A successful execution provides an operational summary similar to:

```text
Status             : SUCCESS
Protected volumes  : 1
Scanned files      : 2
Backed-up files    : 2
Failed files       : 0
```

---

# 02 - Incremental Volume Backup

`02_volume_backup_job.ipynb` performs the actual protection process.

The notebook reads enabled policies and recursively scans each configured source volume.

---

## File Classification

Every source file is classified into one of four states:

```text
NEW
MODIFIED
UNCHANGED
DELETED
```

### NEW

The file does not exist in the previous logical snapshot.

Action:

```text
Physical backup required
```

---

### MODIFIED

The file existed previously but the file metadata indicates a newer version.

Action:

```text
New physical backup version required
```

---

### UNCHANGED

The file is unchanged compared with the previous snapshot.

Action:

```text
Reuse existing backup object
```

No unnecessary physical copy is required.

---

### DELETED

The file existed previously but no longer exists in the source volume.

Action:

```text
Record deletion in snapshot manifest
```

This enables the framework to understand the logical state of the volume for each recovery point.

---

# Incremental Backup Design

The framework intentionally avoids copying the entire volume during every daily execution.

For example:

```text
Previous Snapshot
      |
      v
Scan Current Volume
      |
      +---- NEW ----------> Backup
      |
      +---- MODIFIED -----> Backup
      |
      +---- UNCHANGED ----> Reuse existing object
      |
      +---- DELETED ------> Manifest record
```

This makes the framework more storage-efficient for volumes where only a small percentage of files change between executions.

---

# Physical Backup Storage

Physical backup versions are stored in:

```text
/Volumes/
    backup_admin/
        volume_protection/
            backups/
                objects/
```

Example structure:

```text
objects/
└── royalinsurance/
    └── edw/
        └── app_artifacts/
            │
            ├── <version-hash-1>/
            │   └── insurance-dashboard.jar
            │
            └── <version-hash-2>/
                └── zulu21.54.107-ca-jdk21.0.13-linux_x64.tar.gz
```

The version-specific directory allows historical file versions to coexist.

---

# File Version Fingerprint

The framework generates a deterministic version identifier using file metadata.

Conceptually:

```text
relative file path
        +
file size
        +
modification timestamp
        |
        v
SHA-256 fingerprint
```

Example:

```text
9707c2cfb2ef10a076312de3a87ff78df9c5c2b3df7768dec13e21ac5c0d1512
```

This identifier becomes part of the immutable physical backup path.

---

# Backup Validation

A file is not considered safely backed up merely because the copy command returned successfully.

The framework validates the physical result.

```text
Source File
    |
    v
dbutils.fs.cp()
    |
    v
Destination exists?
    |
    v
Compare file size
    |
    +---- mismatch ----> FAILED
    |
    v
SUCCESS
```

The manifest receives:

```text
backup_status = SUCCESS
```

only after the physical backup object is confirmed.

Failures are recorded as:

```text
backup_status = FAILED
```

with an associated error message.

This prevents invalid recovery points from silently appearing healthy.

---

# Logical Snapshots

Each backup execution generates a unique logical snapshot.

Example:

```text
POLICY_APP_ARTIFACTS_20261120T082641626377Z
```

A snapshot represents the recoverable state of a protected volume at a specific point in time.

Multiple physical file versions can therefore support multiple logical snapshots.

---

# Retention Strategy

Retention is configurable through the policy table.

Example:

```text
retention_days = 7
```

The project currently demonstrates a seven-day recovery window.

Retention processing removes expired manifest information based on policy configuration.

Physical backup objects should only be removed when they are no longer referenced by any retained recovery point.

---

# Safe Orphan Cleanup

The framework includes logic for identifying physical backup objects that are no longer referenced by retained snapshots.

Before path comparison, volume paths are canonicalized.

For example:

```text
/Volumes/a/b//c/file.jar
```

is normalized to:

```text
/Volumes/a/b/c/file.jar
```

This avoids incorrectly classifying legitimate backup files as orphan objects because of path-format differences.

Physical orphan deletion is intentionally controlled separately from detection so cleanup can be validated safely before destructive operations are enabled.

---

# 03 - Point-in-Time Restore

`03_volume_restore.ipynb` provides the recovery workflow.

The restore notebook uses the backup manifest to identify physical objects belonging to a selected recovery point.

---

# Restore Modes

The framework supports two restore modes.

## SNAPSHOT

```text
restore_mode = SNAPSHOT
```

Restores the complete recoverable state represented by the selected snapshot.

---

## SINGLE_FILE

```text
restore_mode = SINGLE_FILE
```

Restores one selected file.

Example:

```text
relative_file_path = insurance-dashboard.jar
```

This is useful when only one deployment artifact or configuration file needs to be recovered.

---

# Restore Targets

The framework supports two target modes.

## STAGING

```text
target_mode = STAGING
```

This is the recommended default.

Files are restored below:

```text
/Volumes/
    backup_admin/
        volume_protection/
            backups/
                restore_staging/
```

The structure includes:

```text
Policy
    |
Snapshot
    |
Restore ID
    |
Restored files
```

Example:

```text
restore_staging/
└── POLICY_APP_ARTIFACTS/
    └── POLICY_APP_ARTIFACTS_20261120T082641626377Z/
        └── <restore-id>/
            ├── insurance-dashboard.jar
            └── zulu21.54.107-ca-jdk21.0.13-linux_x64.tar.gz
```

This allows restored files to be validated before the source volume is modified.

---

## SOURCE

```text
target_mode = SOURCE
```

This mode restores files directly into the original protected Unity Catalog Volume.

For safety, this mode should only be used after staging validation.

---

# Dry-Run Recovery

The default recovery mode is:

```text
dry_run = true
```

A dry run:

1. Validates the selected policy.
2. Validates the snapshot.
3. Locates physical backup objects.
4. Builds the restore plan.
5. Calculates the number and size of recoverable files.
6. Determines the destination paths.
7. Records the restore attempt.
8. Does not physically copy files.

Example restore plan:

```text
Files in plan    : 2
Files restorable : 2
Existing targets : 0
Missing backups  : 0
Bytes to restore : 277,633,698
```

Only after this validation succeeds should the physical restore be executed.

---

# Physical Restore Validation

During an actual restore:

```text
Backup Object
      |
      v
Create Target Directory
      |
      v
Copy File
      |
      v
Target Exists?
      |
      v
Compare Restored Size
      |
      +---- mismatch ----> FAILED
      |
      v
SUCCESS
```

The framework validates the restored file size against the expected file size from the manifest.

---

# Restore Audit

Restore operations are recorded in:

```text
backup_admin.volume_protection.volume_restore_audit
```

Audit information includes:

```text
restore_id
restore_timestamp

policy_id
snapshot_id

source_catalog
source_schema
source_volume

source_relative_path
backup_full_path
restore_target_path

restore_mode
target_mode

overwrite_enabled
dry_run

file_size_bytes

restore_status
error_message

restored_by
created_at
```

Example dry-run status:

```text
DRY_RUN_RESTORE
```

Example successful physical restore:

```text
SUCCESS
```

---

# Recommended Recovery Workflow

The framework intentionally encourages a controlled recovery process.

```text
1. Select Policy
        |
        v
2. Select Snapshot
        |
        v
3. Validate Snapshot
        |
        v
4. Verify Physical Backup Objects
        |
        v
5. Build Restore Plan
        |
        v
6. Dry Run
        |
        v
7. Restore to Staging
        |
        v
8. Validate File Count and Size
        |
        v
9. Optional Restore to Source
```

---

# Example Test Case

The initial use case protects:

```text
/Volumes/royalinsurance/edw/app_artifacts
```

Example protected files:

```text
insurance-dashboard.jar

zulu21.54.107-ca-jdk21.0.13-linux_x64.tar.gz
```

A validated restore plan recovered:

```text
Files in plan    : 2

Files restorable : 2

Missing backups  : 0

Bytes to restore : 277,633,698
```

This demonstrated end-to-end recovery from the manifest-backed snapshot.

---

# Operational Workflow

The intended production-style workflow is:

```text
01_volume_backup_setup
        |
        | Run once / when configuration changes
        v
02_volume_backup_job
        |
        | Scheduled daily
        v
Backup Snapshot
        |
        | Retained according to policy
        v
03_volume_restore
        |
        | Run when recovery is required
        v
Dry Run
        |
        v
Staging Restore
        |
        v
Validation
        |
        v
Optional Source Recovery
```

---

# Scheduling

`02_volume_backup_job` is designed to run as a Databricks Workflow.

Example schedule:

```text
Frequency : Daily
Time      : 01:00
Time Zone : Asia/Kolkata
Mode      : INCREMENTAL
```

Typical parameters:

```text
run_mode = INCREMENTAL

policy_id =

perform_retention_cleanup = true
```

Leaving `policy_id` blank allows all enabled policies to be processed.

---

# Security and Permissions

A production deployment should use a Databricks workflow identity or service principal with only the permissions required by the framework.

Typical requirements include:

```text
USE CATALOG
USE SCHEMA
READ VOLUME
WRITE VOLUME
SELECT
MODIFY
```

Source volumes normally require read access for the backup workflow.

The backup volume requires read and write access.

Source-volume write access should only be granted to the recovery identity if direct SOURCE restoration is required.

---

# Failure Handling

The framework is designed to fail safely.

Examples:

### Physical backup missing

```text
BACKUP_OBJECT_MISSING
```

Restore is blocked.

### Copy failure

```text
backup_status = FAILED
```

The snapshot does not silently appear healthy.

### Restore size mismatch

```text
restore_status = FAILED
```

The file is not considered successfully recovered.

### Existing restore target

If:

```text
overwrite_existing = false
```

the result is:

```text
SKIP_TARGET_EXISTS
```

instead of silently replacing a file.

---

# Design Principles

The implementation follows several principles.

## Configuration over hardcoding

Backup targets are driven by:

```text
volume_backup_policy
```

---

## Incremental over full duplication

Only changed file versions require new physical copies.

---

## Metadata and physical data are separated

Delta tables maintain policy, manifest and operational metadata.

Unity Catalog Volumes contain physical backup objects.

---

## Validate before trusting

A successful API call does not automatically mean a valid backup or restore exists.

Physical objects and file sizes are verified.

---

## Restore safely first

The preferred sequence is:

```text
Dry Run
    ↓
Staging Restore
    ↓
Validation
    ↓
Source Restore
```

---

## Audit everything

Backup executions and restore activity are recorded using Delta tables.

---

# Technology Stack

- Databricks
- Unity Catalog
- Unity Catalog Volumes
- Delta Lake
- PySpark
- Python
- Databricks Notebooks
- Databricks Workflows
- `dbutils.fs`
- Git / GitHub

---

# Future Enhancements

Potential extensions include:

### Backup Health Dashboard

Visualize:

```text
Protected volumes
Successful backups
Failed backups
Storage consumption
Files protected
Recovery points available
Last successful backup
```

### Recovery Timeline

Provide a visual timeline of available restore points.

### Automatic Workflow Alerts

Send notifications for:

```text
Backup failure
Partial success
Missing backup objects
Retention-cleanup failure
Restore failure
```

### Checksum Validation

The current framework uses file metadata fingerprints for identifying file versions.

A future implementation could optionally generate content-level checksums for high-value files where stronger integrity verification is required.

### Multi-Volume Recovery

Restore several related volumes to a coordinated recovery point.

### Recovery Approval Workflow

Require approval before:

```text
target_mode = SOURCE
```

is permitted.

### Recovery Dashboard

Build an application above the Delta control tables to allow users to:

```text
Select volume
Select recovery date
Preview snapshot
Select file
Dry run
Restore
Review audit history
```

---

# What This Project Demonstrates

This project goes beyond copying files between directories.

It demonstrates the design of a small data-platform protection system incorporating:

- Unity Catalog governance
- Policy-driven configuration
- Incremental processing
- File-version tracking
- Delta-based metadata management
- Backup validation
- Recovery-point management
- Retention
- Auditability
- Safe restore patterns
- Operational failure handling

---

# Key Takeaway

The central idea is simple:

> Treat critical Unity Catalog Volume files as governed data assets that require explicit protection, recovery points, validation and auditability.

The resulting framework provides a practical example of how Databricks can be used not only for data processing and analytics, but also for building operational data-platform capabilities.

---

## Author

**Rohan Bhanu**

Built as part of a hands-on Databricks engineering and architecture portfolio exploring practical lakehouse, governance, resiliency, application and recovery patterns.