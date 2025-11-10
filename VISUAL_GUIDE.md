# Visual Workflow Guide

This guide provides easy-to-understand visual representations of how the sshft action works.

## 📊 Complete Workflow

### The Six Phases

```
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃                         sshft Action                            ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛

┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Phase 1   │───▶│   Phase 2   │───▶│   Phase 3   │
│  SSH Setup  │    │ Connection  │    │   Backup    │
│             │    │    Test     │    │  (Upload)   │
└─────────────┘    └─────────────┘    └─────────────┘
      │                   │                   │
      ▼                   ▼                   ▼
   CRITICAL           CRITICAL            CRITICAL
   (stops on          (stops on           (stops on
    error)             error)              error)

┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Phase 4   │───▶│   Phase 5   │───▶│   Phase 6   │
│    File     │    │Post-Script  │    │   Cleanup   │
│  Transfer   │    │ (Optional)  │    │             │
└─────────────┘    └─────────────┘    └─────────────┘
      │                   │                   │
      ▼                   ▼                   ▼
   CRITICAL          NON-CRITICAL         ALWAYS RUNS
   (stops on          (logs errors,        (never fails)
    error)            continues)
```

---

## 🔄 Upload Workflow (Simplified)

```
Your Computer                         Remote Server
═════════════                         ═════════════

    [Files]
       │
       ▼
   Compress────────────────────┐
    (tar.gz)                   │
       │                       │
       ▼                       │
   Transfer ───────────────────┼──────▶ Receive
       │                       │           │
       │                       │           ▼
       │                       │       Extract
       │                       │           │
       │                       │           ▼
       │                       │    [Files at Destination]
       │                       │
       │                       │
    Cleanup ◀──────────────────┴─────▶ Cleanup
```

### With Backup (Default)

```
Remote Server Process
═══════════════════════

Before Transfer:
    [Current Files]
          │
          ▼
      Compress
          │
          ▼
    ~/backups/backup_YYYYMMDD_HHMMSS.tar.gz  ◀── Backup Created
          
Transfer:
    [New Files Arrive]
          │
          ▼
     Replace Old Files
          │
          ▼
    [Updated Files]

Backup Still Safe:
    ~/backups/backup_YYYYMMDD_HHMMSS.tar.gz  ◀── Can rollback if needed
```

---

## ⬇️ Download Workflow (Simplified)

```
Remote Server                         Your Computer
═════════════                         ═════════════

    [Files]
       │
       ▼
   Compress────────────────────┐
    (tar.gz)                   │
       │                       │
       ▼                       │
   Transfer ───────────────────┼──────▶ Receive
       │                       │           │
       │                       │           ▼
       │                       │       Extract
       │                       │           │
       │                       │           ▼
       │                       │    [Files at Destination]
       │                       │
    Cleanup ◀──────────────────┴─────▶ Cleanup
```

Note: Backup is NOT created for downloads (only for uploads)

---

## 🔍 Script Validation Process

```
Your Script
    │
    ▼
┌─────────────────────┐
│  Syntax Check       │──── bash -n
│  (Grammar)          │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Structure Check     │──── Count brackets, braces, etc.
│ (Balance)           │
└──────────┬──────────┘
           │
           ▼
        Valid?
         ╱  ╲
       Yes   No
        │     │
        ▼     ▼
    Execute  Log Error
        │     │
        ▼     ▼
  Capture   Continue
   Output   (Don't fail action)
```

### What Gets Checked

```
✓ Empty script?                  "   " → ERROR
✓ Syntax errors?                 "if [" → ERROR  
✓ Unmatched braces?              "{ ... {" → ERROR
✓ Unmatched parentheses?         "( ... ((" → ERROR
✓ Unmatched brackets?            "[ ... ]]]" → ERROR
✓ Incomplete pipelines?          "ls | | grep" → ERROR
```

---

## 💾 Backup System

### Backup Lifecycle

