# Authentication System Review and Fixes

## Status: TODO

## Problem Statement

The www-ameide-platform service experiences authentication failures with NextAuth v5 + Keycloak integration:
- **Error**: "state value could not be parsed" during OAuth callback
- **Impact**: Users cannot authenticate, blocking platform access
- **Root Cause**: Unstable AUTH_SECRET causing state JWE decryption failures

## The Real Issue

The error "state value could not be parsed" means **the process handling `/api/auth/callback/*` can't decrypt the `state` JWE** it issued at `/api/auth/signin`. 

### Primary Cause: Unstable AUTH_SECRET
- If `AUTH_SECRET` isn't set, Auth.js generates a random secret at boot
- Any hot reload / pod restart / scale-out means callbacks may land on a process with a *different* secret
- Result: **state cannot be decrypted**

### Secondary Issue: Cookie Security with HTTPS + Dev Mode
- In dev mode, Auth.js uses non-`Secure` cookies
- Accessing via HTTPS (`https://platform.dev.ameide.io`) expects secure cookies
- Cookies may not persist through the OAuth flow

## The Fix (Keeping NODE_ENV=development)

### 1. Make AUTH_SECRET Stable Across All Instances

**First, verify current state:**
```bash
kubectl exec -n ameide deploy/www-ameide-platform -- sh -lc 'printf "AUTH_SECRET=%s\n" "$AUTH_SECRET"'
```

**If empty or inconsistent, create and inject it:**
```bash
# Generate a 32-byte secret
SECRET=$(openssl rand -base64 32 | tr -d '\n')
kubectl -n ameide create secret generic www-ameide-auth --from-literal=AUTH_SECRET="$SECRET"

# Wire it into the deployment
kubectl -n ameide patch deploy/www-ameide-platform --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{
     "name":"AUTH_SECRET",
     "valueFrom":{"secretKeyRef":{"name":"www-ameide-auth","key":"AUTH_SECRET"}}
  }}
]'

# Restart to pick up the secret
kubectl -n ameide rollout restart deploy/www-ameide-platform
```

> With a stable `AUTH_SECRET`, you don't need sticky sessions. Any pod can decrypt the state and session.

### 2. Set Proper Environment Variables

Ensure these are set (via Helm values or patch):
```bash
AUTH_URL=https://platform.dev.ameide.io
AUTH_TRUST_HOST=true
AUTH_DEBUG=true   # temporarily, for debugging
```

**Why:**
- `AUTH_URL` lets Auth.js build correct callback/redirect URLs behind TLS termination
- `AUTH_TRUST_HOST=true` tells Auth.js to honor `X-Forwarded-*` headers from the gateway

### 3. Force Secure Cookies on HTTPS (Keep Development Mode)

**File: `app/(auth)/auth.shared.ts`**

```typescript
// Detect HTTPS even in development
const isHttps =
  (process.env.AUTH_URL ?? '').startsWith('https://') ||
  process.env.COOKIE_SECURE === 'true';

export const sharedAuthConfig = {
  // ... existing config ...
  trustHost: true,
  secret: process.env.AUTH_SECRET, // MUST be set now

  // Keep dev cookie names, but make them secure on HTTPS
  cookies: {
    sessionToken: {
      name: process.env.NODE_ENV === 'production'
        ? '__Secure-authjs.session-token'
        : 'authjs.session-token',
      options: {
        httpOnly: true,
        sameSite: 'lax',
        path: '/',
        secure: isHttps,  // <-- secure even in dev when using HTTPS
      },
    },
    // Harden the OAuth flow cookies too:
    state: {
      name: process.env.NODE_ENV === 'production'
        ? '__Secure-authjs.state'
        : 'authjs.state',
      options: { 
        httpOnly: true, 
        sameSite: 'lax', 
        path: '/', 
        secure: isHttps 
      },
    },
    nonce: {
      name: process.env.NODE_ENV === 'production'
        ? '__Secure-authjs.nonce'
        : 'authjs.nonce',
      options: { 
        httpOnly: true, 
        sameSite: 'lax', 
        path: '/', 
        secure: isHttps 
      },
    },
    pkceCodeVerifier: {
      name: process.env.NODE_ENV === 'production'
        ? '__Secure-authjs.pkce.code_verifier'
        : 'authjs.pkce.code_verifier',
      options: { 
        httpOnly: true, 
        sameSite: 'lax', 
        path: '/', 
        secure: isHttps 
      },
    },
  },
} satisfies AuthConfig;
```

