# Architecture Documentation

This document explains the modular architecture of the sshft GitHub Action.

## Design Philosophy

The action is designed with:
- **Modularity**: Each phase is independent and self-contained
- **Fail-fast**: Critical errors stop execution early
- **Graceful degradation**: Non-critical features (scripts) log errors but don't fail
- **Security-first**: All credentials are validated and securely cleaned up
- **Transparency**: Each phase reports its status through outputs

---

## System Architecture

### High-Level Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     GitHub Action Runtime                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Phase 1   │→ │   Phase 2   │→ │   Phase 3   │          │
│  │  SSH Setup  │  │ Connection  │  │   Backup    │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│         ↓                ↓                 ↓                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Phase 4   │→ │   Phase 5   │→ │   Phase 6   │          │
│  │  Transfer   │  │ Post-Script │  │   Cleanup   │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           ↓
                  ┌──────────────────┐
                  │  Action Outputs  │
                  └──────────────────┘
```

---

## Phase Breakdown

### Phase 1: SSH Setup

**Purpose**: Establish secure SSH configuration for source and optionally destination

**Components**:
```
Input: SSH credentials, host, username, port
  ↓
[Validate Key Format] → [Create SSH Directory]
  ↓
[Generate Unique Filenames] → [Save SSH Keys]
  ↓
[Set Permissions (600)] → [Create SSH Configs]
  ↓
[Setup Host Key Checking] → [Handle Passphrases]
  ↓
[For Downloads: Setup Destination SSH] (if destination_host provided)
  ↓
Output: SSH config files for source (and destination)
```

**Key Features**:
- ✅ Dual SSH setup for remote-to-remote transfers
- ✅ SSH key format validation
- ✅ Unique temporary file names
- ✅ Strict permissions (600/700)
- ✅ Optional passphrase support

**Error Handling**:
- Invalid key → Stop
- Directory/permission failure → Stop

---

### Phase 2: Connection Test

**Purpose**: Verify SSH connectivity before operations

**Connections Tested**:
- Source server (always)
- Destination server (for remote-to-remote downloads)

**Error Handling**:
- Connection failure → Stop

---

### Phase 3: Backup (Conditional)

**Purpose**: Create recoverable backup of destination

**Conditions**:
- Only runs for **upload** operations
- Only if `backup_before_transfer` is `true` (default)
- Skips if destination doesn't exist (first deploy)

**Components**:
```
Input: Destination path, direction
  ↓
[Check Direction] → Upload? → Continue / Download? → Skip
  ↓
[Check Destination Exists]
  ↓
  Exists? → [Create ~/backups Directory]
  No? → Skip (first deploy)
  ↓
[Generate Unique Filename] → backup_name_YYYYMMDD_HHMMSS_randomid.tar.gz
  ↓
[Compress Destination] → tar -czf
  ↓
[Verify Backup Created] → [Get Size]
  ↓
[Retention Policy] → Keep last 10 backups
  ↓
Output: backup_created, backup_path, backup_size
```

**Key Features**:
- ✅ Automatic compression (tar.gz)
- ✅ Timestamped filenames
- ✅ Automatic retention (10 backups)
- ✅ Graceful skip for first deploy
- ✅ Size reporting

**Error Handling**:
- Backup enabled + backup fails → Stop transfer
- Destination doesn't exist → Skip gracefully
- Retention cleanup fails → Log warning, continue

---

### Phase 4: File Transfer

**Purpose**: Transfer files between local and remote systems

#### Upload Subflow:
```
Input: Source path, destination path
  ↓
[Validate Source Exists]
  ↓
[Create Temp Directory]
  ↓
[Compress Source] → tar -czf local_file.tar.gz
  ↓
[Transfer to Remote] → scp
  ↓
[Verify Destination Writable]
  ↓
[Extract on Remote] → tar -xzf
  ↓
[Cleanup Temp Files] → Local & Remote
  ↓
Output: success=true
```

#### Download Subflow:
```
Input: Remote source, destination
  ↓
[Verify Source Exists] → [Compress on Remote]
  ↓
