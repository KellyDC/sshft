# Changelog - Backup Feature & Script Validation

## Summary
Added automatic backup functionality and enhanced script validation to prevent errors from malformed post-transfer scripts.

## Changes Made

### 1. New Input Parameter
- **`backup_before_transfer`**: Boolean (default: `true`)
  - Enables/disables automatic backup before file transfer
  - Only applies to upload operations (remote destination backup)

### 2. New Outputs
- **`backup_created`**: Whether a backup was successfully created
- **`backup_path`**: Full path to the backup file on the remote server
- **`backup_size`**: Human-readable size of the backup file

### 3. Backup Step (New)
**Location**: Between "Verify SSH connection" and "Transfer files" steps

**Features**:
- ✅ Creates `~/backups` directory on remote server if it doesn't exist
- ✅ Checks if destination exists before attempting backup
- ✅ Generates unique backup filename with timestamp and random ID
  - Format: `backup_{destination_name}_{YYYYMMDD_HHMMSS}_{random_id}.tar.gz`
- ✅ Compresses destination using `tar.gz` for efficient storage
- ✅ Verifies backup was created successfully
- ✅ Reports backup size and location to user
- ✅ Implements backup retention policy (keeps last 10 backups)
- ✅ Comprehensive error handling with detailed error messages
- ✅ Only runs for upload operations (not downloads)
- ✅ Skips gracefully if destination doesn't exist yet

**Security**:
- Backup directory has restricted permissions (700)
- All paths are properly quoted to handle special characters
- Verification checks ensure backup integrity

### 4. Enhanced Script Validation

#### For Inline Scripts (`post_script`)
**New Validations**:
- ✅ Empty script detection (after trimming whitespace)
- ✅ Local syntax validation using `bash -n` before upload
- ✅ Unmatched quotes detection
- ✅ Pipeline without command detection
- ✅ Unmatched braces `{}` detection with count
- ✅ Unmatched parentheses `()` detection with count
- ✅ Unmatched brackets `[]` detection with count
- ✅ Remote syntax validation (double-check after upload)
- ✅ Detailed error messages with exit codes
- ✅ Validation passes before execution proceeds

#### For Script Path (`post_script_path`)
**New Validations**:
- ✅ File existence check
- ✅ File readability check
- ✅ Empty file detection (size check)
- ✅ Syntax validation using `bash -n`
- ✅ Unmatched braces detection
- ✅ Unmatched parentheses detection
- ✅ Unmatched brackets detection
- ✅ Pipeline validation
- ✅ Warning messages for suspicious patterns
- ✅ Detailed error messages with specific issues

**Error Handling Improvements**:
- All validation errors now include specific details
- Non-fatal warnings are displayed but don't stop execution
- Exit codes are captured and reported
- Script output is sanitized before being set as output
- Better error categorization (syntax vs execution errors)

### 5. Transfer Step Update
**Change**: Added condition to ensure transfer only proceeds after successful backup
```yaml
if: success() || (failure() && steps.backup.conclusion == 'skipped')
```
This ensures:
- Transfer runs if backup succeeded
- Transfer runs if backup was skipped (e.g., downloads or backup disabled)
- Transfer does NOT run if backup failed

## Example Usage

### Basic Usage (with backup)
```yaml
- uses: KellyDC/sshft@main
  with:
    host: ${{ secrets.SERVER_HOST }}
    username: ${{ secrets.SERVER_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: ./dist
    destination: /var/www/html
    # backup_before_transfer defaults to true
```

### Disable Backup
```yaml
- uses: KellyDC/sshft@main
  with:
    host: ${{ secrets.SERVER_HOST }}
    username: ${{ secrets.SERVER_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: ./dist
    destination: /var/www/html
    backup_before_transfer: false
```

### Check Backup Details
```yaml
- uses: KellyDC/sshft@main
  id: deploy
  with:
    host: ${{ secrets.SERVER_HOST }}
    username: ${{ secrets.SERVER_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: ./dist
    destination: /var/www/html

- name: Display backup info
  if: steps.deploy.outputs.backup_created == 'true'
  run: |
    echo "Backup created: ${{ steps.deploy.outputs.backup_path }}"
    echo "Backup size: ${{ steps.deploy.outputs.backup_size }}"
```

### Safe Post-Script with Validation
```yaml
- uses: KellyDC/sshft@main
  with:
    host: ${{ secrets.SERVER_HOST }}
    username: ${{ secrets.SERVER_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: ./dist
    destination: /var/www/html
    post_script: |
      echo "Restarting web server..."
      sudo systemctl restart nginx
      echo "Deployment complete!"
```

## Error Messages

### Backup Errors
- `Remote temp dir creation failed`: Cannot create temporary directory
- `Destination directory issue`: Destination doesn't exist or not writable
- `Remote extraction failed`: Failed to extract backup archive
- `Remote file move failed`: Failed to move backup file

### Script Validation Errors
- `Script is empty or malformed`: Empty script provided
- `Script syntax error: unexpected token or malformed command`: Bash syntax error
- `Script validation failed: malformed syntax`: Structural issues (unmatched braces, etc.)
- `Script execution failed with exit code X`: Script ran but failed
- `Script file does not exist`: Path doesn't exist on remote
- `Script file is not readable`: Permission issue
- `Script file is empty`: Zero-byte file

## Testing Recommendations

1. **Test backup creation**: 
   - Upload to existing destination
   - Verify backup appears in `~/backups`
   - Check backup can be extracted

2. **Test backup skip**: 
   - Upload to non-existent destination
   - Verify backup is skipped gracefully

3. **Test backup disable**: 
   - Set `backup_before_transfer: false`
   - Verify no backup is created

4. **Test script validation**:
   - Try script with unmatched braces
   - Try script with syntax errors
   - Try script with unexpected tokens
   - Verify errors are caught before execution

5. **Test retention policy**:
   - Create 11+ backups
   - Verify only last 10 are kept

## Security Considerations

- Backup directory has restrictive permissions (700)
- All file paths are properly quoted
- Script validation prevents code injection
- Syntax checking on both local and remote
- Temporary files are cleaned up securely
- SSH keys remain secure throughout process

## Backward Compatibility

✅ **Fully backward compatible**
- Existing workflows continue to work without changes
- Backup feature is opt-out (enabled by default)
- No breaking changes to existing inputs/outputs
- New outputs are additive only
