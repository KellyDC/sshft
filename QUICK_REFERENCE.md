# Quick Reference: New Backup & Validation Features

## 📋 Table of Contents
1. [Quick Start](#-quick-start)
2. [Workflow Overview](#-workflow-overview)
3. [New Features](#-new-input)
4. [Backup Features](#-backup-features)
5. [Script Validation](#-script-validation)
6. [Common Use Cases](#-common-use-cases)
7. [Error Messages](#️-error-messages-you-might-see)
8. [Troubleshooting](#-troubleshooting)

---

## 🎯 Quick Start

### Default Behavior (Backup Enabled)
```yaml
- uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.HOST }}
    username: ${{ secrets.USER }}
    key: ${{ secrets.KEY }}
    source: "dist/"
    destination: "/var/www/html/"
  # ✓ Backup automatically created before transfer
  # ✓ Scripts validated before execution
```

---

## 🔄 Workflow Overview

The action follows 6 modular phases:

```
1. SSH Setup          → Validate and configure SSH
2. Connection Test    → Verify connectivity
3. Backup (Upload)    → Create destination backup
4. File Transfer      → Upload or download files
5. Post-Script        → Execute optional scripts
6. Cleanup           → Secure removal of temp files
```

### Visual Flow

```
[SSH Setup] ──✓──> [Test Connection] ──✓──> [Backup] ──✓──> [Transfer] ──✓──> [Script] ──✓──> [Cleanup]
     │                    │                     │               │              │              │
     ✗                    ✗                     ✗               ✗              ⚠️             ✓
     │                    │                     │               │              │              │
     └────────────────────┴─────────────────────┴───────────────┴──────────────┴──────> [End]
```

- **✓** = Success, continue
- **✗** = Critical failure, stop
- **⚠️** = Non-critical error, log and continue

### Phase Details

| Phase | What It Does | Can Fail Action? |
|-------|-------------|------------------|
| **1. SSH Setup** | Validates SSH key, creates config | ✗ Yes |
| **2. Connection** | Tests SSH connection | ✗ Yes |
| **3. Backup** | Backs up destination (upload only) | ✗ Yes (if enabled) |
| **4. Transfer** | Compresses, transfers, extracts files | ✗ Yes |
| **5. Post-Script** | Validates and runs scripts | ⚠️ No (logs only) |
| **6. Cleanup** | Removes SSH keys and temp files | ✓ Always runs |

---

## 🆕 New Input

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `backup_before_transfer` | boolean | `true` | Create backup before uploading |

---

## 🆕 New Outputs

| Output | Type | Description |
|--------|------|-------------|
| `backup_created` | boolean | Whether backup was created |
| `backup_path` | string | Full path to backup file |
| `backup_size` | string | Human-readable backup size |

---

## 📦 Backup Features

### What Happens Automatically:
1. ✅ Checks if destination exists
2. ✅ Creates `~/backups` directory if needed
3. ✅ Compresses destination to tar.gz
4. ✅ Stores with timestamp: `backup_dest_YYYYMMDD_HHMMSS_randid.tar.gz`
5. ✅ Reports location and size
6. ✅ Keeps only last 10 backups (auto-cleanup)
7. ✅ Skips gracefully if destination doesn't exist

### Backup Storage Location:
```
~/backups/
├── backup_html_20231110_143022_abc123.tar.gz
├── backup_html_20231110_153045_def456.tar.gz
└── backup_html_20231110_163108_ghi789.tar.gz
```

---

## 🔍 Script Validation

### What Gets Validated:
- ✅ Empty scripts
- ✅ Syntax errors (`bash -n`)
- ✅ Unmatched braces `{}`
- ✅ Unmatched parentheses `()`
- ✅ Unmatched brackets `[]`
- ✅ Incomplete pipelines
- ✅ File existence (for `post_script_path`)

### Validation Process:
```
Inline Script:
1. Local validation
2. Upload to remote
3. Remote validation
4. Execute

Remote Script:
1. Check existence
2. Check readability
3. Validate syntax
4. Check structure
5. Execute
```

---

## 💡 Common Use Cases

### 1. Check if Backup Was Created
```yaml
- uses: kellydc/sshft@v1
  id: deploy
  with:
    # ... your config

- run: |
    if [ "${{ steps.deploy.outputs.backup_created }}" == "true" ]; then
      echo "Backup: ${{ steps.deploy.outputs.backup_path }}"
      echo "Size: ${{ steps.deploy.outputs.backup_size }}"
    fi
```

### 2. Disable Backup for Temp Files
```yaml
- uses: kellydc/sshft@v1
  with:
    source: "temp/"
    destination: "/tmp/data/"
    backup_before_transfer: false
    # ... other config
```

### 3. Rollback on Failure
```yaml
- uses: kellydc/sshft@v1
  id: deploy
  with:
    # ... config with post_script that might fail

- name: Rollback
  if: failure() && steps.deploy.outputs.backup_created == 'true'
  run: |
    ssh user@host "
      tar -xzf ${{ steps.deploy.outputs.backup_path }} -C /path/to/parent/
    "
```

### 4. Save Backup Info
```yaml
- uses: kellydc/sshft@v1
  id: deploy
  with:
    # ... your config

- run: |
    echo "BACKUP_PATH=${{ steps.deploy.outputs.backup_path }}" >> $GITHUB_ENV
    echo "BACKUP_SIZE=${{ steps.deploy.outputs.backup_size }}" >> $GITHUB_ENV
```

---

## ⚠️ Error Messages You Might See

### Backup Errors
| Error | Meaning |
|-------|---------|
| `Remote temp dir creation failed` | Cannot create temp directory |
| `Destination directory issue` | Destination not writable |
| `Remote extraction failed` | Cannot extract backup |

### Script Validation Errors
| Error | Meaning |
|-------|---------|
| `Script is empty or malformed` | Empty script provided |
| `Script syntax error: unexpected token` | Bash syntax error |
| `Unmatched braces (open: 3, close: 2)` | Missing `}` |
| `Unmatched parentheses (open: 2, close: 1)` | Missing `)` |
| `Unmatched brackets (open: 1, close: 2)` | Extra `]` |
| `Pipeline without command` | `ls |` or `| grep` |
| `Script file does not exist` | Wrong path |
| `Script execution failed with exit code X` | Script ran but failed |

---

## 🔧 Troubleshooting

### Q: Backup not created?
**A**: Check if destination exists. First-time deployments skip backup.

### Q: Script validation failed?
**A**: Test script locally: `bash -n yourscript.sh`

### Q: Backup storage full?
**A**: Retention policy keeps only 10 backups. Check `~/backups` size.

### Q: Need older backup?
**A**: Backups are in `~/backups` on remote server, sorted by date.

---

## 📚 More Information

- **Full Documentation**: See [README.md](README.md)
- **Examples**: See [EXAMPLES.md](EXAMPLES.md)
- **Detailed Changelog**: See [CHANGELOG_BACKUP_FEATURE.md](CHANGELOG_BACKUP_FEATURE.md)
- **Implementation Details**: See [IMPLEMENTATION_SUMMARY.md](IMPLEMENTATION_SUMMARY.md)

---

## 🎨 Quick Examples

### Production Deployment
```yaml
- uses: kellydc/sshft@v1
  id: deploy
  with:
    host: ${{ secrets.HOST }}
    username: ${{ secrets.USER }}
    key: ${{ secrets.KEY }}
    source: "dist/"
    destination: "/var/www/html/"
    post_script: |
      sudo systemctl restart nginx
      if ! curl -f http://localhost/health; then
        echo "Health check failed!"
        exit 1
      fi

- name: Show deployment info
  run: |
    echo "✓ Deployed successfully"
    echo "Backup: ${{ steps.deploy.outputs.backup_path }}"
```

### Development Deployment (No Backup)
```yaml
- uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.DEV_HOST }}
    username: ${{ secrets.DEV_USER }}
    key: ${{ secrets.DEV_KEY }}
    source: "build/"
    destination: "/var/www/dev/"
    backup_before_transfer: false
```

### Critical Deployment with Rollback
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy
        id: deploy
        uses: kellydc/sshft@v1
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USER }}
          key: ${{ secrets.KEY }}
          source: "dist/"
          destination: "/var/www/html/"
          post_script: |
            sudo systemctl restart nginx
            sleep 5
            curl -f http://localhost/health
      
      - name: Rollback on failure
        if: failure() && steps.deploy.outputs.backup_created == 'true'
        uses: kellydc/sshft@v1
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USER }}
          key: ${{ secrets.KEY }}
          source: "."  # dummy
          destination: "/var/www/html/"
          backup_before_transfer: false
          post_script: |
            BACKUP="${{ steps.deploy.outputs.backup_path }}"
            rm -rf /var/www/html/*
            tar -xzf "$BACKUP" -C /var/www/
            sudo systemctl restart nginx
            echo "Rolled back to: $BACKUP"
```

---

## ✨ Benefits Summary

| Feature | Before | After |
|---------|--------|-------|
| Data Safety | Manual backups | Automatic backups |
| Script Errors | Runtime failures | Pre-execution validation |
| Rollback | Manual process | Simple with backup outputs |
| Storage | Unmanaged | Auto retention (10 backups) |
| Error Messages | Generic | Specific & actionable |

---

**Version**: 1.0.0 (November 2024)
**Backward Compatible**: ✅ Yes
**Breaking Changes**: ❌ None
