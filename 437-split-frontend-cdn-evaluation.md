Here’s a vendor-aligned rewrite of your document.

---

# Split Frontend to CDN – Azure Best Practices Aligned

## Summary

Evaluate how to align `www-ameide` (marketing) and `www-ameide-platform` (app) with **Azure-recommended architectures for internet-facing web applications**:

* Use **Azure Front Door** as the **global edge entry point with CDN + WAF**. ([Microsoft Learn][1])
* Serve **static marketing content from a static hosting service + CDN** (Azure Static Web Apps or Storage+CDN). ([Microsoft Learn][2])
* Keep **dynamic, stateful workloads** (platform app, APIs, Keycloak) on **AKS** behind Front Door.

### Final Recommendation (vendor-aligned)

1. **Adopt Azure Front Door in front of AKS for both apps (Option B) as the new default.**
2. **Plan a refactor to move `www-ameide` to static hosting (Option A) once capacity allows.**
3. Treat “all traffic directly to AKS ingress” (current Option C) as a **transitional / legacy state**, not the target architecture.

---

## Current State

Both frontends run as Next.js containers in AKS:

* **Auth**: Auth.js + Keycloak OIDC
* **Sessions**: Redis (Sentinel HA for platform)
* **Ingress**: Envoy Gateway, public internet-facing
* **Delivery**: No global edge layer; static assets and HTML all served via AKS nodes
* **Deployment**: GitOps via ArgoCD

### App Comparison

| Feature      | `www-ameide` (Marketing) | `www-ameide-platform` (App) |
| ------------ | ------------------------ | --------------------------- |
| Auth         | Keycloak + Auth.js       | Keycloak + Auth.js          |
| Sessions     | Redis                    | Redis Sentinel (HA)         |
| Database     | None                     | PostgreSQL                  |
| WebSocket    | Client connects to API   | Client connects to API      |
| gRPC calls   | Envoy proxy              | Envoy proxy (internal)      |
| AI APIs      | None                     | OpenAI, Anthropic, Azure    |
| Telemetry    | None                     | OpenTelemetry               |
| File uploads | None                     | Persistent volume           |

**Deviation vs vendor best practices**

* No **central global edge entry point** (Front Door) for performance and security. ([Microsoft Learn][3])
* Static content is not served from a dedicated **static hosting + CDN** solution. ([Microsoft Learn][4])
* No **WAF on the public edge**; protection is only at the cluster/application level. ([Microsoft Learn][5])

---

## Target Azure Architecture Principles

Target state is guided by Azure’s recommended patterns for public web apps: ([Microsoft Learn][1])

1. **Single global edge entry point**

   * Azure Front Door in front of all public HTTP(S) endpoints (`www-ameide`, `www-ameide-platform`, `auth`, `api`).
   * Origins (AKS, Static Web Apps, Storage) **accept traffic only from Front Door** (Private Link or `X-Azure-FDID`/service tags). ([Microsoft Learn][6])

2. **Static content offload**

   * Marketing/frontdoor pages hosted on **Static Web Apps or Storage+CDN**.
   * AKS used only for dynamic workloads and APIs.

3. **Security on the edge**

   * End-to-end TLS, HTTP→HTTPS redirect, managed certificates on Front Door. ([Microsoft Learn][6])
   * Azure WAF on Front Door with managed rule sets, tuned for the apps. ([Microsoft Learn][5])

4. **Operational excellence**

   * Infrastructure as code (Bicep/Terraform/ARM) for Front Door, WAF, DNS.
   * Centralized logging/metrics from Front Door + AKS.

---

## Options

### Option A: Static Marketing Site + CDN (Target for `www-ameide`)

Convert the **marketing site** to a static or SSG build and host it on **Azure Static Web Apps** (or Azure Storage + CDN / Front Door origin).

| Aspect   | Details                                          |
| -------- | ------------------------------------------------ |
| Scope    | `www-ameide` (marketing) only                    |
| Hosting  | Azure Static Web Apps (preferred) or Storage+CDN |
| Auth     | Client-side OIDC against Keycloak                |
| Sessions | Token-based; remove Redis dependency             |
| Cost     | $0 (Free tier) or ~$9/mo (Standard) for SWA      |
| Effort   | 2–4 weeks refactor                               |

**Changes required**

