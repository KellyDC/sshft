# Security Policy

## Supported Versions

We provide security updates for the following versions:

| Version | Supported          |
| ------- | ------------------ |
| 1.x.x   | ✅ Yes             |
| < 1.0   | ❌ No              |

## Reporting a Vulnerability

We take security vulnerabilities seriously. If you discover a security vulnerability in this GitHub Action, please report it to us as follows:

### Where to Report

1. **GitHub Security Advisories**: Use the [Security tab](https://github.com/KellyDC/sshft/security/advisories) in this repository to report vulnerabilities privately.

2. **Submit an issue**: Contact us by opening a new issue in this repository with the "security" label.  

### What to Include

Please include as much of the following information as possible:

- Type of vulnerability (e.g., remote code execution, information leakage, etc.)
- Full paths of source file(s) related to the vulnerability
- Step-by-step instructions to reproduce the issue
- Proof-of-concept or exploit code (if possible)
- Impact of the vulnerability and how an attacker might exploit it

### Response Timeline

- **Initial Response**: We will acknowledge receipt of your vulnerability report within 2 business days.
- **Status Updates**: We will provide status updates on the vulnerability every 5 business days.
- **Resolution**: We aim to resolve critical vulnerabilities within 30 days and moderate vulnerabilities within 60 days.

### Security Best Practices for Users

When using this action, please follow these security best practices:

1. **SSH Keys**: 
   - Always store SSH private keys as GitHub secrets
   - Use keys with strong passphrases when possible
   - Rotate SSH keys regularly
   - Use dedicated keys for CI/CD (not personal keys)

2. **Host Key Verification**:
   - Keep `strict_host_key_checking` enabled (default) in production
   - Only disable for trusted internal environments

3. **Access Control**:
   - Use principle of least privilege for SSH user accounts
   - Limit SSH access to specific IP ranges when possible
   - Regularly audit SSH access logs

4. **File Permissions**:
   - Ensure destination directories have appropriate permissions
   - Be cautious with file transfers to system directories

5. **Network Security**:
   - Use SSH over secure networks when possible
   - Consider VPN or private networks for sensitive transfers

## Security Features

This action includes several security features:

- SSH key validation before use
- Secure temporary file handling
- Automatic cleanup of sensitive data
- Connection timeout and retry logic
- Support for SSH key passphrases
- Configurable host key checking

## Changelog

Security-related changes will be clearly marked in our release notes and changelog.

## Acknowledgments

We appreciate the security research community and will acknowledge researchers who responsibly disclose vulnerabilities (with their permission).