[Determine Transfer Mode]
  ├─ Remote → Runner (no destination_host)
  │   ↓
  │   [Download via SCP] → [Extract on Runner] → [Validate Paths]
  │
  └─ Remote → Remote (with destination_host)
      ↓
      [Download to Runner] → [Upload to Destination] → [Extract on Destination]
  ↓
[Cleanup Temp Files]
  ↓
Output: success=true
```

**Transfer Modes**:
- **Remote → Runner**: Files land on runner (ephemeral - use `actions/upload-artifact` to persist)
- **Remote → Remote**: Server-to-server via runner as intermediary

**Key Features**:
- ✅ Dual transfer modes
- ✅ Auto-compression
- ✅ Integrity checks
- ✅ Atomic cleanup

**Error Handling**:
- Source doesn't exist → Stop
- Transfer failure → Stop
- Extraction failure → Stop

---

### Phase 5: Post-Script (Optional)

**Purpose**: Execute optional scripts after successful transfer

**Modes**:
1. **Inline Script**: Provided in workflow YAML
2. **Remote Script**: Path to existing script on remote server

#### Inline Script Flow:
```
Input: post_script content
  ↓
[Validate Non-Empty]
  ↓
[Create Temp Script File]
  ↓
[Syntax Validation] → bash -n (local)
  ↓
[Structural Validation]
  ├─ Check unmatched braces
  ├─ Check unmatched parentheses
  ├─ Check unmatched brackets
  └─ Check incomplete pipelines
  ↓
[Upload to Remote] → scp
  ↓
[Syntax Validation] → bash -n (remote, double-check)
  ↓
[Execute Script] → bash script.sh
  ↓
[Capture Output] → stdout/stderr
  ↓
[Cleanup Temp Script]
  ↓
Output: script_executed, script_output, script_error
```

#### Remote Script Flow:
```
Input: post_script_path
  ↓
[Check File Exists]
  ↓
[Check Readable]
  ↓
[Check Non-Empty]
  ↓
[Syntax Validation] → bash -n
  ↓
[Structural Validation]
  ├─ Check unmatched braces
  ├─ Check unmatched parentheses
  ├─ Check unmatched brackets
  └─ Check incomplete pipelines
  ↓
[Execute Script]
  ↓
[Capture Output]
  ↓
Output: script_executed, script_output, script_error
```

**Validation Checks**:
- ✅ Empty script detection
- ✅ Bash syntax errors (`bash -n`)
- ✅ Unmatched braces: `{` vs `}`
- ✅ Unmatched parentheses: `(` vs `)`
- ✅ Unmatched brackets: `[` vs `]`
- ✅ Incomplete pipelines: `|` without commands

**Key Features**:
- ✅ Pre-execution validation
- ✅ Detailed error messages
- ✅ Non-blocking (errors don't fail action)
- ✅ Output sanitization (10KB limit)

**Error Handling**:
- Validation fails → Log error, skip execution
- Execution fails → Log error with exit code
- Output too large → Truncate to 10KB
- All errors → Continue to cleanup

---

### Phase 6: Cleanup

**Purpose**: Securely remove all temporary files and credentials

**Components**:
```
[Always Executes] → Even on failure
  ↓
[Stop ssh-agent] → If passphrase was used
  ↓
[Securely Delete SSH Key]
  ├─ Overwrite with zeros (dd if=/dev/zero)
  └─ Remove file
  ↓
[Remove SSH Config]
  ↓
[Remove Known Hosts]
  ↓
[Remove Local Temp Files]
  ↓
[Remove Remote Temp Files] → Best effort
  ↓
Output: Clean environment
```

**Key Features**:
- ✅ Always executes (trap on EXIT)
- ✅ Secure key deletion (overwrite before remove)
- ✅ Cleanup both local and remote
- ✅ Best-effort approach (never fails)

**Error Handling**:
- All cleanup errors are suppressed (|| true)
- Logs warnings but never fails
- Ensures action completes

---

## Data Flow

### Upload Operation

```
Local Files
    ↓
[Validate] → [Compress]
    ↓
