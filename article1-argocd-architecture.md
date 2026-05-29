# ArgoCD Demystified: How It Works Under the Hood and Why It's the Future of Kubernetes Deployments

*A deep dive into ArgoCD's architecture, its components, and how it interacts with Kubernetes — written from a real homelab journey on Raspberry Pi*

---

## Why I Started This Journey

I run a three-node Raspberry Pi Kubernetes cluster in my homelab. When I decided to adopt GitOps properly, I chose ArgoCD — and what followed was one of the most educational deep dives I've had into Kubernetes internals. This article shares everything I learned about how ArgoCD actually works under the hood.

If you've ever wondered "what's actually happening when ArgoCD syncs an app?" — this is for you.

---

## What is GitOps and Why Does It Matter?

Before diving into ArgoCD, it's worth understanding the philosophy it's built on: **GitOps**.

Traditional CI/CD looks like this:

```
Developer → writes code → CI builds image → someone runs kubectl apply → 🤞
```

The problems with this approach are significant:
- No audit trail of who deployed what and when
- Cluster state drifts away from what's documented
- Credentials for the cluster are spread across CI systems
- Rollbacks are painful and manual

GitOps flips this model entirely. It has four core principles:

1. **Declarative** — describe desired state, not steps
2. **Versioned** — Git is the single source of truth
3. **Pulled** — an agent inside the cluster pulls from Git (not CI pushing in)
4. **Continuously reconciled** — the agent constantly compares desired vs actual state

The security improvement alone is significant. In the push model, your CI system holds cluster credentials — if it's compromised, your cluster is too. In the pull model, credentials never leave the cluster boundary.

---

## The ArgoCD Architecture

ArgoCD is a **Kubernetes controller** — it runs inside your cluster and continuously reconciles the desired state (Git) with the live state (cluster). Here's the full component breakdown.

### The 7 Components

When you install ArgoCD, seven pods are created in the `argocd` namespace:

| Component | Role | Critical? |
|-----------|------|-----------|
| `argocd-server` | Web UI + REST API + CLI endpoint | Yes |
| `argocd-application-controller` | The brain — reconciliation loop | Yes |
| `argocd-repo-server` | Git clone + manifest rendering | Yes |
| `argocd-redis` | Caching layer | Yes |
| `argocd-dex-server` | SSO/OIDC authentication | No |
| `argocd-notifications-controller` | Slack/email/webhook alerts | No |
| `argocd-applicationset-controller` | ApplicationSet management | No |

Kubernetes scheduler decides which node each pod runs on — they're not pinned to the control plane. In my Pi cluster, they spread across all three nodes based on available resources.

### The Application Custom Resource Definition

The most important concept to grasp: **an ArgoCD Application is a Kubernetes CRD**. Just like a `Deployment` manages `Pods`, an `Application` manages your app's entire lifecycle.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/your-repo
    targetRevision: HEAD
    path: manifests/guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

This means Application manifests can be stored in Git and managed by ArgoCD itself — which is the foundation of the **App of Apps** pattern.

---

## How ArgoCD Talks to Kubernetes

This is where it gets interesting. ArgoCD follows a strict pattern: **everything goes through the Kubernetes API Server**. It never talks to etcd, the Scheduler, kubelet, or kube-proxy directly.

```
ArgoCD ──────────────────► Kubernetes API Server
                                    │
                          ┌─────────┼─────────┐
                          ▼         ▼         ▼
                        etcd   Scheduler   kubelet
                     (storage) (placement) (execution)
```

Here's exactly how each component interacts:

**With etcd**: ArgoCD never connects to etcd directly. It stores Application CRs in etcd *via* the API Server. When you create an Application, ArgoCD calls the API Server which then writes to etcd.

**With the Scheduler**: ArgoCD creates Deployment objects via the API Server. The Scheduler independently watches for unscheduled pods and picks nodes — ArgoCD has no involvement in this decision.

**With kubelet**: ArgoCD reads pod status from the API Server. Kubelet reports to the API Server; ArgoCD reads those reports to determine application health.

**With kube-proxy**: ArgoCD creates Service objects via the API Server. kube-proxy independently reads those Services and creates iptables rules — again, ArgoCD doesn't interact with kube-proxy at all.

This design is elegant: ArgoCD is just another Kubernetes client, respecting the same API-first principle as everything else.

---

## The Reconciliation Loop — Step by Step

Let me walk through exactly what happens when you create and sync an application.

### Phase 1: Application Creation

```
You click CREATE APP in UI
         │
         ▼
argocd-server
  └── Receives your HTTP request
  └── Validates the Application spec
  └── Creates Application CR → API Server → etcd
  └── Returns success to your browser
```

### Phase 2: Detection

```
argocd-application-controller
  └── Watches API Server for Application CR events
  └── Detects new guestbook Application
  └── Asks argocd-repo-server: "what's in this Git path?"
  └── Compares Git manifests vs cluster state
  └── Marks Application as OutOfSync
```

### Phase 3: Sync

