# Contributing to sshft

Thank you for considering contributing to sshft! This document provides guidelines and information for contributors.

## How to Contribute

### Reporting Issues

1. **Search existing issues** first to avoid duplicates
2. **Use issue templates** when available
3. **Provide detailed information**:
   - Operating system and version
   - GitHub Actions runner environment
   - Steps to reproduce
   - Expected vs actual behavior
   - Relevant logs or error messages

### Suggesting Features

1. **Check existing feature requests** first
2. **Describe the use case** clearly
3. **Explain the benefit** to users
4. **Consider security implications**

### Code Contributions

#### Before You Start

1. **Fork the repository**
2. **Create a feature branch** from `main`
3. **Discuss major changes** via issues first

#### Development Guidelines

1. **Follow existing code style**
2. **Add comments** for complex logic
3. **Update documentation** for user-facing changes
4. **Add tests** for new functionality
5. **Ensure security best practices**

#### Security Considerations

When contributing, please keep these security aspects in mind:

- **Input validation**: Validate all user inputs
- **Secrets handling**: Never log or expose secrets
- **File permissions**: Use appropriate permissions for temporary files
- **Error messages**: Don't expose sensitive information in errors
- **Dependencies**: Keep dependencies minimal and up-to-date

#### Testing

1. **Run all existing tests**
2. **Add tests for new features**
3. **Test error conditions**
4. **Verify cleanup operations**

#### Documentation

1. **Update README.md** for user-facing changes
2. **Update action.yml** descriptions
3. **Add inline comments** for complex code
4. **Update examples** as needed

## Pull Request Process

1. **Ensure CI passes** on your branch
2. **Update the README.md** with details of changes if needed
3. **Update the action.yml** input/output descriptions if needed
4. **Follow the pull request template**
5. **Request review** from maintainers

### Pull Request Checklist

- [ ] Code follows the existing style
- [ ] Changes are tested
- [ ] Documentation is updated
- [ ] Security implications are considered
- [ ] No secrets or sensitive data in code
- [ ] Error handling is appropriate
- [ ] Cleanup operations work correctly

## Code of Conduct

### Our Pledge

We pledge to make participation in our project a harassment-free experience for everyone, regardless of age, body size, disability, ethnicity, sex characteristics, gender identity and expression, level of experience, education, socio-economic status, nationality, personal appearance, race, religion, or sexual identity and orientation.

### Our Standards

Examples of behavior that contributes to creating a positive environment include:

- Using welcoming and inclusive language
- Being respectful of differing viewpoints and experiences
- Gracefully accepting constructive criticism
- Focusing on what is best for the community
- Showing empathy towards other community members

### Enforcement

Project maintainers are responsible for clarifying the standards of acceptable behavior and are expected to take appropriate and fair corrective action in response to any instances of unacceptable behavior.

## Development Setup

### Prerequisites

- Git
- Basic understanding of GitHub Actions
- Familiarity with SSH and file transfer concepts
- Understanding of shell scripting

### Local Testing

```bash
# Clone your fork
git clone https://github.com/yourusername/sshft.git
cd sshft

# Test YAML syntax
python -c "import yaml; yaml.safe_load(open('action.yml'))"

# Run local validation
.github/workflows/scripts/validate.sh  # If available
```

### Testing Your Changes

1. **Create a test repository** with your changes
2. **Set up test SSH server** (Docker recommended)
3. **Test various scenarios**:
   - File uploads and downloads
   - Directory transfers
   - Error conditions
   - Permission issues
   - Network timeouts

## Release Process

1. **Version bump** in action.yml metadata
2. **Update CHANGELOG.md**
3. **Create release PR**
4. **Tag release** after merge
5. **Update major version tag** (e.g., v1)

## Questions?

- **Open an issue** for questions about development
- **Check existing documentation** first
- **Be patient** - maintainers may not respond immediately

Thank you for contributing! 🚀