* Configure Next.js for static output (`output: "export"` or equivalent; remove SSR-only pages).
* Move from Auth.js server sessions to **client-side OIDC (PKCE) with tokens** in memory + secure cookies.
* Replace any server-only logic in pages with **client-side API calls** to the platform/backend.

**Pros (vendor best practice)**

* **Static Content Hosting pattern**: static files from a global edge cache, minimal compute. ([Microsoft Learn][4])
* **Independent deployment & scaling** for marketing vs platform.
* Offloads marketing traffic from AKS, reducing node pressure and simplifying capacity planning.
* Aligned with Azure’s **“SPA + API + IdP”** guidance.

**Cons / Risks**

* Significant refactor effort, especially around Auth.js and any SSR usage.
* Loss of SSR on marketing pages (SEO must rely on SSG/build-time rendering).
* Introduces a second deployment model alongside GitOps (unless you wire SWA into GitOps via ARM/Bicep/Terraform).

**Recommended role:**
**Strategic target state for the marketing site** once Option B is in place and there is time/budget for refactor.

---

### Option B: Azure Front Door + AKS (New Default)

Introduce **Azure Front Door Standard or Premium** as the **global entry point** for all public endpoints, with AKS as origin.

| Aspect   | Details                                               |
| -------- | ----------------------------------------------------- |
| Scope    | `www-ameide`, `www-ameide-platform`, `auth`, `api`    |
| Hosting  | AKS (unchanged), Front Door as edge                   |
| Security | TLS termination, optional WAF (Premium)               |
| Cost     | ~$35/mo (Standard) or ~$330/mo (Premium with WAF)     |
| Effort   | 1–2 days for initial setup; additional for WAF tuning |

**Changes required**

* Create **Azure Front Door** profile:

  * Origins: Envoy Gateway endpoints for both apps, plus API and Keycloak.
  * Custom domains: `www.ameide.io`, `platform.ameide.io`, `auth.ameide.io`, `api.ameide.io`.
* Configure route rules:

  * Static asset caching: `/_next/static/*`, `/images/*`, `/fonts/*`.
  * Path-based routing if needed (`/api/*`, `/auth/*`, etc.).
* Enable **HTTPS-only** and use managed certificates for custom domains.
* Optionally enable **WAF** (Premium), start in detection mode, then enforce. ([Microsoft Learn][1])
* Lock down AKS ingress to only accept traffic from Front Door (Private Link / `X-Azure-FDID` / service tags). ([Microsoft Learn][6])

**Pros (vendor best practice)**

* Directly matches Azure’s **recommended front-door-in-front-of-web-apps pattern**: global edge entry, CDN, WAF, managed TLS. ([Microsoft Learn][1])
* **No application changes**:

  * Auth.js, Redis, Keycloak, websockets, file uploads keep working.
* **Global CDN caching** for static assets reduces latency and AKS load.
* Single place for:

  * TLS termination.
  * WAF rules and bot protection.
  * Logging and request analytics.

**Cons**

* Additional monthly cost for Front Door + potential WAF.
* Infra complexity: one more component to manage as code.
* Need to integrate cache invalidation (purge) into CI/CD for static assets.

**Recommended role:**
**New baseline / default architecture** for production traffic.
Even if most users are EU-based today, this aligns with Azure guidance and sets a clean path for global expansion and improved security.

---

### Option C: Direct AKS Ingress (Legacy / Transitional)

Keep both apps in AKS, exposed directly through Envoy Gateway to the internet (current state).

| Aspect   | Details                                  |
| -------- | ---------------------------------------- |
| Hosting  | AKS only, Envoy Gateway with public IP   |
| Security | TLS at Envoy; no WAF at edge             |
| Cost     | $0 incremental (no extra Azure services) |
| Effort   | None                                     |

**Pros**

* Simplest architecture from an operational point of view.
* GitOps-only model (ArgoCD) for all changes.
* Shared infrastructure (Redis, Keycloak, Envoy) already paid for.

**Cons vs vendor best practices**

* **No global edge layer** (no Front Door), contrary to recommended designs for internet-facing apps. ([Microsoft Learn][3])
* No built-in **WAF on the public edge**; harder to meet future compliance/security requirements. ([Microsoft Learn][5])
* All traffic hits AKS directly; no caching or offload of static content.

**Recommended role:**
Acceptable as a **transitional state** while traffic and security requirements are modest. Not the long-term target architecture.

---

## Decision Matrix (Vendor-Aligned)