```
Deployment 1:
    [Transfer] → [Backup Created: backup_app_20250101_120000.tar.gz]

Deployment 2:
    [Transfer] → [Backup Created: backup_app_20250102_120000.tar.gz]

Deployment 3:
    [Transfer] → [Backup Created: backup_app_20250103_120000.tar.gz]

    ...

Deployment 11:
    [Transfer] → [Backup Created: backup_app_20250111_120000.tar.gz]
                 [Delete Oldest: backup_app_20250101_120000.tar.gz]

Result: Only last 10 backups kept
```

### Backup File Structure

```
~/backups/
├── backup_html_20250110_120000_abc123.tar.gz  ← Newest
├── backup_html_20250109_120000_def456.tar.gz
├── backup_html_20250108_120000_ghi789.tar.gz
├── backup_html_20250107_120000_jkl012.tar.gz
├── backup_html_20250106_120000_mno345.tar.gz
├── backup_html_20250105_120000_pqr678.tar.gz
├── backup_html_20250104_120000_stu901.tar.gz
├── backup_html_20250103_120000_vwx234.tar.gz
├── backup_html_20250102_120000_yza567.tar.gz
└── backup_html_20250101_120000_bcd890.tar.gz  ← Oldest (will be deleted next)
```

---

## ⚡ Decision Trees

### When Does Backup Run?

```
Start
  │
  ▼
Direction = Upload?
  ├─ No ──────────────────────────────────────────────────┐
  │                                                         │
  ▼                                                         │
backup_before_transfer = true?                             │
  ├─ No ──────────────────────────────────────────────┐   │
  │                                                     │   │
  ▼                                                     │   │
Destination Exists?                                     │   │
  ├─ No ─ "Skip: First Deploy" ──────────────────────┐ │   │
  │                                                    │ │   │
  ▼                                                    │ │   │
CREATE BACKUP                                          │ │   │
  │                                                    │ │   │
  └────────────────────────────────────────────────────┼─┼───┤
                                                       │ │   │
                                                       ▼ ▼   ▼
                                                    Continue to Transfer
```

### Script Execution Decision

```
post_script provided?
  ├─ No ──────────────────────────────────────────────┐
  │                                                     │
  ▼                                                     │
post_script_path provided?                             │
  ├─ No ──────────────────────────────────────────────┤
  │                                                     │
  ▼                                                     │
VALIDATE SCRIPT                                         │
  │                                                     │
  ▼                                                     │
Valid?                                                  │
  ├─ No ─ Log Error ─────────────────────────────────┐│
  │                                                    ││
  ▼                                                    ││
EXECUTE SCRIPT                                         ││
  │                                                    ││
  ▼                                                    ││
Succeeded?                                             ││
  ├─ No ─ Log Error ────────────────────────────────┐││
  │                                                   │││
  ▼                                                   │││
Capture Output                                        │││
  │                                                   │││
  └───────────────────────────────────────────────────┼┼┤
                                                      │││
                                                      ▼▼▼
                                               Continue to Cleanup
```

---

## 🎯 Error Handling Matrix

| Phase | Error Type | Action Response |
|-------|-----------|----------------|
| **1. SSH Setup** | Invalid key | ❌ Stop immediately |
| **1. SSH Setup** | Permission error | ❌ Stop immediately |
| **2. Connection** | Connection failed | ❌ Stop immediately |
| **3. Backup** | Backup failed | ❌ Stop immediately |
| **3. Backup** | Retention cleanup issue | ⚠️ Warn, continue |
| **4. Transfer** | Source missing | ❌ Stop immediately |
| **4. Transfer** | Transfer failed | ❌ Stop immediately |
| **5. Script** | Validation failed | ⚠️ Log, skip execution |
| **5. Script** | Execution failed | ⚠️ Log, continue |
| **6. Cleanup** | Any error | ⚠️ Log, continue |

Legend:
- ❌ = Critical error, stop action
- ⚠️ = Non-critical, log and continue

---

## 📈 State Transitions

