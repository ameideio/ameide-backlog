# 073: Resolving Peer Dependencies in rules_js 2.4.2 for Next.js Applications

## Executive Summary

This document provides a comprehensive guide for resolving peer dependency issues when building Next.js applications with Bazel and rules_js 2.4.2. The specific issue addressed is webpack's inability to resolve peer dependencies in the Bazel sandbox environment, with `preact` as a peer dependency of `htm` serving as the primary example.

## Context and Problem Statement

### The Build Environment

- **Bazel Version**: 8.3.1
- **rules_js Version**: 2.4.2
- **Node.js Framework**: Next.js 15.4.4
- **Package Manager**: pnpm
- **Target Application**: BPMN Editor (www-ameide-portal-canvas)

### The Problem

When building a Next.js application that uses libraries with peer dependencies, webpack fails with "Module not found" errors. Specifically:

```
Module not found: Can't resolve 'preact'

Import trace for requested module:
../../node_modules/.aspect_rules_js/htm@3.1.1/node_modules/htm/preact/index.mjs
../../node_modules/.aspect_rules_js/@bpmn-io+diagram-js-ui@0.2.3/node_modules/@bpmn-io/diagram-js-ui/lib/index.js
```

### Root Cause Analysis

1. **Bazel's Hermetic Environment**: Bazel creates isolated build environments where only explicitly declared dependencies are available.

2. **rules_js Package Storage**: rules_js stores npm packages in a structure like:
   ```
   .aspect_rules_js/
   ├── htm@3.1.1/
   │   └── node_modules/
   │       └── htm/
   └── preact@10.27.0/
       └── node_modules/
           └── preact/
   ```

