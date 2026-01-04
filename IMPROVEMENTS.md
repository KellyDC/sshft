# SSHFT GitHub Action Improvements

## Overview
The SSHFT (SSH File Transfer) action provides secure bidirectional file transfer with support for runner-to-remote uploads, remote-to-runner downloads, and remote-to-remote transfers.

## Key Features

### 1. Bidirectional Transfer Modes
- **Upload**: Runner → Remote server (with automatic backup)
- **Download**: Remote → Runner (ephemeral storage)
- **Download**: Remote → Remote (server-to-server via runner)

### 2. Smart Compression
- Auto-detects compressed files (gz, tar.gz, bz2, xz, zip, etc.)
- Skips re-compression to save bandwidth and CPU
- Consistent directory handling with tar.gz

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