Local tar.gz ──[SCP]──> Remote temp dir
                            ↓
                    [Extract] → [Move to Destination]
                            ↓
                ### Download Operations

#### Remote → Runner
```
Remote Files → [Compress] → SCP → Runner → [Extract] → Local Files
```
⚠️ Runner storage is ephemeral - use `actions/upload-artifact` to persist

#### Remote → Remote
```
Source Server → [Compress] → SCP → Runner → SCP → Dest Server → [Extract] → Files
```
✅ Files persist on destination server

### Backup Operation
```
Destination Files → [Compress] → ~/backups/backup_YYYYMMDD_HHMMSS_id.tar.gz
                                              ↓
                                    [Retention: Keep last 10]
```

---

## State Management

### Environment Variables

State shared between steps:

```bash
# Source SSH (always set)
SSH_KEY_FILE, SSH_CONFIG_FILE, SSH_KNOWN_HOSTS

# Destination SSH (for remote-to-remote downloads)
DEST_SSH_KEY_FILE, DEST_SSH_CONFIG_FILE, DEST_SSH_KNOWN_HOSTS
```

### Step Outputs

```yaml
backup: backup_created, backup_path, backup_size
transfer: success, error
post_script: script_executed, script_output, script_error
```

---

## Error Propagation

### Critical Errors (Stop Execution)

1. **Phase 1**: Invalid SSH key, permission errors
2. **Phase 2**: Connection failure
3. **Phase 3**: Backup creation failure (if enabled and destination exists)
4. **Phase 4**: Source missing, transfer failure, extraction failure, disk space
5. **Phase 5**: Non-blocking (errors logged only)
6. **Phase 6**: Never fails

---

## Security Features

- **File size limits**: 2GB upload, 10GB download
- **Disk space validation**: 20% buffer required
- **Script validation**: Syntax + structure + dangerous command detection
- **Resource limits**: 100 processes, 2GB memory, 5min CPU, 10min timeout
- **Secure cleanup**: SSH keys overwritten before deletion
- **Path validation**: Prevents traversal attacks, validates symlinks
2. **Phase 2**: Connection failure
3. **Phase 3**: Backup failure (if enabled and required)
4. **Phase 4**: Source missing, transfer failure, extraction failure

### Non-Critical Errors (Log and Continue)

These errors are logged but don't stop execution:

1. **Phase 3**: Retention cleanup warnings
2. **Phase 5**: Script validation errors, script execution errors
3. **Phase 6**: Cleanup errors (best-effort)

### Error Flow Diagram

```
┌──────────────┐
│  Error Type  │
└──────┬───────┘
       │
   ┌───▼────────────────┐
   │ Critical Error?    │
   └───┬────────┬───────┘
       │        │
      Yes       No
       │        │
       ▼        ▼
   [Stop]  [Log & Continue]
       │        │
       ▼        ▼
   [Cleanup] [Continue]
```

---

## Security Model

### Credential Handling

```
SSH Key (Input Secret)
    ↓
[Validate Format]
    ↓
[Save to Temp File] → Permissions: 600
    ↓
[Use for Operations]
    ↓
[Overwrite with Zeros]
    ↓
[Delete File]
    ↓
No Traces Left
```

### Permission Model

```
SSH Directory: 700 (rwx------)
SSH Key File:  600 (rw-------)
SSH Config:    600 (rw-------)
Backup Dir:    700 (rwx------)
Backup Files:  644 (rw-r--r--)
```

### Secure Operations

- ✅ Keys never logged or exposed
- ✅ Unique temporary filenames (no collisions)
- ✅ Overwrite keys before deletion
- ✅ Restricted permissions on all sensitive files
- ✅ No hardcoded credentials
- ✅ Proper path quoting (prevent injection)

---

## Performance Optimization

### Compression Strategy

- Uses `tar.gz` (gzip) for good compression/speed balance
- Compression happens at source (reduces transfer time)
- Single archive (fewer round-trips than multiple files)

### Parallel Operations

- Phase execution is sequential (for safety)
- Within phases, operations are optimized:
  - Backup creation while source is being prepared
  - Validation checks run before expensive operations