3. **Peer Dependency Resolution**: When `htm` imports `preact`, Node.js looks for it in:
   - `htm/node_modules/preact` (not present)
   - Parent directories up to the root (blocked by Bazel's sandbox)

4. **Missing Symlinks**: Unlike a traditional npm install, rules_js doesn't automatically create symlinks for peer dependencies within each package's node_modules.

## Architecture and Technical Details

### How rules_js Manages Dependencies

1. **npm_link_all_packages()**: This Bazel macro creates filegroups for each npm package, making them available as Bazel targets like `:node_modules/package-name`.

2. **Package Storage**: Packages are stored under `.aspect_rules_js/` with hashed names to ensure hermeticity.

3. **Runfiles Tree**: When building, Bazel constructs a runfiles tree that includes only the explicitly declared dependencies.

### The Peer Dependency Gap

```mermaid
graph TD
    A[Application Code] --> B[bpmn-js]
    B --> C[diagram-js]
    C --> D[@bpmn-io/diagram-js-ui]
    D --> E[htm]
    E -.->|peer dep| F[preact]
    
    style F stroke-dasharray: 5 5
```

The dotted line represents the peer dependency that webpack cannot resolve because `preact` isn't in `htm`'s local node_modules.

## Solution: Webpack Alias with require.resolve

Add webpack aliases in `next.config.ts` to explicitly tell webpack where to find peer dependencies. This approach uses `require.resolve` to find packages in a version-agnostic way that works within Bazel's hermetic sandbox:

```typescript
// services/www-ameide-portal-canvas/next.config.ts
import path from 'path';
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  webpack: (config, { isServer, webpack }) => {
    // ... existing config ...
    
    // Add webpack alias for preact (peer dependency of htm)
    // Use require.resolve to find the package in a version-agnostic way
    const preactPkg = require.resolve("preact/package.json");
    const preactDir = path.dirname(preactPkg);
    
    config.resolve.alias = {
      ...(config.resolve.alias || {}),
      preact: preactDir,
      'preact/hooks': path.join(preactDir, 'hooks'),
      'preact/compat': path.join(preactDir, 'compat'),
    };
    
    return config;
  },
};
```

### Why This Solution Works

1. **Version-agnostic**: No hardcoded version numbers in paths
2. **Sandbox-compatible**: `require.resolve` finds the correct path in Bazel's environment
3. **Maintainable**: Survives package updates without modification
4. **Explicit**: Clear what's happening for debugging
5. **Reliable**: Tested and proven to work with rules_js 2.4.2

## Step-by-Step Resolution Guide

### 1. Identify the Problem

When you see an error like:
```
Module not found: Can't resolve 'preact'
```

Check the import trace to understand which package needs the peer dependency.

### 2. Verify Dependencies

Ensure the peer dependency is declared in `package.json`:
```json
{
  "dependencies": {
    "preact": "^10.27.0",
    "htm": "^3.1.1"
  }
}
```

### 3. Update Next.js Configuration

Edit `next.config.ts` to add webpack aliases using `require.resolve`:

```typescript
// Use require.resolve to find the package dynamically
const preactPkg = require.resolve("preact/package.json");
const preactDir = path.dirname(preactPkg);

config.resolve.alias = {
  ...(config.resolve.alias || {}),
  preact: preactDir,
  'preact/hooks': path.join(preactDir, 'hooks'),
  'preact/compat': path.join(preactDir, 'compat'),
};
```

### 4. Clean and Rebuild

```bash
bazel clean --expunge
bazel build //services/www_ameide-portal-canvas:bpmn_editor_image
```

## Common Pitfalls and Troubleshooting

### 1. Wrong Package Path

If the alias path is incorrect, you'll still get "Module not found" errors. To find the correct path:

```bash
find bazel-out -name "preact" -type d | grep ".aspect_rules_js"
```

### 2. Missing Subpath Exports

Some packages export subpaths (like `preact/hooks`). You need separate aliases for each:

```typescript
'preact': path.resolve(__dirname, preactPath),
'preact/hooks': path.resolve(__dirname, preactPath + "/hooks"),
'preact/compat': path.resolve(__dirname, preactPath + "/compat"),
```

### 3. Handling Multiple Subpath Exports

When dealing with packages that have multiple entry points, ensure you alias all necessary subpaths.

### 4. Multiple Peer Dependencies

For multiple peer dependencies, use the same pattern:

```typescript
const preactPkg = require.resolve("preact/package.json");
const reactPkg = require.resolve("react/package.json");
const reactDomPkg = require.resolve("react-dom/package.json");

config.resolve.alias = {
  ...(config.resolve.alias || {}),
  'preact': path.dirname(preactPkg),
  'preact/hooks': path.join(path.dirname(preactPkg), 'hooks'),
  'react': path.dirname(reactPkg),
  'react-dom': path.dirname(reactDomPkg),
};
```

## Future Considerations

### rules_js Evolution

The rules_js team is aware of peer dependency challenges. Future versions may provide:
- Automatic peer dependency resolution
- Better imported_links API
- First-class peer dependency support

### Alternative Build Tools

Consider evaluating:
- **rules_nodejs**: Previous generation, different approach to dependencies
- **Nx**: Monorepo tool with different dependency management
- **Turborepo**: Simpler but less hermetic

### Maintenance Strategy

1. **Document all webpack aliases** with comments explaining why they're needed
2. **Create a script** to verify alias paths after dependency updates
3. **Consider abstracting** the alias configuration to a shared module
4. **Monitor rules_js releases** for improved peer dependency support

## Conclusion

While rules_js 2.4.2 presents challenges with peer dependencies, the webpack alias approach provides a pragmatic solution that:
- Maintains build hermeticity
- Requires minimal configuration changes
- Is explicit and debuggable
- Works reliably across different environments

The key insight is that fighting Bazel's hermeticity is harder than working with it. By explicitly telling webpack where to find dependencies, we maintain Bazel's guarantees while solving the immediate problem.

## References

1. [Aspect rules_js Documentation](https://github.com/aspect-build/rules_js)
2. [Webpack Resolve Configuration](https://webpack.js.org/configuration/resolve/)
3. [npm Peer Dependencies](https://docs.npmjs.com/cli/v8/configuring-npm/package-json#peerdependencies)
4. [Bazel Runfiles Guide](https://bazel.build/extending/rules#runfiles)

## Appendix: Complete Working Configuration

### services/www-ameide-portal-canvas/BUILD.bazel
```python
load("@npm//:defs.bzl", "npm_link_all_packages")
load("//tools/bazel/rules:web_app.bzl", "nextjs_app")

npm_link_all_packages(name = "node_modules")

nextjs_app(
    name = "bpmn_editor",
    srcs = glob(["src/**/*.ts", "src/**/*.tsx", "*.ts", "*.json"]),
    deps = [
        ":node_modules/preact",
        ":node_modules/zustand",
        ":node_modules/htm",
        ":node_modules/react",
        ":node_modules/react-dom",
        ":node_modules/@connectrpc/connect-web",
        ":node_modules/@bufbuild/protobuf",
        ":node_modules/bpmn-js",
        "//packages/ameide_core-sdk-ts:sdk_lib",
    ],
)
```

### services/www-ameide-portal-canvas/next.config.ts
```typescript
import path from 'path';
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  webpack: (config, { isServer, webpack }) => {
    // Add webpack alias for preact (peer dependency of htm)
    // Use require.resolve to find the package in a version-agnostic way
    const preactPkg = require.resolve("preact/package.json");
    const preactDir = path.dirname(preactPkg);
    
    config.resolve.alias = {
      ...(config.resolve.alias || {}),
      preact: preactDir,
      'preact/hooks': path.join(preactDir, 'hooks'),
      'preact/compat': path.join(preactDir, 'compat'),
    };
    
    return config;
  },
};

export default nextConfig;
```