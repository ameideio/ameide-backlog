# Connect v2 Migration for TypeScript Packages

**Status:** ✅ Completed  
**Date:** 2025-01-15  
**Packages:** `ameide_core_proto_web`, `ameide_sdk_ts`

## Overview

Successfully migrated TypeScript proto and SDK packages from Connect v1 to Connect v2, following the official migration guide. The migration removes the deprecated `protoc-gen-connect-es` plugin and adopts the new protobuf-es v2 API where service descriptors are now part of the `_pb.js` files.

## Challenge 1: Service Descriptor Location Change

**Problem:** In Connect v1, service descriptors lived in `*_connect.js` files. In v2, they're in `*_pb.js` files alongside message definitions.

**Solution:**
```yaml
# buf.gen.connect.yaml - BEFORE (v1)
plugins:
  - local: protoc-gen-es
    out: ../ameide_core_proto_web/src/generated
    opt: [target=ts]
  - local: protoc-gen-connect-es  # REMOVED in v2
    out: ../ameide_core_proto_web/src/generated
    opt: [target=ts]

# buf.gen.connect.yaml - AFTER (v2)
plugins:
  - local: protoc-gen-es
    out: ../ameide_core_proto_web/src/generated
    opt:
      - target=ts
      - import_extension=js  # Required for NodeNext
```

## Challenge 2: Message API Changes

**Problem:** Connect v2 uses protobuf-es v2's schema-based API instead of classes.

**Before (v1):**
```typescript
import { ArtifactView } from "@ameide/core-proto-web/artifacts/v1/types";
const artifact = new ArtifactView({
  id: "123",
  name: "Test"
});
```

**After (v2):**
```typescript
import { create } from "@bufbuild/protobuf";
import { ArtifactViewSchema } from "@ameide/core-proto-web/artifacts/v1/pb";
const artifact = create(ArtifactViewSchema, {
  id: "123",
  name: "Test"
});
```

## Challenge 3: TypeScript Resolution with Package Exports

**Problem:** TypeScript couldn't resolve re-exported types through package.json `exports` field when using star re-exports (`export *`) with NodeNext module resolution.

**Failed Approach:**
```typescript
// shims/artifacts/v1/pb.ts
export * from '../../../generated/ameide_core_proto/artifacts/v1/artifacts_pb.js';
```

**Working Solution:** Use named re-exports
```typescript
// shims/artifacts/v1/pb.ts
// Service descriptors
export {
  ArtifactCommandService,
  ArtifactQueryService,
} from '../../../generated/ameide_core_proto/artifacts/v1/artifacts_pb.js';

// Schemas (for create() function)
export {
  AggregateSchema,
  ArtifactViewSchema,
  // ... other schemas
} from '../../../generated/ameide_core_proto/artifacts/v1/artifacts_pb.js';

// Types
export type {
  Aggregate,
  ArtifactView,
  // ... other types
} from '../../../generated/ameide_core_proto/artifacts/v1/artifacts_pb.js';
```

## Challenge 4: tsup DTS Mode Failures

**Problem:** tsup's built-in `.d.ts` bundling (`dts: true`) failed with TypeScript errors when dealing with package.json exports + NodeNext + re-exports.

**Solution:** Separate JavaScript bundling from type generation:

```typescript
// tsup.config.ts
export default defineConfig({
  entry: ['src/index.ts', 'src/client.ts', /* ... */],
  format: ['esm', 'cjs'],
  dts: false,  // Let tsc handle declarations
  // ...
});
```

```json
// tsconfig.types.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "declaration": true,
    "emitDeclarationOnly": true,
    "outDir": "./dist",
    "noEmit": false
  }
}
```

```json
// package.json
{
  "scripts": {
    "build": "tsup && tsc -p tsconfig.types.json"
  }
}
```

## Challenge 5: Bazel ESM Detection

**Problem:** Bazel's ts_project rule couldn't detect ESM modules correctly, causing TS1479 errors.

**Attempted Solutions:**
1. Added `"type": "module"` to package.json ✅
2. Included package.json in Bazel data attribute ✅
3. Set `supports_workers = False` in ts_project ✅

**Status:** Known Bazel limitation. Packages work correctly with native tooling (npm/tsc/tsup).

## Migration Checklist

- [x] Remove `@connectrpc/protoc-gen-connect-es` dependency
- [x] Update buf.gen.yaml to use only `protoc-gen-es` v2
- [x] Add `import_extension=js` for NodeNext compatibility
- [x] Update all imports from `/connect` to `/pb`
- [x] Replace `new Message()` with `create(Schema, data)`
- [x] Update shims to use named re-exports
- [x] Configure tsup with `dts: false`
- [x] Add separate tsconfig for type generation
- [x] Clean up old `*_connect.*` files (42 files removed)
- [x] Update package.json exports for dual ESM/CJS

