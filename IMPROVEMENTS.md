# SSHFT GitHub Action Improvements

## Overview
The SSHFT (SSH File Transfer) GitHub composite action has been enhanced with improved temporary directory management, smart compression logic, and robust cleanup mechanisms.

## Key Improvements

### 1. Random Temporary Directory Management
- **Unique Random IDs**: Uses OpenSSL to generate cryptographically secure 8-byte hex strings combined with timestamps
- **Safe Remote Paths**: Creates temporary directories in `/tmp/sshft_temp_${UNIQUE_SUFFIX}` format to avoid conflicts
- **Collision Avoidance**: The combination of timestamp and random hex ensures virtually no chance of directory name collisions

#### Before:
```bash
TAR_FILE="temp_sshft_${BASE_NAME}.tar.gz"
# Used ~/temp_extract directory (potential conflicts)
```

#### After:
```bash
RANDOM_ID=$(openssl rand -hex 8)
TIMESTAMP=$(date +%s)
UNIQUE_SUFFIX="${TIMESTAMP}_${RANDOM_ID}"
REMOTE_TEMP_DIR="/tmp/sshft_temp_${UNIQUE_SUFFIX}"
TAR_FILE="sshft_${BASE_NAME}_${UNIQUE_SUFFIX}.tar.gz"
```

### 2. Smart Compression Logic
The action now intelligently handles already compressed files to avoid double compression:

#### Compression Detection Function
```bash
is_compressed() {
  local file="$1"
  local extension="${file##*.}"
  
  # Check common compressed extensions
  case "$extension" in
    gz|tgz|tar.gz|bz2|tbz2|tar.bz2|xz|txz|tar.xz|zip|rar|7z|tar|lz4|zst)
      return 0 ;;
  esac
  
  # Also checks MIME type if available
  # Returns 0 if compressed, 1 if not
}
```

#### Smart Upload Logic
- **Single Compressed Files**: Transfers directly without re-compression
- **Directories**: Always compresses for consistency and bandwidth efficiency
- **Regular Files**: Compresses only if not already compressed

#### Smart Download Logic
- **Compressed Remote Files**: Downloads directly without re-compression
- **Regular Remote Files**: Compresses before transfer
- **Directories**: Always compresses for efficient transfer

### 3. Enhanced Cleanup Mechanisms

#### Comprehensive Cleanup Function
```bash
cleanup() {
  echo "Cleaning up temporary files..."
  # Clean up local temp directory
  rm -rf "$TEMP_DIR" 2>/dev/null || true
  # Clean up remote temp directory and any temp files
  ssh -F "$SSH_CONFIG_FILE" ${{ inputs.host }} "rm -rf \"$REMOTE_TEMP_DIR\" /tmp/sshft_*_${UNIQUE_SUFFIX}.tar.gz" 2>/dev/null || true
}
trap cleanup EXIT
```

#### Cleanup Features
- **Trap Handler**: Automatically cleans up on script exit (success or failure)
- **Manual Cleanup**: Additional cleanup after successful operations
- **Pattern Matching**: Removes any stray temporary files with the unique suffix
- **Error Suppression**: Uses `2>/dev/null || true` to prevent cleanup errors from affecting the main operation

### 4. Improved Error Handling

#### Better Error Messages
- More specific error messages for different failure scenarios
- Separate error codes for remote vs local operations
- Clear indication of which step failed

#### Robust Operations
- Creates remote temporary directories before operations
- Verifies directory creation success
- Handles edge cases like empty directories

### 5. Bandwidth Optimization

#### Transfer Efficiency
- **Avoids Double Compression**: Prevents unnecessary bandwidth usage
- **Smart File Handling**: Transfers compressed files directly when possible
- **Consistent Directory Handling**: Always compresses directories for predictable behavior

#### File Size Benefits
- Compressed files transferred directly save CPU cycles
- Reduced network usage for already optimized files
- Maintains compression ratios for pre-compressed content

## Usage Examples

### Transferring Mixed Content
```yaml
- name: Upload mixed files
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "dist/"  # Contains both .js files and .gz assets
    destination: "/var/www/html/"
    # The action will compress .js files but transfer .gz files directly
```

### Downloading Compressed Logs
```yaml
- name: Download compressed logs
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "/var/log/app.log.gz"
    destination: "./logs/"
    direction: "download"
    # The .gz file will be downloaded directly without re-compression
```

## Security Enhancements

### File Transfer Security
- **Size Limits**: 2GB maximum to prevent resource exhaustion
- **Disk Space Validation**: Checks available space with 20% buffer
- **Path Normalization**: Removes trailing slashes, validates paths
- **Symlink Safety**: Detects and validates symlinks
- **Destination Validation**: Checks existence, type, and permissions

### Script Execution Security
- **Dangerous Command Detection**: Blocks destructive commands (rm -rf /, dd, shutdown, mkfs, etc.)
- **Privilege Escalation Prevention**: Blocks sudo/su
- **Remote Code Execution Prevention**: Blocks curl/wget piped to bash
- **Resource Limits**: ulimit restrictions (100 processes, 1GB files, 5min CPU, 2GB memory)
- **Execution Timeout**: 10-minute maximum per script
- **Command Injection Prevention**: Detects nested substitutions

### Temporary Directory Security
- Uses `/tmp` with unique identifiers to prevent directory traversal
- Proper cleanup prevents information leakage
- Random naming prevents predictable attacks

### File Handling Security
- All temporary files use unique suffixes
- Comprehensive cleanup removes all traces
- Error paths also trigger cleanup

## Backward Compatibility

All existing functionality remains unchanged:
- Same input parameters
- Same output format
- Same error handling behavior
- Existing workflows will continue to work without modification

## Performance Benefits

1. **Reduced Transfer Time**: Direct transfer of compressed files
2. **Lower CPU Usage**: No unnecessary compression/decompression cycles
3. **Better Bandwidth Utilization**: Optimized transfer decisions
4. **Improved Reliability**: Better cleanup and error handling

## Testing Recommendations

Test the improved action with:
1. Single compressed files (should transfer directly)
2. Directories with mixed file types
3. Large compressed archives
4. Error scenarios to verify cleanup
5. Concurrent runs to verify unique directory handling