# 070: YAML Central Configuration for Hermetic Test Environments

## Status: Completed

## Summary
Implemented a single source of truth for test configuration using YAML that generates JSON at build time, eliminating hardcoded environment variables and configuration duplication across the codebase.

## Problem Statement
The test environment configuration was scattered across multiple locations:
- Hardcoded environment variables in `test_env.bzl` macros
- Duplicated values in `.bazelrc` profiles
- Hardcoded secrets in BUILD files
- Inconsistent configuration between local and CI environments
- No clear separation between configuration and secrets

This violated Bazel's hermetic principles and made test configuration difficult to maintain.

## Solution Implemented

### 1. Single Source of Truth
Created `tests/config.yaml` as the canonical source for all test configuration:
```yaml
# Core Configuration
environment: test
postgres_host: localhost
postgres_port: 5432
postgres_db: ameide_test
kafka_bootstrap_servers: localhost:9092
redis_url: redis://localhost:6379
log_level: INFO
```

### 2. Build-Time JSON Generation
Implemented `generate_test_config` rule in `test_env.bzl`:
```python
generate_test_config(
    name = "test_config.json",
    src = "config.yaml",
    out = "test_config.json",
)
```

### 3. Simplified Test Macros
Updated `integration_test()` and `unit_test()` macros to only set:
```python
test_env = {
    "TEST_CONFIG_JSON": "$(location //tests:test_config.json)",
    "PYTHONDONTWRITEBYTECODE": "1",
}
```

### 4. Clean .bazelrc
Removed all hardcoded environment variables from `.bazelrc`. Profiles now only contain build/test flags:
```bash
# CI configuration
build:ci --noshow_progress
test:ci --test_output=all
# No hardcoded env vars!
```

### 5. Runtime Overrides
Environment-specific values and secrets are injected at runtime:
```bash
bazel test --config=ci \
  --test_env=POSTGRES_HOST=db-graph \
  --test_env=POSTGRES_PASSWORD=$SECRET \
  //tests/integration:test_name
```

## Implementation Details

### Key Files Modified

1. **tools/bazel/test_env.bzl**
   - Added `generate_test_config` rule
   - Removed all hardcoded environment variables
   - Simplified `integration_test()` and `unit_test()` macros

2. **tests/config.yaml**
   - Created as single source of truth
   - Contains all non-sensitive defaults
   - Includes comprehensive service configuration

3. **tests/BUILD.bazel**
   - Added `generate_test_config` target
   - Generates `test_config.json` from YAML

4. **.bazelrc**
   - Removed all `--test_env` flags with hardcoded values
   - Kept only build and test behavior flags

5. **tests/shared/test_config.py**
   - Enhanced to load JSON config
   - Provides `get_bazel_config()` method
   - Supports environment variable overrides

### Configuration Loading Pattern
```python
from tests.shared.test_config import config

# Values come from JSON
host = config.get_bazel_config("postgres_host")

# Secrets from environment
password = os.environ.get("POSTGRES_PASSWORD")
```

## Benefits Achieved

1. **True Hermeticity**: Tests produce consistent results regardless of host environment
2. **Single Source of Truth**: All configuration in one YAML file
3. **Clear Secret Boundary**: Obvious which values must be injected
4. **Reduced Duplication**: No more config scattered across files
5. **Easier Maintenance**: Change configuration in one place
6. **Better Security**: No accidental secret commits

## Validation

Implemented validation to prevent secrets in YAML:
```python
# In _parse_yaml_to_dict()
if any(secret in key.lower() for secret in ["password", "secret", "key", "token"]):
    if value and value != "false":
        fail("ERROR: Found potential secret '%s' in config.yaml")
```

## Migration Guide

For existing tests:
1. Remove hardcoded config from BUILD files
2. Use `integration_test()` or `unit_test()` macros
3. Access config via `config.get_bazel_config()`
4. Inject secrets via `--test_env` at runtime

## Future Improvements

1. **Deterministic YAML Parser**: Implement proper YAML parsing in Starlark
2. **Config Validation**: Add schema validation for config.yaml
3. **Secret Detection**: Enhanced scanning for accidental secrets
4. **Multi-Environment**: Support for multiple config files (dev, staging, prod)
5. **Config Documentation**: Auto-generate config reference from YAML

## Lessons Learned

1. **Starlark Limitations**: No native YAML support required creative solutions
2. **JSON Determinism**: Starlark's `json.encode()` automatically sorts keys
3. **Runtime vs Build Time**: Clear separation improves security
4. **Macro Design**: Simpler macros are easier to maintain
5. **Documentation**: Critical for adoption of new patterns

## References

- Original GitHub issue: #955 (--test_env_file proposal)
- Bazel hermetic testing principles
- Test environment configuration best practices

## Metrics

- Removed: ~50 hardcoded environment variables
- Consolidated: 5 configuration sources → 1
- Simplified: 200+ lines of macro code
- Improved: Test reproducibility across environments