# 074: Yarn Migration Analysis - Would It Solve the Peer Dependency Madness?

## Executive Summary

**Short answer**: Probably not. The peer dependency resolution issues are primarily caused by how rules_js structures packages in Bazel's hermetic environment, not by the package manager itself. However, Yarn's Plug'n'Play (PnP) mode offers some interesting possibilities that could either help or make things worse.

## Current State Analysis

### What We Have Now
- **Package Manager**: pnpm 10.13.1
- **Bazel Integration**: rules_js 2.4.2
- **Problem**: Peer dependencies can't be resolved in Bazel's hermetic sandbox

### The Real Culprit

The issue isn't pnpm. It's the interaction between:
1. Bazel's hermetic build environment
2. rules_js's package layout strategy
3. Node.js module resolution algorithm
4. Webpack's expectations

## Yarn Migration Analysis

### Yarn Classic (v1) with node_modules

**Would it help?** ❌ No

Yarn Classic creates the same `node_modules` structure as npm/pnpm. rules_js would still:
- Store packages in `.aspect_rules_js/`
- Create the same isolated package structure
- Result in the same peer dependency resolution failures

**Migration effort**: Low  
**Benefit**: None for this issue

### Yarn Berry (v2+) with node_modules

**Would it help?** ❌ No

Same as Yarn Classic when using node_modules linker. The package layout would be identical.

**Migration effort**: Medium (need to handle Yarn 2+ config)  
**Benefit**: None for this issue

### Yarn Berry with Plug'n'Play (PnP)

**Would it help?** 🤔 Maybe, but probably makes it worse

Yarn PnP completely changes how Node.js resolves modules:

```javascript
// Instead of node_modules folders, Yarn PnP uses:
// - .pnp.cjs (resolver logic)
// - .yarn/cache/ (zip archives)
// - Virtual filesystem for packages
```

**Potential Benefits:**
1. **Explicit peer dependencies**: PnP forces explicit declaration of all dependencies
2. **No node_modules**: Could simplify Bazel integration
3. **Deterministic resolution**: More aligned with Bazel's philosophy

**Likely Problems:**
1. **rules_js compatibility**: rules_js expects node_modules structure
2. **Tool compatibility**: Many tools don't support PnP
3. **Webpack configuration**: Requires PnP webpack plugin
4. **Next.js support**: Limited and experimental
5. **Debugging nightmare**: Virtual filesystem adds complexity

**Migration effort**: Very High  
**Risk**: Extreme

## Alternative Package Managers Analysis

### npm
**Would it help?** ❌ No  
Same fundamental issue. npm creates node_modules just like pnpm.

### Bun
**Would it help?** ❌ No  
Bun is a runtime + package manager. Package management still creates node_modules.

### Deno
**Would it help?** ✅ Yes, but...  
Deno uses URLs for imports, no node_modules. But requires rewriting entire codebase.

## The Real Solutions Comparison

| Solution | Complexity | Reliability | Maintenance | Future-proof |
|----------|------------|-------------|-------------|--------------|
| **Current (webpack alias)** | Low | High | Medium | Good |
| **Yarn Classic** | Low | High | Same | Good |
| **Yarn PnP** | Very High | Unknown | High | Unknown |
| **Wait for rules_js fix** | None | N/A | None | Best |
| **Different build tool** | Very High | Unknown | Unknown | Unknown |

## What Yarn PnP Integration Would Look Like

If we were to attempt Yarn PnP with rules_js:

### 1. Yarn Configuration
```yaml
# .yarnrc.yml
nodeLinker: "pnp"
pnpMode: "strict"

# Would need custom rules_js integration
pnpIgnorePatterns:
  - "bazel-*"
```

### 2. Bazel Changes Required
```python
# BUILD.bazel - Hypothetical PnP support
load("@yarn//:pnp.bzl", "yarn_pnp_install")  # Doesn't exist

yarn_pnp_install(
    name = "node_modules",
    pnp_file = ".pnp.cjs",
    cache_folder = ".yarn/cache",
)
```

### 3. Webpack Configuration
```javascript
// webpack.config.js
const PnpWebpackPlugin = require('pnp-webpack-plugin');

module.exports = {
  resolve: {
    plugins: [PnpWebpackPlugin],
  },
  resolveLoader: {
    plugins: [PnpWebpackPlugin.moduleLoader(module)],
  },
};
```

### 4. Next.js Issues
- Next.js has experimental PnP support
- Many Next.js plugins don't work with PnP
- Requires constant workarounds

## Real-World Migration Paths

### Option 1: Stay with pnpm + webpack aliases ✅
**Pros:**
- Already working
- Well understood
- Minimal configuration
- Easy to maintain

**Cons:**
- Manual alias management
- Version-specific paths

### Option 2: Implement custom Bazel rules
Create custom rules that understand peer dependencies:

```python
# Hypothetical custom rule
peer_dependency_link(
    name = "preact_for_htm",
    package = "preact",
    for_package = "htm",
)
```

**Pros:**
- Cleaner than webpack aliases
- More Bazel-native

**Cons:**
- Significant development effort
- Maintenance burden

### Option 3: Fork and patch rules_js
Add peer dependency support directly to rules_js:

```python
npm_link_all_packages(
    name = "node_modules",
    auto_link_peer_deps = True,  # Hypothetical feature
)
```

**Pros:**
- Solves problem at the source
- Benefits entire community

**Cons:**
- Massive effort
- Need to maintain fork

## Recommendation

**Don't migrate to Yarn for this issue.** The problem isn't the package manager—it's the impedance mismatch between Node.js's module resolution and Bazel's hermetic builds.

### Stick with the current solution because:

1. **It works**: Webpack aliases solve the problem reliably
2. **It's maintainable**: One configuration file, easy to update
3. **It's explicit**: Clear what's happening, easy to debug
4. **Low risk**: No breaking changes to the build system

### When to reconsider:

1. **rules_js 3.x**: If Aspect announces peer dependency support
2. **Multiple projects**: If you have many services with the same issue
3. **Different build tool**: If moving away from Bazel entirely

## Conclusion

The "madness" isn't really about pnpm vs Yarn—it's about the fundamental tension between:
- Node.js's flexible module resolution
- Bazel's hermetic build philosophy
- rules_js's current implementation

Yarn won't solve this. Yarn PnP might even make it worse by adding another layer of complexity that rules_js doesn't understand.

The webpack alias solution, while not elegant, is actually the most pragmatic approach. It's explicit, maintainable, and works reliably. Sometimes the best solution is the simple one that already works.

## Alternative: The Nuclear Option

If you really want to escape this complexity, consider:

1. **Drop Bazel for frontend**: Use Nx, Turborepo, or even just npm workspaces
2. **Keep Bazel for backend**: Where it shines with hermetic builds
3. **Clear boundaries**: Frontend and backend as separate build systems

But this is a much larger architectural decision that goes beyond just peer dependencies.