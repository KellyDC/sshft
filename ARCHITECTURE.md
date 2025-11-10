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
│                     GitHub Action Runtime                    │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Phase 1   │→ │   Phase 2   │→ │   Phase 3   │         │
│  │  SSH Setup  │  │ Connection  │  │   Backup    │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│         ↓                ↓                 ↓                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Phase 4   │→ │   Phase 5   │→ │   Phase 6   │         │
│  │  Transfer   │  │ Post-Script │  │   Cleanup   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                               │
└─────────────────────────────────────────────────────────────┘
                           ↓
                  ┌──────────────────┐
                  │  Action Outputs  │
                  └──────────────────┘
```

---

## Phase Breakdown

### Phase 1: SSH Setup

**Purpose**: Establish secure SSH configuration

**Components**:
```
Input: SSH key, host, username, port, passphrase
  ↓
[Validate Key Format] → [Create SSH Directory]
  ↓
[Generate Unique Filenames] → [Save SSH Key]
  ↓
[Set Permissions (600)] → [Create SSH Config]
  ↓
[Setup Host Key Checking] → [Handle Passphrase]
  ↓
Output: SSH_KEY_FILE, SSH_CONFIG_FILE paths
```

**Key Features**:
- ✅ SSH key format validation
- ✅ Unique temporary file names (no conflicts)
- ✅ Strict permission control (600/700)
- ✅ Optional passphrase support
- ✅ Configurable host key checking

**Error Handling**:
- Invalid key format → Stop
- Directory creation failure → Stop
- Permission setting failure → Stop

---

### Phase 2: Connection Test

**Purpose**: Verify SSH connectivity before file operations

**Components**:
```
Input: SSH config from Phase 1
  ↓
[Execute Test Command] → "echo SSH connection successful"
  ↓
[Evaluate Response]
  ↓
Output: Connection status (pass/fail)
```

**Key Features**:
- ✅ Pre-flight connectivity check
- ✅ Early failure detection
- ✅ Uses configured SSH settings

**Error Handling**:
- Connection failure → Stop (prevents wasted operations)

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
Input: Remote source path, local destination
  ↓
[Verify Remote Source Exists]
  ↓
[Compress on Remote] → tar -czf
  ↓
[Download to Local] → scp
  ↓
[Create Local Destination]
  ↓
[Extract Locally] → tar -xzf
  ↓
[Cleanup Temp Files] → Local & Remote
  ↓
Output: success=true
```

**Key Features**:
- ✅ Bidirectional transfer (upload/download)
- ✅ Automatic compression
- ✅ Integrity checks at each step
- ✅ Atomic cleanup (trap handlers)

**Error Handling**:
- Source doesn't exist → Stop
- Transfer failure → Stop
- Extraction failure → Stop
- Cleanup failure → Log, continue

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
                    Destination Files
```

### Download Operation

```
Remote Files
    ↓
[Validate] → [Compress]
    ↓
Remote tar.gz ──[SCP]──> Local temp dir
    ↓
[Extract] → [Move to Destination]
    ↓
Local Files
```

### Backup Operation

```
Destination Files (Remote)
    ↓
[Compress] → tar.gz
    ↓
~/backups/backup_name_timestamp_id.tar.gz
    ↓
[Retention Policy] → Keep last 10
```

---

## State Management

### Environment Variables

The action uses GitHub environment variables to pass state between steps:

```bash
# Set in Phase 1 (SSH Setup)
SSH_KEY_FILE=/path/to/temp_key_timestamp_random
SSH_CONFIG_FILE=/path/to/temp_config_timestamp_random
SSH_KNOWN_HOSTS=/path/to/temp_known_hosts_timestamp_random

# Available in all subsequent phases
```

### Step Outputs

Each phase produces outputs via `$GITHUB_OUTPUT`:

```yaml
Phase 1: None (internal state only)
Phase 2: None (success/failure via exit code)
Phase 3: backup_created, backup_path, backup_size
Phase 4: success, error
Phase 5: script_executed, script_output, script_error
Phase 6: None (cleanup always succeeds)
```

---

## Error Propagation

### Critical Errors (Stop Execution)

These errors terminate the action immediately:

1. **Phase 1**: Invalid SSH key, permission errors
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