> Don't set a `domain` unless you need cross-subdomain sharing. Host-only cookies are correct here.

### 4. Ensure Edge and Node Use Same Secret

**File: `app/(auth)/auth.edge.ts`**

```typescript
import NextAuth from "next-auth";
import { sharedAuthConfig } from "./auth.shared";

// No need to redefine providers/callbacks here
// Edge runtime only needs to read/verify the session
export const { auth } = NextAuth(sharedAuthConfig);
```

### 5. Gateway Headers Verification

Check that the gateway forwards headers correctly:
```bash
curl -k -sI https://platform.dev.ameide.io/ | sed -n '1,20p'
```

Should see:
- `x-forwarded-proto: https`
- `x-forwarded-host: platform.dev.ameide.io`

## Testing the Fix

### Proper OAuth Flow Test (Use POST, not GET!)

```bash
jar=/tmp/ameide.cookies
rm -f "$jar"

# Start the sign-in (POST initiates OAuth and sets authjs.state cookie)
curl -k -s -D - -c "$jar" -b "$jar" \
  -X POST "https://platform.dev.ameide.io/api/auth/signin/keycloak?callbackUrl=%2F" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data '' | sed -n '1,40p'
```

**Expected in response:**
- `HTTP/2 302`
- `Location: https://auth.dev.ameide.io/realms/ameide/...`
- `Set-Cookie: authjs.state=...; Path=/; SameSite=Lax; Secure; HttpOnly`

With stable `AUTH_SECRET` and secure cookies, the callback should work without "state value could not be parsed" errors.

## Optional Hardening

1. **Keycloak Client Configuration:**
   - Valid Redirect URIs: `https://platform.dev.ameide.io/api/auth/callback/*`
   - Web Origins: `https://platform.dev.ameide.io`

2. **Helm Values Update:**
   Add to the Helm chart values to ensure AUTH_SECRET is always set:
   ```yaml
   auth:
     secret: "{{ .Values.auth.secret | required "auth.secret is required" }}"
   ```

3. **Error Page Fix:**
   Ensure `/api/auth/error` route is properly exported and not wrapped in a throwing layout.

## Key Insights

1. **The root cause is AUTH_SECRET instability**, not the NODE_ENV setting
2. **OAuth requires POST** to `/api/auth/signin/keycloak` to properly initiate the flow
3. **Cookies can be secure in dev mode** by detecting HTTPS from AUTH_URL
4. **Don't need to switch to production mode** - just fix the secret and cookie security

## Success Criteria

- [x] AUTH_SECRET is stable across all pod instances
- [x] No "state value could not be parsed" errors in logs
- [x] OAuth flow completes with POST to signin endpoint
- [x] Sessions persist across pod restarts
- [x] Cookies are secure when using HTTPS

## Implementation Checklist

- [x] Create Kubernetes Secret for AUTH_SECRET (using Helm values)
- [x] Update deployment to use secretKeyRef
- [x] Modify auth.shared.ts to force secure cookies on HTTPS
- [x] Ensure edge config uses shared config
- [x] Test with POST to signin endpoint
- [x] Remove conflicting wellKnown/internalBase configuration
- [x] Fix Redis to point to master node
- [x] Configure Keycloak hostname settings

## References

- [NextAuth v5 OAuth Documentation](https://authjs.dev/reference/core#oauth)
- [NextAuth Cookie Configuration](https://authjs.dev/reference/core#cookies)
- [JWE State Token Explanation](https://authjs.dev/concepts/security)

## Notes

- The earlier hypothesis about "flip to production" was rejected because it's not necessary
- The real fix is ensuring AUTH_SECRET stability and proper cookie security
- GET requests to signin endpoints don't initiate OAuth (explains why cURL tests didn't set cookies)
- This approach maintains development features while fixing the authentication issues