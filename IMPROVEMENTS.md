# SSHFT GitHub Action Improvements

## Overview
The SSHFT (SSH File Transfer) action provides secure bidirectional file transfer with support for runner-to-remote uploads, remote-to-runner downloads, and remote-to-remote transfers.

## Key Features

### 1. Bidirectional Transfer Modes
- **Upload**: Runner → Remote server (tar.gz + scp, with automatic backup)
- **Download**: Remote → Runner (rsync for efficient sync)
- **Download**: Remote → Remote (rsync direct transfer via runner)

### 2. Smart Transfer Methods
- **Upload**: tar.gz compression for atomic deployments
- **Download**: rsync for efficient sync with built-in compression
- Auto-detects compressed files (gz, tar.gz, bz2, xz, zip, etc.) on upload
- Skips re-compression to save bandwidth and CPU
- Preserves file permissions, timestamps, and attributes on download

### 3. Unique Temporary Management
- Cryptographically secure random IDs (`openssl rand -hex 8`)
- Timestamp-based unique suffixes
- Safe `/tmp/sshft_temp_${UNIQUE_SUFFIX}` paths
- Zero collision risk

### 4. Comprehensive Cleanup
- Trap handlers for automatic cleanup on exit
- Pattern matching removes all temporary files
- Cleans both local and remote resources
- Error-suppressed for reliability

### 5. Security Features
- **File limits**: 2GB upload, 10GB download
- **Disk validation**: 20% buffer check
- **Script security**: Blocks dangerous commands, resource limits
- **Secure cleanup**: Keys overwritten before deletion

### 6. Recent Improvements (January 2026)

#### Backup Filename Optimization
- Removed redundant `backup_` prefix from backup filenames
- Descriptive names using full destination path: `var_www_mysite_20260107_123456_abc123.tar.gz`
- Cleaner organization since files are already in `~/backups/` directory

#### Download Optimization with rsync
- Replaced tar+scp with rsync for all download operations
- **Benefits**:
  - Simpler workflow (no compress/extract cycle)
  - Preserves file permissions, timestamps, and ownership
  - Built-in compression during transfer (`-z` flag)
  - Handles hidden files (dotfiles) automatically
  - Better for incremental syncs and backups
  - Direct server-to-server transfers more efficient

#### Hidden Files & Empty Directory Handling
- Upload now includes dotfiles (`.env`, `.htaccess`, etc.)
- Graceful handling of empty directories
- `shopt -s dotglob` ensures all files are copied

## Usage Examples

### Remote-to-Remote Transfer
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

### Download with Persistence
```yaml
- uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: "/var/log/app.log"
    destination: "./logs/"
    direction: "download"

# Persist runner downloads
- uses: actions/upload-artifact@v4
  with:
    name: logs
    path: ./logs/
```

## Backward Compatibility

All existing functionality preserved - existing workflows work without modification.