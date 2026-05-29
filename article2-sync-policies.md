# ArgoCD Sync Policies: What Actually Happens When You Delete a Kubernetes Resource

*Five hands-on experiments that reveal the truth about manual sync, automated sync, selfHeal, and a surprising discovery about repository-level polling — all tested on a real Raspberry Pi cluster*

---

## The Question That Started It All

"What actually happens if someone deletes a Kubernetes deployment that ArgoCD is managing?"

It sounds simple. The answer turns out to be nuanced — and depends entirely on which sync policy you've configured. I ran five experiments on my three-node Raspberry Pi k3s cluster to find out exactly what happens in each scenario. What I discovered surprised me.

---

## Understanding Sync Policies

ArgoCD has three main sync policy configurations, each with fundamentally different behavior:

```yaml
# Policy 1: Manual (default)
syncPolicy: {}

# Policy 2: Automated
syncPolicy:
  automated: {}

# Policy 3: Automated + selfHeal + prune
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

Before I ran any experiments, my mental model was: "automated syncs automatically, manual doesn't." What I discovered was considerably more interesting.

---

## The Setup

I'm running:
- k3d cluster (k3s inside Docker) on Raspberry Pi ARM64
- ArgoCD v3.4.2
- App of Apps pattern with a `root-app` managing child applications
- Two apps: `guestbook` and `nginx-app`, both pointing to my GitOps repo

The App of Apps pattern matters here: sync policies for child apps are defined in Git YAML files, not set via CLI. Attempting to change sync policy via `argocd app set` gets reverted by the parent app's next sync cycle. This is actually correct GitOps behavior — the Git repo is authoritative for everything, including sync policies.

---

## Experiment 1: Manual Sync + kubectl delete

**Setup**: `syncPolicy: {}` (manual)

**Action**:
```bash
kubectl delete deployment guestbook-ui -n default
```

**Result**:
```
Pods:       No resources found ❌
Deployment: No resources found ❌
ArgoCD:     OutOfSync detected ⚠️
            App stays DOWN — no auto-recovery
```

ArgoCD correctly detected the drift and showed `OutOfSync`. But it did absolutely nothing. The app stayed down.

**How long did it stay down?** Until I ran `argocd app sync guestbook` manually. In a production scenario without active monitoring, this could mean hours of downtime before anyone notices.

**Recovery command**:
```bash
argocd app sync argocd/guestbook
# Deployment recreated, 3 pods Running in seconds
```

**Conclusion**: Manual sync policy puts the operator in full control of when deployments happen. It's appropriate for production environments where human approval before deployment is required. The tradeoff is that accidental deletions create downtime until someone intervenes.

---

## Experiment 2: Automated (No selfHeal) + kubectl delete

**Setup**: `automated: prune: true` (no selfHeal)

**Action**:
```bash
kubectl delete deployment guestbook-ui -n default
```

**Immediate result**:
```
Pods:   No resources found ❌
ArgoCD: OutOfSync detected ⚠️
        No auto-recovery... yet
