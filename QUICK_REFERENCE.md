# Quick Reference Cheat Sheet

> 📖 **For full documentation, see [README.md](README.md)**

## 🎯 Basic Usage

```yaml
- uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: "dist/"
    destination: "/var/www/html/"
```

## 📊 Inputs/Outputs

### Required Inputs
- `host` - SSH server address
- `username` - SSH username  
- `key` - SSH private key (use GitHub Secrets)
- `source` - Path to upload/download
- `destination` - Target path

### Optional Inputs
- `direction` - `upload` (default) or `download`
- `backup_before_transfer` - `true` (default) or `false`
- `post_script` / `post_script_path` - Script to run after transfer
- `port` - SSH port (default: `22`)
- `destination_host` - For remote-to-remote downloads
- `destination_username` / `destination_key` - Destination server credentials

### Key Outputs
- `backup_created` - Whether backup was made
- `backup_path` - Path to backup file
- `error` - Error message if failed
- `script_executed` - Script ran successfully

## 🔄 Common Patterns

### Upload with Backup
```yaml
- uses: kellydc/sshft@v1
  id: deploy
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: "dist/"
    destination: "/var/www/html/"
```

### Download (Remote → Runner)
```yaml
- uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}  # Source server
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: "/var/log/app.log"  # Remote path
    destination: "./logs/"  # Runner path
    direction: "download"

# ⚠️ Runner storage is ephemeral. Save as artifacts to persist!
- uses: actions/upload-artifact@v4
  with:
    name: downloaded-files
    path: ./logs/
```

### Download (Remote → Remote)
```yaml
- uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SOURCE_HOST }}  # Source server
    username: ${{ secrets.SOURCE_USER }}
    key: ${{ secrets.SOURCE_KEY }}
    source: "/data/backup.tar.gz"
    destination: "/backups/"  # Destination server path
    direction: "download"
    destination_host: ${{ secrets.DEST_HOST }}
    destination_username: ${{ secrets.DEST_USER }}
    destination_key: ${{ secrets.DEST_KEY }}
```

### Disable Backup
```yaml
- uses: kellydc/sshft@v1
  with:
    # ... same as above
    backup_before_transfer: false
```

### With Rollback
```yaml
- uses: kellydc/sshft@v1
  id: deploy
  with:
    # ... upload config

- if: failure() && steps.deploy.outputs.backup_created == 'true'
  run: |
    ssh user@host "tar -xzf ${{ steps.deploy.outputs.backup_path }} -C /var/www/"
```

### With Post-Script
```yaml
- uses: kellydc/sshft@v1
  with:
    # ... upload config
    post_script: |
      systemctl restart nginx
      curl -f http://localhost/health
```

## ⚠️ Common Errors

| Error | Meaning | Fix |
|-------|---------|-----|
| `Source size exceeds limit` | Upload > 2GB or Download > 10GB | Split into smaller transfers |
| `Insufficient disk space` | Not enough space | Free up space on destination |
| `Script blocked: dangerous commands` | Contains rm -rf /, sudo, etc. | Remove dangerous commands |
| `Destination is not a directory` | Path is a file | Use a directory path |

## 🔒 Security Limits

- **Max file size**: 2GB for uploads, 10GB for downloads
- **Blocked commands**: rm -rf /, dd, sudo, shutdown, curl\|bash
- **Resource limits**: 100 processes, 2GB memory, 10min timeout
- **Backups**: Last 10 kept (auto-cleanup)

## 🛠️ Troubleshooting

**Q: Backup not created?**  
A: Check if destination exists. First deploy skips backup.

**Q: Script failed?**  
A: Test locally: `bash -n script.sh`. Check for blocked commands.

**Q: Out of space?**  
A: Check `~/backups` directory. Old backups auto-deleted after 10.

## 📚 More Help

- [README.md](README.md) - Full documentation
- [EXAMPLES.md](EXAMPLES.md) - More examples
- [SECURITY.md](SECURITY.md) - Security details