### Resource Usage

- Temporary files cleaned up immediately after use
- Trap handlers ensure cleanup on unexpected exits
- No long-running background processes

---

## Extensibility

### Adding New Features

The modular design makes it easy to add features:

1. **New validation**: Add to Phase 5 validation checks
2. **New backup options**: Extend Phase 3 logic
3. **New transfer modes**: Add to Phase 4 subflows
4. **New outputs**: Add to appropriate phase

### Backward Compatibility

All new features maintain backward compatibility:

- New inputs have sensible defaults
- New outputs are additive (don't break existing workflows)
- Existing behavior unchanged unless explicitly configured

---

## Testing Strategy

### Unit-Level Testing

Each phase can be tested independently:

```bash
# Test SSH Setup
./test_phase1.sh

# Test Connection
./test_phase2.sh

# Test Backup
./test_phase3.sh
```

### Integration Testing

Test complete workflows:

```bash
# Test full upload flow
./test_upload_flow.sh

# Test full download flow
./test_download_flow.sh

# Test with post-script
./test_with_script.sh
```

### Error Testing

Test error conditions:

```bash
# Invalid SSH key
# Connection failure
# Missing source
# Script validation errors
```

---

## Monitoring and Observability

### Logging Levels

The action provides detailed logging:

```
INFO:  Standard operation messages
WARN:  Non-critical issues (script errors, retention cleanup)
ERROR: Critical failures (connection, transfer)
DEBUG: Detailed operation info (for troubleshooting)
```

### Output Visibility

Users can monitor action status through:

- Step logs (visible in GitHub Actions UI)
- Step outputs (accessible in workflow)
- Exit codes (for conditional workflow logic)

### Example Monitoring

```yaml
- uses: kellydc/sshft@v1
  id: deploy
  with:
    # ... config

- name: Monitor deployment
  run: |
    echo "Transfer: ${{ steps.deploy.outputs.success }}"
    echo "Backup: ${{ steps.deploy.outputs.backup_created }}"
    echo "Script: ${{ steps.deploy.outputs.script_executed }}"
```

---

## Comparison with Traditional Approaches

### vs. Manual SCP

| Feature | Manual SCP | sshft Action |
|---------|-----------|--------------|
| Setup | Manual every time | Automated |
| Compression | Manual | Automatic |
| Backup | Manual | Automatic |
| Validation | None | Comprehensive |
| Error Handling | Manual | Built-in |
| Cleanup | Manual | Automatic |

### vs. rsync

| Feature | rsync | sshft Action |
|---------|-------|--------------|
| Incremental | ✓ Yes | ✗ Full transfer |
| Backup | Manual | Automatic |
| Compression | Optional | Always |
| GitHub Integration | Manual setup | Native |
| Post-scripts | Manual | Validated & automated |

### vs. Git Deploy

| Feature | Git Deploy | sshft Action |
|---------|-----------|--------------|
| Built Artifacts | Need Git | Any files |
| Backup | None | Automatic |
| Bidirectional | ✗ Push only | ✓ Upload/Download |
| Post-scripts | Hooks | Validated scripts |

---

## Future Enhancements

Potential improvements while maintaining architecture:

1. **Selective Backup**: Backup only specific subdirectories
2. **Backup Compression Levels**: Configurable compression
3. **Parallel Transfers**: Multiple files simultaneously
4. **Resume Support**: Resume interrupted transfers
5. **Bandwidth Throttling**: Rate limiting for large transfers
6. **Pre-transfer Hooks**: Scripts before transfer
7. **Notification Integration**: Slack/email notifications
8. **Metrics Collection**: Transfer speed, sizes, times

---

## Conclusion

The sshft action's modular architecture provides:

- **Reliability**: Each phase is independent and well-tested
- **Security**: Comprehensive credential and permission handling
- **Flexibility**: Easy to extend without breaking existing functionality
- **Transparency**: Clear logging and output at every stage
- **Usability**: Smart defaults with extensive configuration options

This design ensures the action is production-ready while remaining maintainable and extensible.