```

This is where my original mental model was wrong. I assumed "automated" meant ArgoCD would immediately recover from any drift. It doesn't.

**Automated sync triggers on Git changes** — not on cluster drift. Without selfHeal, automated only means "sync when Git changes." Cluster drift without a Git change trigger = no sync.

**What I did next**: waited. The app stayed down. Then I made a change to `manifests/nginx/deployment.yaml` in GitHub — a completely different application's manifest file.

**What happened**: the guestbook deployment was recreated.

This led to a surprising discovery.

---

## The Surprising Discovery: Repository-Level Polling

When I changed the nginx deployment file, guestbook recovered. I initially assumed this proved "different file in same app path triggers sync." But guestbook and nginx are *different applications* pointing to *different paths* in the same repository.

Checking the sync history:

```bash
argocd app history argocd/guestbook | tail -3
# Showed a sync at the exact time I changed the nginx file
```

**The discovery**: ArgoCD polls at the **repository level**, not the file level or the application path level.

When *any* file changes in a repository, ArgoCD re-evaluates *all* applications that watch that repository. If any of those applications are OutOfSync, ArgoCD auto-syncs them.

This has real implications:

| Change | Guestbook syncs? | Why? |
|--------|-----------------|------|
| guestbook file changes | Yes ✅ | Direct path change |
| nginx file changes (same repo) | Yes ✅ | Same repo → re-evaluates all apps! |
| nginx file changes (different repo) | No ❌ | Different repo, not monitored |
| No Git change at all | No ❌ | No trigger without selfHeal |

**Practical implication**: In a monorepo GitOps setup, any commit will trigger re-evaluation of all apps in that repo. Drifted apps can get inadvertently recovered by unrelated commits. This can be either a feature or a footgun depending on your perspective.

If you need true app isolation, use separate repositories per team or application.

---

## Experiment 3: The Clean Isolation Test

To truly test automated without selfHeal, I would need:
- guestbook pointing to Repo A
- nginx pointing to Repo B

Then:
1. Delete guestbook deployment (Repo A drift detected)
2. Change nginx in Repo B
3. Guestbook should stay down (different repo, no re-evaluation)

This confirmed the repository-level polling behavior from the other direction: **only changes in the *same* repository trigger re-evaluation of affected applications**.

**Conclusion for automated without selfHeal**: Your app will recover eventually — as long as someone commits something to the same repository. In an active development environment with frequent commits, this might mean recovery in minutes. In a dormant repo, it could be hours.

---

## Experiment 4: Automated + selfHeal + kubectl delete

**Setup**: `automated: prune: true, selfHeal: true`

**Action**:
```bash
kubectl delete deployment guestbook-ui -n default
kubectl get pods -n default -w
```

**Result**:
```
(deployment deleted)
guestbook-ui-xxx   0/1   Pending            0s
guestbook-ui-xxx   0/1   ContainerCreating  0s
guestbook-ui-xxx   1/1   Running            4s  ← 4 SECONDS!
```

The deployment was back in **4 seconds**. I ran this test multiple times — the recovery time was consistently under 10 seconds.

**Why so fast?** selfHeal runs a dedicated watch on cluster state. Unlike automated (which waits for a Git change trigger), selfHeal watches the cluster directly and acts immediately when it detects drift, without waiting for a polling cycle.

**Repeated the test**: Deleted the deployment three more times. Each time, 3 pods were running again within 4-10 seconds. With selfHeal enabled, it's practically impossible to keep the app down through kubectl deletions alone.

**Conclusion**: selfHeal provides true GitOps enforcement. The cluster cannot deviate from Git state for more than a few seconds. This is appropriate for staging environments and production apps where accidental or malicious kubectl changes must be immediately reverted.

---

## Experiment 5: Verifying selfHeal via JSON

This experiment revealed an important ArgoCD CLI quirk.

After enabling selfHeal, the CLI showed:
```
Sync Policy: Automated (Prune)
```

No mention of selfHeal. I verified the Git YAML had `selfHeal: true`. To check what was actually in the cluster:

```bash
kubectl get application guestbook -n argocd \
  -o jsonpath='{.spec.syncPolicy}' | python3 -m json.tool
```

Output:
```json
{
    "automated": {
        "prune": true,
        "selfHeal": true
    },
    "syncOptions": [
        "CreateNamespace=true"
    ]
}
```

selfHeal was active — the CLI just doesn't display it explicitly.

**Lesson**: Don't trust the ArgoCD CLI's short display for sync policy verification. Use `kubectl get application -o json` to see the actual spec.

---

## The Complete Truth Table

After all five experiments, here's the definitive behavior matrix:

| Scenario | Manual | Auto (no selfHeal) | Auto + selfHeal |
|----------|--------|-------------------|-----------------|
| Delete + no Git change | Down ❌ | Down ❌ | Recovers in ~4s ✅ |
| Delete + same repo commit | Down ❌ | Recovers ✅ | Recovers in ~4s ✅ |
| Delete + different repo commit | Down ❌ | Down ❌ | Recovers in ~4s ✅ |
| Manual kubectl patch | Stays changed | Stays changed | Reverted to Git ✅ |
| Git change | Waits for manual sync ⏳ | Auto-syncs ✅ | Auto-syncs ✅ |

---

## How to Actually Delete Resources in GitOps

One question that came up naturally: what's the *right* way to delete a Kubernetes resource when ArgoCD is managing it?

**Wrong approach**: `kubectl delete deployment guestbook-ui`
- selfHeal recreates it immediately
- Without selfHeal, it stays deleted until next Git commit
- No audit trail
- Breaks GitOps principles

**Right approach**: Remove the file from Git and let ArgoCD prune it.

With `prune: true` in your sync policy:
```
Delete manifests/guestbook/deployment.yaml from Git
→ Commit and push
→ ArgoCD detects the file is gone
→ Prune deletes the deployment from cluster
→ Clean deletion with full Git audit trail ✅
```

This is the GitOps way. Every deletion is a commit. Every commit has an author, timestamp, and message. You can revert any deletion with `git revert`.

The finalizer in Application manifests ensures clean cascading deletion:

```yaml
metadata:
  finalizers:
    - resources-finalizer.argocd.argoproj.io