## Final Configuration

### Proto Package (`ameide_core_proto_web`)
```json
{
  "type": "module",
  "exports": {
    "./artifacts/v1/pb": {
      "types": "./dist/shims/artifacts/v1/pb.d.ts",
      "import": "./dist/shims/artifacts/v1/pb.js",
      "default": "./dist/shims/artifacts/v1/pb.js"
    }
  }
}
```

### SDK Package (`ameide_sdk_ts`)
```json
{
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "default": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsup && tsc -p tsconfig.types.json"
  }
}
```

## Verification

```bash
# Proto package builds
cd packages/ameide_core_proto_web && npm run build  # ✅

# SDK builds with dual format
cd packages/ameide_sdk_ts && npm run build     # ✅

# Runtime works
node -e "import('@ameide/core-proto-web/artifacts/v1/pb').then(m => 
  console.log('Services:', Object.keys(m).filter(k => k.includes('Service'))))"
# Output: Services: ['ArtifactCommandService', 'ArtifactQueryService', 'CrossReferenceService']

# CJS works
node -e "const sdk = require('@ameideio/ameide-sdk-ts'); 
  console.log('CJS:', Object.keys(sdk))"
# Output: CJS: ['AmeideClient', 'makeTransport', ...]
```

## Root Cause: The Four-Way Fight

The "spaghetti feeling" comes from juggling four opinionated systems that only *almost* agree:

1. **Node ESM rules** - needs real `.js` extensions, `"type": "module"`, package `exports`
2. **TypeScript's resolver** - NodeNext ≠ Node; struggles with `export *` through subpath exports
3. **Codegen (buf/protobuf-es/connect)** - v1→v2 moved services + changed messages to schema types
4. **Bazel** - sandboxes files; ts_project doesn't "see" `package.json` the way Node/TS expect

Each is sensible alone; together, they fight.

## The "No-Surprises" Recipe

### 1. One Source of Truth for Protos
- Use **protoc-gen-es v2** only (no `protoc-gen-connect-es`)
- `buf.gen.yaml`: `clean: true`, `include_imports: true`, `opt: target=ts, import_extension=js`

### 2. Keep Generated Code Out of Your Compiler's Hair
- Generate TS (`*_pb.ts`) then treat it like third-party code
- Don't hand-edit generated files. Put all stability behind **thin shims**

### 3. Make Shims Explicit (No Wildcard Re-exports)
```typescript
// ❌ BAD: TypeScript can't resolve this through subpath exports
export * from '../../../generated/ameide_core_proto/artifacts/v1/artifacts_pb.js';

// ✅ GOOD: Explicit named exports
export {
  ArtifactCommandService,
  ArtifactQueryService,
} from '../../../generated/ameide_core_proto/artifacts/v1/artifacts_pb.js';

export type {
  Aggregate,
  ArtifactView,
  // ... specific types
} from '../../../generated/ameide_core_proto/artifacts/v1/artifacts_pb.js';
```

### 4. Pick One Module Story and Stick to It
- `module: "NodeNext"`, `moduleResolution: "NodeNext"`, real `.js` extensions everywhere
- Add `"lib": ["ES2020","DOM","DOM.Iterable"]` if you touch `Headers.entries()` & friends

### 5. Split JS Bundling from Type Emission
- **tsup**: JS only (esm+cjs), `dts: false`
- **tsc**: declarations only via `tsconfig.types.json` (`emitDeclarationOnly: true`)

### 6. Subpath Exports Are Contracts
```json
// proto pkg package.json
{
  "exports": {
    "./artifacts/v1/pb": {
      "types": "./dist/shims/artifacts/v1/pb.d.ts",
      "import": "./dist/shims/artifacts/v1/pb.js"
    }
    // Don't export old *_connect paths (v2 puts services in *_pb)
  }
}
```

### 7. Bazel: Minimize Friction
- Avoid using `ts_project` to produce publishable JS; let tsup/tsc do that
- If you *do* type-check under Bazel, add `data = ["package.json"]` so NodeNext rules apply

### 8. Pin Versions & Add Smoke Tests
```json
// Pin in package.json
{
  "dependencies": {
    "@bufbuild/protobuf": "2.6.3",  // exact version
    "@connectrpc/connect": "2.0.4"   // exact version
  }
}
```

Add two tiny tests:
- `node` ESM import of `@pkg/artifacts/v1/pb` at runtime
- `tsc --noEmit` that imports the same and references a `...Schema`

## Mental Model