```
You click SYNC
         │
         ▼
argocd-repo-server
  └── Clones the Git repository
  └── Renders templates (plain YAML/Helm/Kustomize)
  └── Returns rendered manifests to app-controller
         │
         ▼
argocd-application-controller
  └── POST /apis/apps/v1/deployments → API Server
  └── POST /api/v1/services → API Server
         │
         ▼
Kubernetes API Server
  └── Stores in etcd
  └── Scheduler picks nodes
  └── kubelet pulls images and starts pods
```

### Phase 4: Status Reporting

```
kubelet reports pod status → API Server
argocd-application-controller reads status
argocd-redis caches the result
argocd-server reads from Redis
UI shows: ❤️ Healthy ✅ Synced
argocd-notifications-controller sends alerts if configured
```

The entire cycle happens continuously — the application-controller runs a reconciliation loop that keeps comparing desired state (Git) with live state (cluster) every few minutes.

---

## ArgoCD's RBAC Model

ArgoCD uses Kubernetes ServiceAccounts with ClusterRoles to interact with the API Server. When you install ArgoCD, it creates these automatically:

**argocd-application-controller** gets:
- `get/list/watch` on ALL resource types
- `create/update/delete` on managed resources
- `update` on Application status

**argocd-server** gets:
- `get/list` on Applications
- `create/delete` on Applications
- Read access to cluster info

This follows the principle of least privilege — each component only gets what it needs.

---

## The CoreDNS Discovery

Running ArgoCD on bare-metal k3s on Raspberry Pi taught me something I wouldn't have learned on managed Kubernetes: **CoreDNS is itself a Kubernetes service**.

CoreDNS runs as a Deployment in `kube-system`, exposed via the `kube-dns` Service at ClusterIP `10.43.0.10`. Every pod gets this address injected into `/etc/resolv.conf` automatically:

```
nameserver 10.43.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
```

When ArgoCD's `argocd-repo-server` needs to connect to `argocd-redis`, it resolves `argocd-redis.argocd.svc.cluster.local` via CoreDNS — which looks up the Service in etcd (via the API Server) and returns the ClusterIP.

This chain — pod → CoreDNS → API Server → etcd → ClusterIP → pod — is the foundation of all Kubernetes service discovery.

---

## k3d vs k3s: A Practical Comparison

My homelab runs k3s directly on three Raspberry Pis for production workloads (Nextcloud, Grafana, Prometheus). For ArgoCD learning, I run k3d — k3s inside Docker containers on the same hardware. The difference taught me a lot.

**k3d** wraps k3s in Docker. Docker manages all networking in isolation — iptables, network interfaces, DNS routing. This means:
- CoreDNS works out of the box
- No conflicts with host OS networking
- Resetting is as simple as `k3d cluster delete && k3d cluster create`

**k3s directly on Pi OS** must manage its own networking and can conflict with existing system rules. I learned this the hard way when running `iptables -F` during troubleshooting broke the entire cluster networking. k3s expects certain iptables chains to exist and doesn't recover gracefully if they're wiped.

For anyone learning ArgoCD on bare metal: use k3d. For production homelab workloads: k3s is excellent.

---

## The App of Apps Pattern

Once you understand that an ArgoCD Application is just a Kubernetes CRD stored in etcd, the App of Apps pattern becomes obvious.

**Store Application manifests in Git → ArgoCD applies them → child Applications are created automatically.**

This is how teams manage dozens or hundreds of services:

```
Git repo: company-gitops/
└── apps/
    ├── frontend.yaml      ← Application CR for frontend
    ├── backend.yaml       ← Application CR for backend
    ├── database.yaml      ← Application CR for database
    └── monitoring.yaml    ← Application CR for monitoring
```

One parent Application points to `apps/`. ArgoCD reads all YAML files, applies them as Application CRs, and suddenly you have four managed applications — created automatically from Git.

Adding a new service: create one YAML file and commit. Removing a service: delete the YAML file. ArgoCD handles everything else.

---

## Why This Architecture is Brilliant

ArgoCD's design elegantly solves three hard problems:

**Security**: Cluster credentials never leave the cluster. Your CI system only needs write access to Git — never to Kubernetes.

**Auditability**: Every deployment is a Git commit. Who deployed what, when, and why is in your Git history. Roll back = `git revert`.

**Drift prevention**: The continuous reconciliation loop means your cluster can't stay in an unauthorized state for long. selfHeal reverts manual changes automatically.

---

## Conclusion

Understanding ArgoCD's internals transformed how I think about Kubernetes deployments. The key insights:

- ArgoCD is just a Kubernetes controller — it follows the same API-first principle as everything else
- Everything communicates through the API Server — never directly to etcd, Scheduler, or kubelet
- An Application is a CRD — which enables the App of Apps pattern
- CoreDNS is itself a Kubernetes service — service discovery is Kubernetes all the way down
- GitOps isn't just a workflow — it's a security model

In my next article, I'll go deep on sync policies — how ArgoCD decides when to sync, what happens when you delete resources manually with different policies, and a surprising discovery about how repository-level polling affects all your applications simultaneously.

---

*I run a three-node Raspberry Pi homelab cluster and document my learnings publicly. Follow for more Kubernetes, GitOps, and homelab content.*

*Tags: #Kubernetes #GitOps #ArgoCD #DevOps #Homelab #RaspberryPi #CloudNative*