```

With this finalizer, deleting an ArgoCD Application also deletes all the Kubernetes resources it managed. Without it, resources are orphaned — still running but no longer managed.

---

## The App of Apps Sync Policy Gotcha

In the App of Apps pattern, sync policies for child apps are defined in their Git YAML files — not via CLI. This caused a confusing moment:

```bash
argocd app set argocd/guestbook --sync-policy none
# Appears to work...

# But 3 minutes later:
argocd app get argocd/guestbook | grep "Sync Policy"
# Sync Policy: Automated (Prune)  ← reverted!
```

The root-app detected that the guestbook Application CR in the cluster differed from `apps/guestbook.yaml` in Git and reverted the change. This is correct behavior — the root-app is enforcing Git as the source of truth for the application definitions themselves.

**Rule**: In App of Apps, always change sync policies in Git, never via CLI.

---

## Production Recommendations

Based on the experiments:

**Development environments**: `automated + selfHeal + prune`
- Fast feedback loop
- Auto-recovery from any mistakes
- Full GitOps enforcement for learning good habits

**Staging environments**: `automated + selfHeal + prune`  
- Mirrors production behavior
- Tests that your GitOps workflows actually work

**Production environments**: Depends on risk tolerance:
- High-risk or regulated: `manual` — human approval before every deployment
- Standard services: `automated + selfHeal` — fast recovery, audit trail in Git
- Never: direct kubectl changes — defeats the purpose of GitOps

---

## Useful Commands for Debugging

```bash
# See what's different between Git and cluster
argocd app diff argocd/guestbook

# See full sync history with commit revisions
argocd app history argocd/guestbook

# Pull latest Git state without syncing
argocd app get argocd/guestbook --refresh

# Verify actual sync policy (not just CLI display)
kubectl get application guestbook -n argocd \
  -o jsonpath='{.spec.syncPolicy}' | python3 -m json.tool

# Force immediate sync
argocd app sync argocd/guestbook --force
```

`argocd app diff` is especially powerful — it shows exactly what ArgoCD will change during a sync, like `git diff` but for your cluster state.

---

## Key Takeaways

Five experiments, five lessons:

1. **Manual sync** means the app stays down until you manually intervene — even if ArgoCD detected the drift immediately

2. **Automated sync** triggers on Git changes, not on cluster drift. A drifted app won't recover until something commits to the same repository

3. **Repository-level polling** means any commit to a repo re-evaluates all apps in that repo — not just the app whose files changed

4. **selfHeal** provides true GitOps enforcement with ~4 second recovery time. It's not optional for serious GitOps

5. **Always delete resources from Git** — let ArgoCD prune them. Never use kubectl delete on ArgoCD-managed resources

The deeper lesson: GitOps is not just a deployment workflow — it's a contract. Git is the authoritative source of truth, and ArgoCD enforces that contract continuously. The sync policy determines how strictly that contract is enforced and how quickly violations are corrected.

---

*I run a three-node Raspberry Pi homelab cluster and document my learnings publicly. The next article in this series covers Helm and Kustomize integration with ArgoCD.*

*Tags: #Kubernetes #GitOps #ArgoCD #DevOps #Homelab #CloudNative #Platform Engineering*
