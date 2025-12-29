# sshft Action Examples

Complete examples for the sshft GitHub Action with security features.

## Table of Contents

- [Security Notes](#security-notes)
- [Backup Feature Examples](#backup-feature-examples)
- [Basic Examples](#basic-examples)
- [Web Application Deployment](#web-application-deployment)
- [Database Operations](#database-operations)
- [Docker Container Management](#docker-container-management)
- [System Maintenance](#system-maintenance)
- [Error Handling](#error-handling)
- [Best Practices](#best-practices)

## Security Notes

### Blocked Commands
The action blocks potentially dangerous commands:
- **System destruction**: `rm -rf /`, `rm -rf ~`, `rm -rf *`
- **Disk operations**: `dd` to/from devices, `mkfs`, `fdisk`, `parted`, `wipefs`
- **System control**: `shutdown`, `reboot`, `halt`, `poweroff`
- **Privilege escalation**: `sudo`, `su`
- **Remote code execution**: `curl/wget | bash`, `curl/wget | sh`
- **Fork bombs**: `:(){:|:&};:`
- **Security bypass**: `iptables -F`, `ufw disable`, `setenforce 0`

### Resource Limits
Scripts execute with strict limits:
- **Processes**: Max 100
- **File size**: Max 1GB
- **CPU time**: Max 5 minutes
- **Memory**: Max 2GB
- **Timeout**: 10 minutes total

### File Transfer Limits
- **Max file/directory size**: 2GB for uploads, 10GB for downloads
- **Disk space check**: Validates 20% extra buffer available
- **Symlink validation**: Checks and warns about symlinks

## Backup Feature Examples

The action automatically backs up the destination before uploading files. This ensures you can rollback if something goes wrong.

### Basic Backup Usage

```yaml
- name: Deploy with automatic backup
  id: deploy
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "dist/"
    destination: "/var/www/html/"
    # backup_before_transfer: true (enabled by default)

- name: Display backup information
  if: steps.deploy.outputs.backup_created == 'true'
  run: |
    echo "✓ Backup created successfully"
    echo "Location: ${{ steps.deploy.outputs.backup_path }}"
    echo "Size: ${{ steps.deploy.outputs.backup_size }}"
```

### Disable Backup for Temporary Files

```yaml
- name: Upload temp files without backup
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "temp-data/"
    destination: "/tmp/uploads/"
    backup_before_transfer: false
```

### Deploy with Rollback Capability

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy application
        id: deploy
        uses: kellydc/sshft@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "dist/"
          destination: "/var/www/html/"
          post_script: |
            sudo systemctl restart nginx
            
            # Test if application is responding
            sleep 5
            if ! curl -f http://localhost:8080/health; then
              echo "ERROR: Health check failed"
              exit 1
            fi
            echo "Deployment successful"
      
      - name: Rollback on failure
        if: failure() && steps.deploy.outputs.backup_created == 'true'
        uses: kellydc/sshft@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "rollback-not-needed"  # Dummy source
          destination: "/var/www/html/"
          backup_before_transfer: false
          post_script: |
            BACKUP_PATH="${{ steps.deploy.outputs.backup_path }}"
            DEST_DIR="/var/www/html"
            
            echo "Rolling back to backup: $BACKUP_PATH"
            
            # Remove failed deployment
            rm -rf "$DEST_DIR"/*
            
            # Extract backup
            tar -xzf "$BACKUP_PATH" -C "$(dirname "$DEST_DIR")"
            
            sudo systemctl restart nginx
            echo "Rollback completed successfully"
```

### Save Backup Information for Later

```yaml
- name: Deploy and save backup info
  id: deploy
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "app/"
    destination: "/var/www/app/"

- name: Save backup info to artifact
  if: steps.deploy.outputs.backup_created == 'true'
  run: |
    mkdir -p backup-info
    cat > backup-info/deployment.json << EOF
    {
      "backup_path": "${{ steps.deploy.outputs.backup_path }}",
      "backup_size": "${{ steps.deploy.outputs.backup_size }}",
      "deployment_time": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
      "commit_sha": "${{ github.sha }}"
    }
    EOF

- name: Upload backup info
  uses: actions/upload-artifact@v3
  if: steps.deploy.outputs.backup_created == 'true'
  with:
    name: backup-info
    path: backup-info/deployment.json
```

### Monitor Backup Storage

```yaml
- name: Deploy with backup monitoring
  id: deploy
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "dist/"
    destination: "/var/www/html/"
    post_script: |
      echo "=== Backup Storage Report ==="
      BACKUP_DIR="$HOME/backups"
      
      # Total size of all backups
      TOTAL_SIZE=$(du -sh "$BACKUP_DIR" 2>/dev/null | cut -f1)
      echo "Total backup size: $TOTAL_SIZE"
      
      # Number of backups
      BACKUP_COUNT=$(ls -1 "$BACKUP_DIR" | wc -l)
      echo "Number of backups: $BACKUP_COUNT"
      
      # List recent backups
      echo "Recent backups:"
      ls -lht "$BACKUP_DIR" | head -n 6

- name: Alert if backup storage is high
  if: success()
  run: |
    echo "Deployment completed with backup"
    echo "Backup: ${{ steps.deploy.outputs.backup_path }}"
    echo "Size: ${{ steps.deploy.outputs.backup_size }}"
```

### First-Time Deployment (No Backup)

```yaml
- name: Initial deployment to new server
  id: deploy
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.NEW_SERVER_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "dist/"
    destination: "/var/www/html/"
    # Backup will be skipped automatically if destination doesn't exist

- name: Check if it was first deployment
  run: |
    if [ "${{ steps.deploy.outputs.backup_created }}" != "true" ]; then
      echo "This was a first-time deployment (no backup needed)"
    else
      echo "Updated existing deployment (backup created)"
    fi
```

## Basic Examples

### Simple Service Restart

```yaml
- name: Deploy and restart service
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "dist/"
    destination: "/var/www/html/"
    post_script: |
      sudo systemctl restart nginx
      echo "Nginx restarted successfully"
```

### File Permission Adjustment

```yaml
- name: Upload and set permissions
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "scripts/"
    destination: "/opt/scripts/"
    post_script: |
      chmod +x /opt/scripts/*.sh
      chown root:root /opt/scripts/*
      echo "Permissions updated"
```

### Create Symbolic Links

```yaml
- name: Deploy and create symlinks
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "release-v2.0/"
    destination: "/opt/releases/v2.0/"
    post_script: |
      ln -sf /opt/releases/v2.0 /opt/app/current
      echo "Symlink created to new release"
```

## Web Application Deployment

### Node.js Application

```yaml
- name: Deploy Node.js app
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "build/"
    destination: "/var/www/myapp/"
    post_script: |
      cd /var/www/myapp
      npm install --production
      pm2 restart myapp || pm2 start server.js --name myapp
      echo "Node.js application deployed and restarted"
```

### PHP Application with Composer

```yaml
- name: Deploy PHP app
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "src/"
    destination: "/var/www/html/"
    post_script: |
      cd /var/www/html
      composer install --no-dev --optimize-autoloader
      php artisan cache:clear
      php artisan config:cache
      sudo systemctl reload php8.2-fpm
      echo "PHP application deployed"
```

### Static Site with Cache Clearing

```yaml
- name: Deploy static site
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "public/"
    destination: "/var/www/static/"
    post_script: |
      # Clear CDN cache
      curl -X POST "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/purge_cache" \
        -H "Authorization: Bearer ${CF_API_TOKEN}" \
        -H "Content-Type: application/json" \
        --data '{"purge_everything":true}'
      
      # Restart nginx
      sudo systemctl reload nginx
      echo "Static site deployed and cache cleared"
```

## Database Operations

### Run Database Migrations

```yaml
- name: Deploy with migrations
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "app/"
    destination: "/opt/myapp/"
    post_script: |
      cd /opt/myapp
      ./migrate.sh up
      echo "Database migrations completed"
```

### Backup Before Deployment

```yaml
- name: Backup and deploy
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "updates/"
    destination: "/var/app/"
    post_script: |
      # Create backup
      BACKUP_DIR="/var/backups/app/$(date +%Y%m%d_%H%M%S)"
      mkdir -p $BACKUP_DIR
      cp -r /var/app/* $BACKUP_DIR/
      
      # Also backup database
      mysqldump -u dbuser -p"$DB_PASSWORD" mydb > $BACKUP_DIR/database.sql
      
      echo "Backup created at $BACKUP_DIR"
```

## Docker Container Management

### Rebuild and Restart Container

```yaml
- name: Deploy Docker app
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "docker/"
    destination: "/opt/docker/myapp/"
    post_script: |
      cd /opt/docker/myapp
      docker-compose down
      docker-compose build --no-cache
      docker-compose up -d
      echo "Docker container rebuilt and started"
```

### Update Running Container

```yaml
- name: Update container config
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "configs/"
    destination: "/opt/docker/configs/"
    post_script: |
      docker-compose restart
      docker ps --filter "name=myapp" --format "{{.Status}}"
      echo "Container restarted with new config"
```

## System Maintenance

### Clean Old Files

```yaml
- name: Deploy and cleanup
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "releases/v3.0/"
    destination: "/opt/releases/v3.0/"
    post_script: |
      # Remove releases older than 7 days
      find /opt/releases -maxdepth 1 -type d -mtime +7 -exec rm -rf {} \;
      
      # Clean temp files
      rm -rf /tmp/app_*
      
      echo "Old files cleaned up"
```

### System Health Check

```yaml
- name: Deploy and check health
  id: deploy
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "app/"
    destination: "/var/app/"
    post_script: |
      echo "=== System Health Check ==="
      echo "Hostname: $(hostname)"
      echo "Uptime: $(uptime)"
      echo "Disk Usage: $(df -h / | tail -n1 | awk '{print $5}')"
      echo "Memory: $(free -h | grep Mem | awk '{print $3 "/" $2}')"
      echo "CPU Load: $(uptime | awk -F'load average:' '{print $2}')"
```

### Log Rotation

```yaml
- name: Deploy and rotate logs
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "app/"
    destination: "/var/app/"
    post_script: |
      # Archive old logs
      ARCHIVE_DIR="/var/log/archive/$(date +%Y%m)"
      mkdir -p $ARCHIVE_DIR
      
      find /var/app/logs -name "*.log" -mtime +30 -exec gzip {} \;
      find /var/app/logs -name "*.log.gz" -exec mv {} $ARCHIVE_DIR/ \;
      
      echo "Logs rotated and archived"
```

## Error Handling

### Using Remote Script with Fallback

```yaml
- name: Deploy with fallback script
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "app/"
    destination: "/var/app/"
    post_script_path: "/opt/scripts/deploy.sh"

- name: Check if script executed
  if: steps.deploy.outputs.script_executed != 'true'
  run: echo "Warning: Post-deployment script did not execute"
```

### Conditional Script Execution

```yaml
- name: Deploy application
  id: deploy
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "dist/"
    destination: "/var/www/html/"
    post_script: |
      if [ -f /var/www/html/.maintenance ]; then
        echo "Maintenance mode detected, skipping restart"
        exit 0
      fi
      
      sudo systemctl restart nginx
      echo "Service restarted"

- name: Handle script errors
  if: steps.deploy.outputs.script_error != ''
  run: |
    echo "Script error occurred: ${{ steps.deploy.outputs.script_error }}"
    # Send notification, create issue, etc.
```

### Retry Logic in Script

```yaml
- name: Deploy with retry logic
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "app/"
    destination: "/var/app/"
    post_script: |
      MAX_RETRIES=3
      RETRY_COUNT=0
      
      while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
        if sudo systemctl restart myapp; then
          echo "Service restarted successfully"
          exit 0
        fi
        
        RETRY_COUNT=$((RETRY_COUNT + 1))
        echo "Retry $RETRY_COUNT/$MAX_RETRIES failed, waiting 5 seconds..."
        sleep 5
      done
      
      echo "Failed to restart service after $MAX_RETRIES attempts"
      exit 1
```

## Security Notes

### Blocked Commands
The action blocks potentially dangerous commands including:
- `rm -rf /`, `rm -rf ~`, `rm -rf *` - System destruction
- `dd` to/from devices - Disk overwrites
- `shutdown`, `reboot`, `halt` - System control
- `sudo`, `su` - Privilege escalation
- `curl/wget | bash` - Remote code execution
- Fork bombs: `:(){:|:&};:`
- Disk formatting: `mkfs`, `fdisk`, `parted`

### Resource Limits
Scripts execute with:
- Max 100 processes
- Max 1GB file size
- Max 5 minutes CPU time
- Max 2GB memory
- 10-minute timeout

## Best Practices

### 1. Use Existing Remote Scripts for Complex Logic

For complex deployment logic, maintain scripts on the remote server:

```yaml
- name: Deploy using remote script
  uses: kellydc/sshft@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: "dist/"
    destination: "/var/www/html/"
    post_script_path: "/opt/scripts/post-deploy.sh"
```

**Benefits:**
- Easier to test and debug on the server
- Version control on the server side
- Reusable across multiple deployments
- Better for complex multi-step operations

### 2. Keep Inline Scripts Simple

Use inline scripts for simple, one-off commands:

```yaml
post_script: |
  sudo systemctl restart nginx
  echo "Deployment completed at $(date)"
```

### 3. Include Error Messages

Always provide clear feedback:

```yaml
post_script: |
  if ! sudo systemctl restart nginx; then
    echo "ERROR: Failed to restart nginx"
    exit 1
  fi
  echo "SUCCESS: Nginx restarted"
```

### 4. Use Environment Variables Securely

Never hardcode sensitive data in scripts:

```yaml
# BAD - Don't do this
post_script: |
  mysql -u root -pMyPassword123 ...

# GOOD - Use server-side environment or config files
post_script: |
  source /etc/myapp/config.sh
  mysql -u "$DB_USER" -p"$DB_PASS" ...
```

### 5. Implement Proper Logging

```yaml
post_script: |
  LOG_FILE="/var/log/deployments/$(date +%Y%m%d_%H%M%S).log"
  mkdir -p /var/log/deployments
  
  {
    echo "Deployment started at $(date)"
    sudo systemctl restart myapp
    echo "Deployment completed at $(date)"
  } | tee -a "$LOG_FILE"
```

### 6. Test Scripts Before Using

Always test your scripts directly on the server before using them in the action:

```bash
# SSH into your server and test
ssh user@host "bash -c 'your script commands here'"
```

### 7. Use Absolute Paths

Always use absolute paths in scripts to avoid confusion:

```yaml
# GOOD
post_script: |
  cd /var/www/html
  /usr/local/bin/node --version

# BAD - May fail depending on working directory
post_script: |
  cd ../www/html
  node --version
```

### 8. Handle Long-Running Processes

For processes that take time, provide progress updates:

```yaml
post_script: |
  echo "Starting database migration..."
  php /var/www/html/artisan migrate --force
  echo "Migration completed"
  
  echo "Rebuilding cache..."
  php /var/www/html/artisan cache:clear
  echo "Cache cleared"
  
  echo "Restarting services..."
  sudo systemctl restart php8.2-fpm
  echo "All tasks completed successfully"
```

### 9. Validate Before Executing Critical Commands

```yaml
post_script: |
  # Validate before restarting
  if ! nginx -t; then
    echo "ERROR: Nginx configuration is invalid"
    exit 1
  fi
  
  sudo systemctl reload nginx
  echo "Nginx reloaded successfully"
```

### 10. Document Your Scripts

Use comments to explain what your script does:

```yaml
post_script: |
  # Check if application is already running
  if pgrep -x "myapp" > /dev/null; then
    echo "Stopping existing process..."
    pkill -x "myapp"
  fi
  
  # Start the application in background
  cd /opt/myapp
  nohup ./start.sh > /var/log/myapp.log 2>&1 &
  
  echo "Application started successfully"
```

## Troubleshooting

### Script Not Executing

Check the action outputs:

```yaml
- name: Debug script execution
  if: always()
  run: |
    echo "Script executed: ${{ steps.deploy.outputs.script_executed }}"
    echo "Script error: ${{ steps.deploy.outputs.script_error }}"
    echo "Script output: ${{ steps.deploy.outputs.script_output }}"
```

### Common Issues

1. **Permission denied**: Ensure the user has sudo privileges or appropriate permissions
2. **Command not found**: Use absolute paths for commands
3. **Syntax errors**: Test your script locally first
4. **Timeout**: Break long-running tasks into smaller steps
5. **Environment variables**: Source necessary configuration files in your script

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Bash Scripting Guide](https://www.gnu.org/software/bash/manual/)
- [SSH Best Practices](https://www.ssh.com/academy/ssh/command)