| Scenario                                 | Recommendation (Best Practice)                                                              |
| ---------------------------------------- | ------------------------------------------------------------------------------------------- |
| Most users in EU, but production-facing  | **Option B**: Front Door Standard in front of AKS for both apps                             |
| Global users, latency matters            | **Option B**: Front Door with aggressive static caching; later add Option A for marketing   |
| Need WAF / stronger security posture     | **Option B Premium**: Front Door Premium + WAF with managed rule sets                       |
| Marketing site independence / agility    | **Option A for `www-ameide`** (Static Web App or Storage+CDN) + **Option B** overall        |
| Tight budget, low risk, local usage only | Temporary **Option C**, but plan migration to **Option B** when traffic/security justify it |

---

## Recommendation

### 1. Adopt Front Door as the New Baseline

Move from “direct AKS ingress” to **Azure Front Door + AKS**:

* Implement **Option B** now:

  * Front Door Standard (or Premium if WAF needed).
  * Origins: `www-ameide`, `www-ameide-platform`, `auth`, `api`.
  * HTTPS-only, managed certificates, basic caching for static assets.
  * Lock AKS ingress to traffic from Front Door only.

This aligns the entire stack with Azure’s **recommended front-door pattern** for internet-facing workloads and prepares you for WAF and global expansion. ([Microsoft Learn][1])

### 2. Plan Static Hosting for Marketing

Treat **Option A** as the strategic evolution for `www-ameide`:

* Schedule a **2–4 week refactor** to:

  * Make `www-ameide` compatible with static/SSG output.
  * Switch auth to client-side OIDC tokens.
* Host on **Azure Static Web Apps** (or Storage+CDN) behind the same Front Door.
* Keep `www-ameide-platform` on AKS (dynamic app, websockets, uploads, AI, etc.).

### 3. Treat “All in AKS” as Transitional

Keep **Option C** as:

* A **short-term** cost-optimized state while Front Door is rolled out.
* Not the desired end-state in any scenario where:

  * Security, compliance, or WAF is required.
  * Latency for non-local users matters.
  * Marketing scaling or independence becomes important.

---

## Related / Clean-Up Actions

* **Existing Static Web App `ameide`**

  * Either repurpose it for the future `www-ameide` static site, or
  * Decommission it to avoid confusion until the refactor is planned.

* **DNS & Domains**

  * Align DNS so that **all public hostnames** resolve to Azure Front Door, not directly to Envoy/public IPs.
  * Use Front Door’s managed certificates for `ameide.io` and `*.ameide.io`.

* **Security Hardening**

  * Lock down AKS ingress to allow only Front Door (service tags / `X-Azure-FDID`) as the source. ([Microsoft Learn][6])
  * If using Front Door Premium, enable WAF in detection mode, tune rules, then switch to prevention.

This version puts the **Azure reference architecture** front and center and frames your current AKS-only approach as a stepping stone rather than the final destination.

[1]: https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-front-door?utm_source=chatgpt.com "Architecture Best Practices for Azure Front Door"
[2]: https://learn.microsoft.com/en-us/azure/static-web-apps/overview?utm_source=chatgpt.com "What is Azure Static Web Apps?"
[3]: https://learn.microsoft.com/en-us/azure/architecture/web-apps/?utm_source=chatgpt.com "Web applications architecture design - Azure"
[4]: https://learn.microsoft.com/en-us/azure/architecture/patterns/static-content-hosting?utm_source=chatgpt.com "Static Content Hosting pattern - Azure Architecture Center"
[5]: https://learn.microsoft.com/en-us/azure/web-application-firewall/afds/waf-front-door-best-practices?utm_source=chatgpt.com "Best practices for Azure Web Application Firewall in ..."
[6]: https://learn.microsoft.com/en-us/azure/frontdoor/best-practices?utm_source=chatgpt.com "Best Practices - Azure Front Door"




From the point of view of CNCF / Linux Foundation–style thinking, the advice is basically:

> **Use open, vendor-neutral building blocks (CNCF projects), keep cloud-specific stuff thin and replaceable, and keep your data + control plane portable.** ([CNCF][1])

Let’s translate that into your world (Next.js + AKS + Keycloak + Envoy).

---

## 1. What “open source / Linux Foundation” people actually push

Across CNCF and Linux Foundation docs/blogs, a few themes repeat:

1. **Vendor-neutral core platform**

   * CNCF explicitly positions itself as a *vendor-neutral hub* to avoid any one vendor controlling your fate. ([CNCF][1])
   * Their guidance is: build on **open, portable APIs and runtimes** like:

     * Kubernetes, Envoy, Prometheus, OpenTelemetry, Argo, etc. ([CNCF][2])

2. **Avoid deep coupling to proprietary services**

   * Linux Foundation material and related analysis repeatedly mention that open source is adopted *specifically to avoid vendor lock-in* and unify technology investments. ([The Linux Foundation][3])
   * “Cloud-native” from a CNCF perspective means *portable*, not “use every shiny managed PaaS your cloud offers”. ([CNCF][4])

3. **Governance under neutral foundations**

   * When projects move under the Linux Foundation or CNCF, one stated benefit is reduced lock-in and more predictable licensing / governance (Redis → Valkey under LF is a recent example). ([altinity.com][5])

So an open-source community answer to “how should I architect this?” is roughly:

> **Put CNCF / LF projects at the center (Kubernetes, Envoy, OpenTelemetry, Argo, Keycloak, Postgres, etc.), and treat any particular cloud’s CDN or WAF as an optional edge plug-in, not as your platform.**

You’re actually *already* very aligned there.

---

## 2. Where your current setup already matches OSS/CNCF guidance

Your stack is basically a CNCF brochure:

* **Kubernetes** (AKS is still just Kubernetes – API-portable to any distro). ([CNCF][6])
* **Envoy** as ingress (CNCF graduated project, high-performance edge/service proxy). ([CNCF][2])
* **ArgoCD** (CNCF project) for GitOps. ([CNCF][6])
* **OpenTelemetry** for telemetry (CNCF). ([CNCF][6])
* **Keycloak, Redis, Postgres** – all OSS, independent of any single cloud.

From a CNCF/LF perspective, the *lock-in risk* is not AKS itself (you can move to any other Kubernetes), it’s how much you lean on **Azure-specific PaaS (Front Door, Static Web Apps, etc.)** vs OSS components you control.

---

## 3. What would OSS / Linux Foundation folks recommend for your “CDN split” topic?

### Principle 1 – Keep the “platform” OSS & portable

They’d say: your **platform** should be:

* Kubernetes + Envoy/Envoy Gateway (for ingress/API gateway) ([CNCF][2])
* Keycloak for auth (OSS, portable).
* Redis/Postgres/OpenTelemetry/ArgoCD.

Then you can run that platform:

* On AKS today.
* On EKS/GKE/on-prem/bare metal tomorrow, with no app changes other than cluster wiring.

### Principle 2 – Use OSS for HTTP edge & caching where feasible

Instead of Azure Front Door as your only option, OSS people would suggest:

* **Envoy Gateway** or other CNCF-ish ingress as your “edge gateway”:

  * Implements the Kubernetes Gateway API, is fully OSS, and designed for unified ingress + API gateway use. ([CNCF][7])
* Add an **OSS caching layer** in front of your apps for static content:

  * Examples: **Varnish Cache**, **Apache Traffic Server**, or even tuned **Nginx** as reverse proxies / HTTP accelerators – all widely used for building CDNs. ([Varnish Cache][8])

You can deploy these:

* As part of your Kubernetes cluster(s), or
* In a small dedicated edge cluster / PoP close to your users.

That gives you:

* The same “cache `/_next/static/*` and images at the edge” behavior you’d use Front Door for.
* But all implemented with OSS you can run on *any* cloud or on-prem.

### Principle 3 – Static hosting should be generic, not tied to one cloud

For the “static export marketing site” angle, the OSS-ish recommendation is:

* Do the **static export** with Next.js if it makes sense.
* Host the output on something **generic & swappable**:

  * S3-compatible object storage (e.g. MinIO, Ceph RGW) or plain blob storage.
  * Fronted by your OSS edge (Envoy/Varnish/Nginx) or any commodity CDN.
* Use only **standard HTTP, TLS, DNS** features – no proprietary vendor routing rules or serverless edge functions.

That means:

* Today: maybe it’s Azure Blob + OSS gateway / generic CDN.
* Tomorrow: it could be S3, GCS, or your own MinIO cluster, without touching the app.

This is exactly the sort of thing Linux Foundation and CNCF call out as a way to avoid lock-in: open protocols, open software, portable data formats. ([CNCF][4])

---

## 4. Concretely: what does that mean for your options A / B / C?

Taking your previous options and repainting them in “Linux Foundation / CNCF” colours:

### OSS-aligned Option 1 – Stay on Kubernetes + add OSS edge caching

Instead of Azure Front Door:

* Keep **both Next.js apps on Kubernetes** (AKS now, portable later).
* Use **Envoy Gateway** as your unified ingress/API gateway (CNCF/Envoy project). ([CNCF][7])
* Enable **caching for static paths** (`/_next/static/*`, `/images/*`, `/fonts/*`) in Envoy or via a dedicated Varnish/ATS layer.
* Keep GitOps (ArgoCD) for the whole thing.

This is almost your current Option C but with:

* Better performance via caching and tuning.
* No new proprietary components.
* Fully portable between clouds.

### OSS-aligned Option 2 – Static marketing, but cloud-agnostic

Instead of “Azure Static Web Apps”:

* Do static export for `www-ameide` if/when it makes sense.
* Host static files on **generic object storage** + an OSS gateway:

  * e.g. MinIO / Ceph behind Envoy / Nginx / Varnish.
* Put that behind the **same Envoy Gateway / OSS edge** you use for the app.

Now “marketing site independence” is achieved with:

* Next.js + git + Kubernetes + OSS edge,
* Not Azure-specific build + routing semantics.

### OSS-aligned stance on Azure Front Door / Static Web Apps

Open-source communities wouldn’t say “never use a cloud CDN”; they’d say:

* If you use something like Front Door or a commercial CDN:

  * Treat it as a **thin, easily replaceable edge**:

    * Static caching.
    * Basic TLS.
    * Maybe WAF, but only with rules you can export/re-implement elsewhere.
  * Don’t bake business logic, auth flows, or data storage into it.
* Keep your **core logic, data, and observability** in OSS projects with neutral governance (CNCF/LF). ([CNCF][1])

That way, if you ever want to drop Front Door for Cloudflare, Fastly, or self-hosted Envoy/Varnish, you can.

---

## 5. So what would they *actually* tell you to do?

If you asked a typical CNCF / Linux Foundation crowd, I think you’d hear something like:

1. **You’re already on the right track.**

   * Kubernetes, Envoy, Argo, OpenTelemetry, Keycloak, Postgres, Redis – this is a very CNCF-shaped stack.

2. **Don’t rush into Azure-specific web frontends unless there’s a killer reason.**

   * If you do use Front Door or Static Web Apps, keep them at the edges and avoid provider-specific features you can’t reproduce elsewhere.

3. **First step: solve performance with OSS you already know.**

   * Tune Envoy/Envoy Gateway as OSS ingress and add a Varnish/ATS caching tier if needed.
   * That gives you a “CDN-ish” setup with no new lock-in.

4. **If you go static for marketing, make it cloud-agnostic.**

   * Static export to generic object storage.
   * Fronted by an OSS proxy/CDN component.
   * Keep CI/CD and routing definitions in Git & K8s, not in a proprietary web UI.

If you’d like, I can sketch a very concrete “CNCF-style” reference diagram for your case (with Envoy Gateway + Varnish + Keycloak + Next.js) so you have a vendor-neutral counterpart to the Azure architecture diagram you already wrote.

[1]: https://www.cncf.io/?utm_source=chatgpt.com "Cloud Native Computing Foundation (CNCF)"
[2]: https://www.cncf.io/projects/envoy/?utm_source=chatgpt.com "Envoy | CNCF"
[3]: https://www.linuxfoundation.jp/wp-content/uploads/2018/03/LFN_ACG_2018_Impact_of_Open_Source-C_03.26.18.pdf?utm_source=chatgpt.com "The Impact of Open Source Technologies on the ..."
[4]: https://www.cncf.io/about/faq/?utm_source=chatgpt.com "FAQ | CNCF"
[5]: https://altinity.com/blog/learning-from-rockset-avoiding-the-dangers-of-lock-in-to-cloud-database-as-a-services?utm_source=chatgpt.com "Learning from Rockset: Avoiding the Dangers of Lock-in to ..."
[6]: https://www.cncf.io/projects/?utm_source=chatgpt.com "Graduated and Incubating Projects"
[7]: https://www.cncf.io/blog/2025/06/11/a-year-of-envoy-gateway-ga-building-growing-and-innovating-together/?utm_source=chatgpt.com "A Year of Envoy Gateway GA: Building, Growing, and ..."
[8]: https://varnish-cache.org/?utm_source=chatgpt.com "Varnish Cache"