```
┌─────────────┐
│  STARTING   │
└──────┬──────┘
       │
       ▼
┌─────────────┐     Error      ┌─────────────┐
│   SETUP     │───────────────▶│   FAILED    │
└──────┬──────┘                └──────┬──────┘
       │                              │
       ▼                              │
┌─────────────┐     Error             │
│ CONNECTING  │──────────────────────┤
└──────┬──────┘                       │
       │                              │
       ▼                              │
┌─────────────┐     Error             │
│  BACKING UP │──────────────────────┤
└──────┬──────┘                       │
       │                              │
       ▼                              │
┌─────────────┐     Error             │
│ TRANSFERRING│──────────────────────┤
└──────┬──────┘                       │
       │                              │
       ▼                              │
┌─────────────┐                       │
│  SCRIPTING  │ (errors logged only) │
└──────┬──────┘                       │
       │                              │
       ▼                              │
┌─────────────┐                       │
│  CLEANING   │◀─────────────────────┘
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  COMPLETE   │
└─────────────┘
```

---

## 🔐 Security Flow

```
SSH Key (Secret)
      │
      ▼
┌──────────────────┐
│  Validate Format │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Save to Temp     │
│ (Permissions 600)│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Use for Ops     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Overwrite w/     │
│ Zeros            │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Delete File     │
└────────┬─────────┘
         │
         ▼
    No Traces
```

---

## 📊 Outputs Timeline

```
Timeline:  ───────────────────────────────────────────────▶

Phase 1:   [SSH Setup]
Outputs:   (none - internal state only)

Phase 2:   [Connection Test]
Outputs:   (exit code indicates success/failure)

Phase 3:   [Backup]
Outputs:   backup_created ────┐
           backup_path    ────┼─▶ Available for remaining workflow
           backup_size    ────┘

Phase 4:   [Transfer]
Outputs:   success ───────────┐
           error      ────────┼─▶ Available for remaining workflow
                               │
Phase 5:   [Post-Script]       │
Outputs:   script_executed ───┤
           script_output   ────┼─▶ Available for remaining workflow
           script_error    ────┤
                               │
Phase 6:   [Cleanup]           │
Outputs:   (none)              │
                               │
                               ▼
                         All outputs available
                         for your workflow steps
```

---

## 🎨 Color-Coded Component Map

```
Components by Function:

🔵 Input Processing
   ├─ SSH credentials
   ├─ Source/destination paths
   └─ Configuration options

🟢 Validation
   ├─ SSH key format
   ├─ Connection test
   ├─ Source/destination existence
   └─ Script syntax & structure

🟡 Core Operations
   ├─ Backup creation
   ├─ File compression
   ├─ File transfer
   └─ File extraction

🟣 Optional Features
   ├─ Post-script execution
   └─ Script validation

🔴 Error Handling
   ├─ Critical errors (stop)
   ├─ Non-critical errors (log)
   └─ Cleanup on failure

🟤 Cleanup
   ├─ Temporary files
   ├─ SSH credentials
   └─ Remote temp files

📊 Output Generation
   ├─ Backup information
   ├─ Transfer status
   └─ Script results
```

---

## 💡 Quick Reference

### Most Common Flow

```
1. You: Provide SSH credentials + files
         ↓
2. Action: Validates everything
         ↓
3. Action: Creates backup (if uploading)
         ↓
4. Action: Transfers files (compressed)
         ↓
5. Action: Runs your script (if provided)
         ↓
6. Action: Cleans up securely
         ↓
7. You: Check outputs for status
```

### Minimal Configuration

```yaml
- uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.HOST }}
    username: ${{ secrets.USER }}
    key: ${{ secrets.KEY }}
    source: "dist/"
    destination: "/var/www/html/"
```

Result:
- ✅ Backup created automatically
- ✅ Files transferred securely
- ✅ Scripts validated (if provided)
- ✅ Everything cleaned up

---

## 📚 Related Documentation

- **README.md** - Main documentation
- **ARCHITECTURE.md** - Technical deep-dive
- **EXAMPLES.md** - Usage patterns
- **QUICK_REFERENCE.md** - Quick lookup

---

**Remember**: Each phase is independent. If something fails, you know exactly where to look!
