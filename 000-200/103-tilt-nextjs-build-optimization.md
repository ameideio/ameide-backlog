# Tilt NextJS Build Optimization

**Issue**: Every change to NextJS services triggers a full production build with optimization, causing significant delays (60-90s per change).

**Root Cause**: The current Tiltfile uses production Dockerfiles that run `npm run build`, which includes:
- Production optimizations (minification, tree-shaking)
- Static site generation
- Full bundling for production deployment

## Current Setup Problems

1. **Production Builds in Development**: 
   - Dockerfile runs `npm run build` on every change
   - Next.js production optimizer runs unnecessarily
   - No hot module replacement available

2. **No Live Updates**:
   - Live update is commented out in Tiltfile (lines 192-195)
   - Every change triggers full Docker rebuild
   - No file syncing for quick iterations

3. **Inefficient Layer Caching**:
   - Source files copied before build
   - Any source change invalidates build cache
   - Dependencies rebuilt unnecessarily

## Proposed Solutions

### Solution 1: Development Dockerfiles (Recommended - Fastest)

Create separate `Dockerfile.dev` for each NextJS service that uses development mode:

```dockerfile
# services/www-ameide/Dockerfile.dev
FROM node:18-alpine
WORKDIR /app

# Install dependencies (cached layer)
COPY package.json ./
RUN corepack enable pnpm && pnpm install --no-frozen-lockfile

# Copy source (changes frequently)
COPY . .

# Run development server with Turbopack
ENV NODE_ENV=development
ENV NEXT_TELEMETRY_DISABLED=1
EXPOSE 3000
CMD ["pnpm", "dev"]
```

**Benefits**:
- Instant hot reload with Turbopack
- No production build overhead
- Changes reflect immediately
- ~10x faster iteration

**Implementation**:
1. Create Dockerfile.dev for each service
2. Update Tiltfile `docker_build` to use Dockerfile.dev
3. Enable live_update for source syncing

### Solution 2: Enable Live Updates

Re-enable and optimize the commented live_update configuration:

```python
docker_build(
    repo,
    service_path,
    dockerfile=service_path + '/Dockerfile.dev',  # Use dev dockerfile
    live_update=[
        sync(service_path + '/src', '/app/src'),
        sync(service_path + '/public', '/app/public'),
        sync(service_path + '/app', '/app/app'),  # Next.js app directory
        run('cd /app && pnpm install --no-frozen-lockfile', trigger=[service_path + '/package.json']),
    ]
)
```

**Benefits**:
- Sub-second updates for code changes
- Only rebuilds on dependency changes
- Works with dev server hot reload

### Solution 3: Conditional Build Mode

Modify existing Dockerfiles to support both dev and production:

```dockerfile
ARG BUILD_MODE=production

# ... dependency stages ...

FROM node:18-alpine AS runner
WORKDIR /app

# Copy built app or source based on mode
ARG BUILD_MODE
RUN if [ "$BUILD_MODE" = "development" ]; then \
      echo "Development mode - skipping build"; \
    else \
      pnpm build; \
    fi

CMD [ "$BUILD_MODE" = "development" ] && pnpm dev || npm start
```

Then in Tiltfile:
```python
docker_build(
    repo,
    service_path,
    dockerfile=service_path + '/Dockerfile',
    build_args={'BUILD_MODE': 'development'}
)
```

### Solution 4: Optimize Production Dockerfile Caching

Improve the existing production Dockerfile for better caching:

```dockerfile
# Better layer caching
FROM node:18-alpine AS deps
WORKDIR /app
# Dependencies only - cached unless package files change
COPY package.json ./
RUN --mount=type=cache,target=/root/.pnpm-store \
    corepack enable pnpm && pnpm install --no-frozen-lockfile

FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
# Source files - separate layer
COPY . .
# Build with cache mount
RUN --mount=type=cache,target=/app/.next/cache \
    pnpm build
```

## Performance Comparison

| Approach | Initial Build | Code Change | Pros | Cons |
|----------|--------------|-------------|------|------|
| Current (Production) | 60-90s | 60-90s | Production-ready | Very slow iteration |
| Dev Dockerfile | 10-15s | <1s (hot reload) | Fastest iteration | Different from prod |
| Live Update | 60-90s | <1s | Minimal rebuilds | Complex setup |
| Conditional Build | 15-20s | 15-20s | Single Dockerfile | Still rebuilds |
| Optimized Caching | 60-90s | 20-30s | Production builds | Still slow |

## Recommended Implementation Plan

### Phase 1: Quick Win (1 hour)
1. Create Dockerfile.dev for all 4 NextJS services
2. Update Tiltfile to use dev dockerfiles
3. Test hot reload functionality

### Phase 2: Optimize (30 mins)
1. Enable live_update in Tiltfile
2. Configure proper sync patterns
3. Test file syncing

### Phase 3: Production Parity (Optional)
1. Add development vs production profiles to Tiltfile
2. Use production builds for staging/demo
3. Keep dev builds for local development

## Implementation Status

### ✅ Completed
- [x] Create services/www_ameide_platform/Dockerfile.dev (formerly www-ameide_canvas)
- [x] Update Tiltfile `deploy_nextjs_app` function to use Dockerfile.dev
- [x] Make solution generic - ANY service with Dockerfile.dev gets dev mode
- [x] Enable live_update with comprehensive directory syncing
- [x] Test hot reload with www-ameide-platform (<1s updates working!)

### Clean Architecture Achieved
The refactored solution is now:
- **Generic**: Any NextJS service with `Dockerfile.dev` automatically gets dev mode
- **No hardcoding**: Removed service-specific conditions
- **Consistent**: All dev-mode services get the same hot reload setup
- **Simple to extend**: Just add `Dockerfile.dev` to any service to enable

### To Enable Dev Mode for Other Services
Simply create a `Dockerfile.dev` in the service directory:
- [x] Create services/www-ameide/Dockerfile.dev
- ~~[ ] Create services/www-ameide_portal/Dockerfile.dev~~ (service deleted)
- ~~[ ] Create services/www-ameide_portal_canvas/Dockerfile.dev~~ (service deleted)

No Tiltfile changes needed - it will automatically detect and use dev mode!

## Expected Outcomes

**Before**:
- 60-90s per change
- Full production builds
- No hot reload
- Slow feedback loop

**After**:
- <1s for most changes
- Instant hot reload
- Fast feedback loop
- 10-100x faster iteration

## Notes

- Next.js 15 includes Turbopack which is significantly faster than Webpack
- The package.json already has `"dev": "next dev --turbopack"` configured
- Development mode skips optimizations that aren't needed locally
- Can still do production builds for final testing before deployment

## Related Files

- /workspace/Tiltfile (lines 176-232 for NextJS deployment)
- /workspace/services/www_*/Dockerfile (current production builds)
- /workspace/backlog/092-skaffold-nextjs-integration.md (similar optimization for Skaffold)