```
protos ──buf/protoc-gen-es v2──▶ *_pb.ts
                              ▲
                              │  (explicit named re-exports)
                         shims/pb.ts ──▶ published subpath export
                                              ▲
                                              │
                                           SDK imports
                            tsup(JS only) + tsc(d.ts only)
```

## Lessons Learned

1. **Named exports > star exports** for TypeScript with package.json exports
2. **Separate type generation from bundling** when using complex module setups
3. **Test both ESM and CJS** imports to ensure dual-format compatibility
4. **Clean old files** - leftover v1 files can cause confusing errors
5. **Follow vendor guides exactly** - the Connect team's migration guide was prescriptive for good reasons
6. **The intersection is the problem** - ESM + TS + codegen + Bazel each work fine alone
7. **Explicit shim exports + split builds** are the big stabilizers

## References

- [Connect v2 Migration Guide](https://connectrpc.com/docs/web/migrating)
- [Protobuf-ES v2 Manual](https://github.com/bufbuild/protobuf-es/blob/v2/MANUAL.md)
- [TypeScript NodeNext Resolution](https://www.typescriptlang.org/docs/handbook/modules/reference.html#node16-nodenext)

---

## Final Simplification (2025-08-17)

### Major Architecture Change: Removed `ameide_core_proto_web`

After successfully migrating to Connect-ES v2, we discovered an even simpler architecture:

**Before**: Three packages with complex dependencies
```
ameide_core_proto → ameide_core_proto_web → ameide_sdk_ts → platform
```

**After**: Two packages with clear boundaries
```
ameide_core_proto → ameide_sdk_ts → platform
```

### Key Changes

1. **Eliminated `ameide_core_proto_web` entirely** (169 files deleted)
   - Proto generation now happens directly in `ameide_core_proto`
   - No more separate web-specific proto package
   - Simpler build process with fewer moving parts

2. **SDK as the single boundary**
   - SDK re-exports ALL proto types: `export * from "@ameide/core-proto"`
   - Platform NEVER imports from proto directly
   - Clean architectural boundary enforced

3. **Simplified build process**
   - Proto: `pnpm run generate` (using buf)
   - SDK: `pnpm run build` (using tsc -b)
   - No more tsup, just TypeScript compilation

4. **Pure ESM with CJS compatibility**
   ```json
   {
     "type": "module",
     "exports": {
       ".": {
         "types": "./dist/index.d.ts",
         "import": "./dist/index.js",
         "default": "./dist/index.js"  // CJS fallback
       }
     }
   }
   ```

### Cleanup Performed

- ✅ Removed entire `packages/ameide_core_proto_web` directory
- ✅ Removed unused dependencies (`@connectrpc/connect-query`)
- ✅ Removed unused buf configs (grpc-web, go, python, local)
- ✅ Removed duplicate `interceptors.ts` file
- ✅ Cleaned up .tgz files from services
- ✅ Updated all documentation references
- ✅ Updated BUILD.bazel to remove proto-web references

### Working pnpm Workspace Configuration

The migration also standardized on pnpm workspaces:

```yaml
# pnpm-workspace.yaml
packages:
  - 'packages/*'
  - 'services/www_ameide_platform'

injectWorkspacePackages: true
```

With proper workspace protocol in dependencies:
```json
{
  "dependencies": {
    "@ameide/core-proto": "workspace:*",
    "@ameideio/ameide-sdk-ts": "workspace:*"
  }
}
```

### Final Architecture Benefits

1. **Simpler mental model**: Just proto → SDK → platform
2. **Faster builds**: One less package to build
3. **Clearer boundaries**: SDK is THE boundary for proto types
4. **Better IDE support**: Direct TypeScript compilation without bundlers
5. **Easier debugging**: Standard tsc output without transformation

### Verification

```bash
# Build everything in workspace
pnpm run build

# Proto builds successfully
cd packages/ameide_core_proto && pnpm run build  # ✅

# SDK builds with proto re-exports
cd packages/ameide_sdk_ts && pnpm run build  # ✅

# Platform uses SDK only
cd services/www_ameide_platform && pnpm run build  # ✅

# Docker build works
docker build -f services/www_ameide_platform/Dockerfile.release .  # ✅

# Kubernetes deployment via Tilt
tilt up  # ✅ Health checks pass
```

### Lessons from Simplification

1. **Question intermediate layers** - `ameide_core_proto_web` added no value
2. **SDK as boundary works** - Clean separation without complexity
3. **Standard tools win** - tsc over tsup reduced configuration
4. **Workspace protocols work** - pnpm's `workspace:*` solved resolution
5. **Less is more** - Removing a package made everything clearer

The Connect-ES v2 migration is now complete with a simpler, cleaner architecture than originally planned.