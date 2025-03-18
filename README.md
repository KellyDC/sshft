# sshft (SSH File Transfer)

This action allows you to transfer files or directories to a remote server via SSH using SCP.

## Inputs

| Name          | Description                           | Required | Default |
| ------------- | ------------------------------------- | -------- | ------- |
| `host`        | SSH host to connect to                | Yes      | -       |
| `port`        | SSH port                              | No       | 22      |
| `username`    | SSH username                          | Yes      | -       |
| `key`         | SSH private key                       | Yes      | -       |
| `passphrase`  | Passphrase for the SSH private key    | No       | -       |
| `source`      | Source file or directory to transfer  | Yes      | -       |
| `destination` | Destination path on the remote server | Yes      | -       |
| `flags`       | Additional flags to pass to scp       | No       | -       |

## Example Usage

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Transfer files to server
        uses: kellydc/sshft@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "dist/"
          destination: "/var/www/html/"

      - name: Transfer single file
        uses: kellydc/sshft@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "config.json"
          destination: "/etc/myapp/config.json"
```

## Security Notes

- Always store your SSH private key as a GitHub secret.
- The action will automatically detect whether to use recursive copying for directories.
- For more complex SSH operations, consider using a full-featured SSH action.
