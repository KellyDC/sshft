# Quick Start Guide

> 📖 **For complete documentation, see [README.md](README.md)**

## Minimal Example

```yaml
- uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: "dist/"
    destination: "/var/www/html/"
```

## Common Patterns

### Download Files
```yaml
- uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: "/var/log/app/"
    destination: "./logs/"
    direction: "download"
```

### With Rollback
```yaml
- uses: kellydc/sshft@v1
  id: deploy
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: "dist/"
    destination: "/var/www/html/"

- if: failure() && steps.deploy.outputs.backup_created == 'true'
  run: ssh user@host "tar -xzf ${{ steps.deploy.outputs.backup_path }} -C /var/www/"
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
      curl -f http://localhost/health
```

## What You Need to Know

- ✅ **Auto-backup**: Creates backup before upload (disable with `backup_before_transfer: false`)
- ✅ **10GB limit**: Files/directories over 10GB are rejected
- ✅ **Scripts**: Dangerous commands (rm -rf /, sudo, shutdown, etc.) are blocked
- ⚠️ **Backups**: Only kept for last 10 deployments (auto-cleanup)

## Key Inputs

| Input | Required | Default |
|-------|----------|---------|
| `host` | Yes | - |
| `username` | Yes | - |
| `key` | Yes | - |
| `source` | Yes | - |
| `destination` | Yes | - |
| `direction` | No | `upload` |
| `backup_before_transfer` | No | `true` |
| `post_script` | No | - |

## Key Outputs

| Output | Description |
|--------|-------------|
| `backup_created` | Backup was created (boolean) |
| `backup_path` | Path to backup file |
| `backup_size` | Backup file size |
| `script_executed` | Script ran successfully |
| `error` | Error message if failed |

## Documentation

- **[README.md](README.md)** - Complete documentation
- **[QUICK_REFERENCE.md](QUICK_REFERENCE.md)** - Cheat sheet
- **[EXAMPLES.md](EXAMPLES.md)** - More examples
- **[SECURITY.md](SECURITY.md)** - Security details

