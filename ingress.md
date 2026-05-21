## Are the changes aligned? Yes.

After deploying, two Ingress objects will exist for `dev-stagecraft.ustpace.com`:

| Ingress | Path | Service |
|---|---|---|
| `stage-craft-ai-v1` | `(/api(/?.*))` | stage-craft-ai (backend) |
| `pace-stage-craft-ui-v1` | `/` | pace-stage-craft-ui (frontend) |

- `/api/internal/pipelines/...` → matches `(/api(/?.*))` → **backend → JSON** ✓  
- `/dashboard`, `/`, `/assets/...` → matches `/` → **frontend → HTML** ✓

---

## The concept

**NGINX Ingress Controller** is the traffic cop sitting in front of your cluster. Every HTTP request goes through it first. It reads all `Ingress` objects across the namespace and decides where to forward the request.

```
Browser
  │
  ▼
NGINX Ingress Controller  (dev-stagecraft.ustpace.com)
  │
  ├── /api/*  ──────────────► stage-craft-ai pod  (FastAPI, returns JSON)
  │
  └── /*  ────────────────► pace-stage-craft-ui pod  (Nginx serving Angular, returns HTML)
```

**Why does `/api/*` win over `/*`?**  
NGINX ingress evaluates paths by specificity — longer/more specific paths take priority over catch-alls. `/api/internal/...` matches the backend rule first, so it never reaches the frontend `/` catch-all.

**What does `rewrite-target: $1` do?**  
Without it, the backend pod would receive the full URL as-is. But the path in the ingress rule uses a **capture group** `(/api(/?.*))` — the outer `(...)` is group `$1`. With `rewrite-target: $1`, NGINX rewrites the URL to just what's inside group 1 before forwarding:

```
Incoming:  /api/internal/pipelines/canonical-ir/batch
                │
         capture group $1 = /api/internal/pipelines/canonical-ir/batch
                │
Backend receives:  /api/internal/pipelines/canonical-ir/batch  ✓
```

This is the same mechanism used for `dev.ustpace.com` where the prefix stage-craft-ai is stripped: path is `/stage-craft-ai(/api(/?.*))` → `$1` = `/api/...` → backend never sees stage-craft-ai.

User: any chnage in ingress front end required?

why there is front end and backend ingress?

so where does this flow begin?
and how is it routed?
when the url is hit in broswer," https://dev-stagecraft.ustpace.com/projects"
i get the ui,, thats all i know.
where these middle ingresses come in between.
and what about htese hosts,paths,, rewrite stripping and all?

GitHub Copilot: Read [](file:///home/ust-minnu-antony/PACE/stage-craft-ai-2/edgeops-ui/values-pace-stage-craft-ui-dev.yaml)

**No change needed in the frontend ingress.** It already routes the whole domain `dev-stagecraft.ustpace.com` to the frontend, which is the correct catch-all. The backend ingress we fixed is more specific, so it wins for `/api/*` before the frontend even gets a chance.

---

## The full picture

### Why two separate Ingresses?

Because they are **two separate deployed apps** — one Helm chart for the frontend (edgeops-ui), one for the backend (edgeops-modules). Each chart creates its own `Ingress` object. The NGINX Ingress Controller watches ALL Ingress objects and **merges them** into one routing table at runtime. You don't configure them in one place — you let each app own its piece.

---

### Full flow when you type `https://dev-stagecraft.ustpace.com/projects`

```
Your Browser
     │
     │  DNS lookup: dev-stagecraft.ustpace.com
     ▼
AWS Load Balancer  (internal-ada8f46c7323511eabb33027dc39ec8c-2121569696.us-east-1.elb.amazonaws.com)
     │
     │  forwards all traffic into the cluster on port 80/443
     ▼
NGINX Ingress Controller Pod  (running inside Kubernetes)
     │
     │  reads all Ingress objects in the cluster
     │  builds a routing table:
     │
     │    dev-stagecraft.ustpace.com
     │      /api/*   ──► stage-craft-ai service  (backend)
     │      /*       ──► pace-stage-craft-ui service  (frontend)
     │
     │  your request is /projects → matches /*
     ▼
pace-stage-craft-ui Service  (ClusterIP)
     │
     ▼
pace-stage-craft-ui Pod  (nginx serving the Angular build)
     │
     │  returns index.html
     ▼
Browser receives HTML → Angular boots → renders /projects page
```

---

### What are `hosts`, `paths`, and `rewrite-target`?

**`host`** — which domain name this rule applies to. NGINX reads the `Host:` header of the incoming request and matches it. If you had `staging.ustpace.com` and `dev-stagecraft.ustpace.com` both pointing to the same load balancer, NGINX splits them by this header.

**`paths`** — once the host matches, which URL path should go where. NGINX picks the **most specific** match:

```
/api/internal/pipelines/...  →  matches /api/*  first  →  backend
/projects                    →  doesn't match /api/*   →  falls to /*  →  frontend
```

**`rewrite-target`** — before NGINX forwards the request to your pod, it **rewrites the URL**. The capture group `(...)` in the path is the "what to keep" part. `$1` means "send only what was inside group 1 to the pod":

```
Old domain:  dev.ustpace.com
  Ingress path:   /stage-craft-ai(/api(/?.*))
  Browser sends:  /stage-craft-ai/api/internal/pipelines
                             ↑
                         stripped (not in group 1)
  Backend receives:           /api/internal/pipelines  ✓

New domain:  dev-stagecraft.ustpace.com
  Ingress path:   (/api(/?.*))
  Browser sends:  /api/internal/pipelines
                  ↑
              already group 1, nothing stripped
  Backend receives: /api/internal/pipelines  ✓
```

Without `rewrite-target`, the pod would receive the full raw URL exactly as the browser sent it — which would break the old domain since the FastAPI app doesn't know about the stage-craft-ai prefix in its routes.

---

### Why does the frontend NOT need rewrite?

The Angular app is a **static file server**. It always returns `index.html` for any path (Angular handles routing client-side in the browser). So it doesn't care what path NGINX forwards — no stripping needed.
