# 071: JSON-Only Test Configuration Cleanup Investigation

## Status: Proposed

## Summary
Conduct a thorough investigation to identify and clean up any remaining deviations from the JSON-only test configuration pattern implemented in #070. Ensure complete alignment across the codebase.

## Investigation Checklist

### 1. Configuration File Audit

#### Hardcoded Values in BUILD Files
- [ ] Search all BUILD.bazel files for `env = {` blocks
- [ ] Find localhost:port patterns in BUILD files
- [ ] Identify service names/URLs in test definitions
- [ ] Check for database names, topics, queues in BUILD files

```bash
# Search commands
rg -t bazel 'env\s*=\s*{' --glob '*/BUILD.bazel'
rg -t bazel '(localhost|127\.0\.0\.1):\d+' --glob '*/BUILD.bazel'
rg -t bazel 'test\.py.*env.*=.*{' -A 5
```

#### Duplicate Configuration Sources
- [ ] Find other YAML/JSON config files
- [ ] Check for config.py modules with defaults
- [ ] Identify environment-specific config files
- [ ] Look for test-specific configuration classes

```bash
fd -e yaml -e yml -e json | grep -i config | grep -v test_config
rg 'class.*Config' --glob '**/test*/**/*.py'
```

### 2. BUILD File Compliance

#### Test Macro Usage
- [ ] Find py_test rules not using integration_test/unit_test macros
- [ ] Identify custom test wrappers bypassing standard macros
- [ ] Check for missing test_config.json dependencies
- [ ] Verify proper data attribute usage

```bash
# Find non-compliant tests
rg '^py_test\(' --glob '*/BUILD.bazel' -B 2 -A 10 | grep -v test_env.bzl
rg 'native\.py_test' --glob '*/BUILD.bazel'
```

#### Environment Variables in BUILD
- [ ] Check for POSTGRES_*, KAFKA_*, REDIS_* patterns
- [ ] Find SERVICE_URL, API_ENDPOINT definitions
- [ ] Look for _HOST, _PORT suffixes
- [ ] Identify test-specific overrides

### 3. Source Code Patterns

#### Direct Environment Access
- [ ] Find os.environ[] usage in tests
- [ ] Check os.getenv() with defaults
- [ ] Look for environ.get patterns
- [ ] Identify config not using test_config module

```python
# Bad patterns to find:
os.environ["POSTGRES_HOST"]
os.getenv("KAFKA_SERVERS", "localhost:9092")
config = {"host": os.environ.get("DB_HOST", "localhost")}
```

#### Hardcoded Test Configuration
- [ ] Service URLs in test files
- [ ] Default ports in fixtures
- [ ] Database names in tests
- [ ] Timeout values not from config

```bash
rg 'localhost:\d+|127\.0\.0\.1:\d+' --glob '**/*test*.py'
rg 'http://localhost|https://localhost' --glob '**/*test*.py'
rg 'timeout\s*=\s*\d+' --glob '**/*test*.py'
```

### 4. Configuration Completeness

#### Missing from config.yaml
- [ ] Service endpoints not in config
- [ ] Test behavior settings missing
- [ ] Resource limits undefined
- [ ] Feature flags not centralized

#### Should be Runtime Only
- [ ] Identify values that change per environment
- [ ] Document which values are override-only
- [ ] Create override documentation

### 5. Secret Management Audit

#### Secret Detection
- [ ] Scan config.yaml for sensitive values
- [ ] Check BUILD files for credentials
- [ ] Review test fixtures for secrets
- [ ] Verify CI scripts don't expose secrets

```bash
rg -i 'password|secret|key|token|credential' tests/config.yaml
rg -t bazel 'PASSWORD|SECRET|KEY|TOKEN.*=.*["\']' 
rg 'api_key|auth_token|client_secret' --glob '**/*test*.py'
```

#### Secret Documentation
- [ ] List all required runtime secrets
- [ ] Document injection methods
- [ ] Create CI/CD examples
- [ ] Add pre-commit secret scanning

### 6. .bazelrc Cleanup

#### Profile Audit
- [ ] Check each --config for env vars
- [ ] Verify profiles only have flags
- [ ] Remove duplicate configurations
- [ ] Document profile purposes

```bash
rg '--test_env=' .bazelrc
rg 'test:.*--test_env' .bazelrc
```

### 7. Test Infrastructure

#### Test Helpers
- [ ] Review TestConfig implementation
- [ ] Check config loading in conftest.py
- [ ] Verify fixtures use central config
- [ ] Update test base classes

#### Container Tests
- [ ] Check docker-compose env vars
- [ ] Review testcontainer configurations
- [ ] Verify service dependency declarations
- [ ] Update container helper utilities

### 8. Documentation Review

#### Outdated Examples
- [ ] README files with old patterns
- [ ] Code comments with env vars
- [ ] Wiki/docs with old configuration
- [ ] CI/CD templates needing updates

#### Missing Documentation
- [ ] How to add new config values
- [ ] Override patterns and examples
- [ ] Secret management guide
- [ ] Troubleshooting guide

### 9. CI/CD Alignment

#### GitHub Actions
- [ ] Check workflows files for hardcoded values
- [ ] Verify secret injection patterns
- [ ] Update build matrices
- [ ] Fix cache key generation

#### Other CI Systems
- [ ] Jenkins configurations
- [ ] CI/CD configuration files
- [ ] Local development scripts
- [ ] Docker build arguments

### 10. Validation Implementation

#### Automated Checks
- [ ] Create config validation script
- [ ] Add pre-commit hooks
- [ ] Implement CI compliance checks
- [ ] Build linting rules

#### Test Coverage
- [ ] Verify config loading tests exist
- [ ] Add override behavior tests
- [ ] Test secret injection patterns
- [ ] Validate profile behavior

## Expected Deliverables

1. **Violation Report**: List of all files requiring updates
2. **Migration PRs**: Separate PRs for each component
3. **Validation Suite**: Automated compliance checking
4. **Documentation Updates**: Comprehensive guides
5. **Team Training**: Knowledge sharing session

## Success Metrics

- Zero hardcoded configuration in BUILD files
- All tests use JSON config as primary source
- No duplicate configuration definitions
- Clear secret/config separation
- 100% macro adoption for tests
- Passing compliance checks in CI

## Timeline

- Week 1: Investigation and report generation
- Week 2: Critical path migrations
- Week 3: Documentation and validation
- Week 4: Team training and rollout

## References

- #070: Original YAML central configuration
- Bazel hermetic testing documentation
- Test configuration best practices
- Secret management guidelines
