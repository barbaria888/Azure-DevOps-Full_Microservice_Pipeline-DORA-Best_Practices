# .NET Microservices on AKS — CI/CD with Azure DevOps, Docker Hub & Trivy
<img src="https://logos-world.net/wp-content/uploads/2024/10/Azure-DevOps-Logo.png" height="60">

> Building a reliable delivery path for containerized .NET microservices — with real cost constraints, actual failures, and the engineering trade-offs that came out of them.

This project started as a learning exercise built on top of the excellent [aspnetrun/run-devops](https://github.com/aspnetrun/run-devops) repository.  
I kept the original application structure (Shopping API + Shopping Client + MongoDB), then rebuilt the entire delivery and deployment path around constraints I actually hit while running this on an **Azure for Students** subscription.

The goal was never to build the most complex pipeline possible.  
It was to understand what happens between a `git push` and a running pod — and to make every piece of that path work for real.

| Image | Status |
| --- | --- |
| Shopping Client | [![Build Status](https://dev.azure.com/hardikaroracs22/Azure-DevOps-Full_Microservice_Pipeline/_apis/build/status/shoppingclient-pipeline?branchName=main)](https://dev.azure.com/hardikaroracs22//_build/latest?definitionId=14&branchName=main) |
| Shopping API | [![Build Status](https://dev.azure.com/ezozkme/shopping/_apis/build/status/shoppingapi-pipeline?branchName=main)](https://dev.azure.com/ezozkme/shopping/_build/latest?definitionId=13&branchName=main) |

---

## Architecture

<img src="https://github.com/barbaria888/Azure-DevOps-Full_Microservice_Pipeline-DORA-Best_Practices/blob/main/images/Gemini_Generated_Image_8eylpp8eylpp8eyl.png">


Two independent pipelines, each following the same pattern:

| Service | Docker Hub Image | Trigger Path | Pipeline |
|---|---|---|---|
| Shopping API | `hardik0811/shoppingapi` | `Shopping/Shopping.API/**` | `shoppingapi-pipeline.yaml` |
| Shopping Client | `hardik0811/shoppingclient` | `Shopping/Shopping.Client/**` | `shoppingclient-pipeline.yaml` |

**CI stage** (runs on `main`, `develop`, and `Deployment` branches):
1. Restore & build (.NET 8)
2. Docker build with correct build context
3. Trivy container scan (report published as artifact)
4. Push to Docker Hub/ACR(prior to optimisation)
   
<img src="https://github.com/barbaria888/Azure-DevOps-Full_Microservice_Pipeline-DORA-Best_Practices/blob/main/images/Pasted%20image%2020260917004746.png">

**CD stage** (runs only on `main` and `Deployment`, with environment approval):
1. Deploy shared infrastructure (MongoDB, ConfigMaps, Secrets, HPA)
2. Deploy the service to AKS via `KubernetesManifest@1`
[<img src="https://github.com/felicitousbeing999-h/Obsidian01/blob/main/Pasted%20image%2020260916191822.png">](https://github.com/felicitousbeing999-h/Obsidian01/blob/main/Pasted%20image%2020260916200633.png)

---

## Engineering Decisions & Trade-Offs

These are the choices I made and why. I tried to reason about each one rather than default to "what the tutorial said."

### 1. ACR → Docker Hub — Cost Engineering

The original project used Azure Container Registry. That makes sense for enterprise workloads.  
On a student subscription, it was an unnecessary recurring cost for a project with two small images.

| | Before | After |
|---|---|---|
| Registry | ACR (paid) | Docker Hub (free tier) |
| Monthly impact | Recurring registry cost | Near-zero |
| Trade-off | Private registry + ACR integration | Public images, manual `imagePullSecrets` |

This single decision reduced my monthly Azure bill by **~36.85%**.

> This was not "ACR is bad."  
> It was a conscious trade-off: for this scale, the operational and financial overhead of a private registry was not justified.

I wrote about the full migration and cost analysis here:  
→ [Right-sizing my microservice Kubernetes setup](https://hardik0811arora.hashnode.dev/right-sizing-my-microservice-based-kubernetes-deployment-escaping-acr-costs-and-fixing-oomkilled-pods)

### 2. Docker Build Context Alignment

The Dockerfiles expect the build context to be the `Shopping/` folder:

```dockerfile
COPY ["Shopping.API/Shopping.API.csproj", "Shopping.API/"]
```

Running the build from the repository root fails silently — Docker just can't find the files.  
The pipeline sets `buildContext: '$(Build.SourcesDirectory)/Shopping'` to align all three pieces:

```text
Dockerfile path  +  Build context  +  COPY paths  =  Working build
```

This was a small detail that cost me several hours of debugging. It's the kind of thing that's obvious in hindsight but invisible in the error message.

### 3. Artifact Identity Must Be Consistent

One of the more educational (and embarrassing) bugs:

```text
Build produced:  hardik0811/shoppingapi:31
Push tried:      hardik0811/shoppingclient:31
```

Docker correctly said the image didn't exist. The root cause was a variable mismatch across stages.

The rule that came out of it:

> **Build, scan, and push must agree on the exact identity of the artifact.**

Both pipelines now construct the image name identically at every stage:
```
$(dockerHubUsername)/$(imageRepository):$(Build.BuildId)
```

### 4. Trivy Security Gate — Soft Now, Hard Later

Trivy is intentionally configured with `--exit-code 0` and `continueOnError: true`.

This is not because I don't care about security.  
It's because I wanted to prove the artifact can travel through the entire system first, before a security gate could mask deployment problems.

The intended progression:

```text
Phase 1 (current):  Scan → Report → Continue    (prove the path works)
Phase 2 (next):     Scan → PASS → Continue
                         → FAIL → Stop pipeline  (enforce the gate)
```

A soft security gate during build-out is useful.  
It should never quietly become the permanent configuration.

### 5. Reusable Templates Over Copy-Paste

Each service has its own pipeline file, but common steps live in shared templates:

```text
templates/
├── ci/
│   └── build-dotnet.yml        # .NET restore, build, Docker build
├── security/
│   └── trivy-scan.yml          # Container vulnerability scan
├── cd/
│   ├── deploy-infra.yml        # Mongo, ConfigMaps, Secrets, HPA
│   └── deploy-aks.yml          # Service deployment to AKS
└── common/
    └── variables.yml           # All shared configuration
```

The service pipeline defines **what** is different (project path, Dockerfile, image name).  
The template defines **how** to build and deploy any service.

If I add a third microservice tomorrow, it's a new pipeline file with four variables — not another 80 lines of almost-identical YAML.

### 6. Resource Right-Sizing — The OOMKilled Incident

On a single-node AKS cluster (`Standard_B2s_v2`), I hit a classic resource trap:

- MongoDB was repeatedly **OOMKilled** because the memory limit was set to `128Mi`
- The API started returning `500`s because it couldn't reach the database
- The client showed blank pages

The fix was not "give everything more resources."  
It was to size requests and limits based on what the workload actually needs:

| Component | Before | After | Why |
|---|---|---|---|
| MongoDB memory | `64Mi / 128Mi` | `256Mi / 512Mi` | `mongod` baseline is ~200Mi |
| MongoDB CPU | `250m / 500m` | `100m / 250m` | Mongo is memory-bound, not CPU-bound |
| Shopping API | `250m / 500m` | `50m / 250m` | Small .NET API, minimal CPU at rest |

### 7. AKS with Entra ID + Azure RBAC

The cluster uses Azure RBAC instead of Kubernetes-native RBAC:

```bash
az aks create \
  --enable-aad \
  --enable-azure-rbac \
  --node-vm-size Standard_B2s_v2 \
  --node-count 1
```

This means:
- Authentication goes through **Entra ID** (formerly Azure AD)
- Authorization is managed via **Azure role assignments**, not `ClusterRoleBindings`
- Pipeline uses `useClusterAdmin: true` to bypass `kubelogin` issues in CI

---

## The Failures That Taught Me the Most

This is the condensed version. The full "Caveman War Log" lives in my [Obsidian notes](https://github.com/felicitousbeing999-h/Obsidian01/tree/main/Azure-Devops-Microservice-App).

```mermaid
timeline
    title Building the Pipeline — Failure Timeline
    section Registry
        ACR cost too high : Migrated to Docker Hub free tier
    section Cluster
        AKS deleted to save money : Recreated with 1-node B2s
        Cloud Shell crashed on paste : Switched to file + --no-wait
    section Pipeline
        Service connection not found : Wrong type — needed ARM, not Kubernetes
        Trivy artifact name invalid : Slash in name — renamed to trivy-report
        403 on listClusterUserCredential : Added RBAC role to service principal
        kubelogin not found : Set useClusterAdmin to true
    section Runtime
        MongoDB OOMKilled at 128Mi : Bumped to 256Mi/512Mi
        API returning 500s : Mongo was dead — cascading failure
        kubectl Forbidden : Needed --admin credentials
        Wrong namespace : Deployed to application, expected default
```

A few that stand out:

| Failure | What I Learned |
|---|---|
| **Trivy artifact name had a `/` in it** | `hardik0811/shoppingapi` is a valid image name but an invalid Azure DevOps artifact name. Renamed to `trivy-report`. |
| **Cloud Shell disconnected on long paste** | Write scripts to a file, run with `--no-wait`, poll status. Don't paste 40 lines into a web terminal. |
| **403 on AKS credentials** | With Azure RBAC enabled, the service principal needs an explicit role assignment on the cluster resource. It's not automatic. |
| **CD deployed to wrong namespace** | Variables said `application`, but the cluster was configured for `default`. A one-word mismatch that silently succeeded with zero running pods. |

---

## Repository Structure

```text
run-devops/
│
├── Shopping/                           # Application source code
│   ├── Shopping.API/                   # .NET 8 Web API + Dockerfile
│   └── Shopping.Client/                # .NET 8 MVC Client + Dockerfile
│
├── pipelines/                          # Azure DevOps pipeline definitions
│   ├── shoppingapi-pipeline.yaml       # CI/CD for Shopping API
│   ├── shoppingclient-pipeline.yaml    # CI/CD for Shopping Client
│   └── templates/
│       ├── ci/
│       │   └── build-dotnet.yml        # .NET build + Docker build template
│       ├── security/
│       │   └── trivy-scan.yml          # Trivy container scan template
│       ├── cd/
│       │   ├── deploy-infra.yml        # Shared infra (Mongo, ConfigMaps, HPA)
│       │   └── deploy-aks.yml          # Service deployment to AKS
│       └── common/
│           └── variables.yml           # All shared pipeline variables
│
├── k8s/                                # Kubernetes manifests (used by CD)
│   ├── mongo.yaml                      # MongoDB Deployment + Service
│   ├── mongo-secret.yaml               # MongoDB credentials
│   ├── mongo-configmap.yaml            # MongoDB connection string
│   ├── shoppingapi-configmap.yaml      # API configuration
│   ├── shoppingapi.yaml                # API Deployment + Service
│   └── shoppingclient.yaml             # Client Deployment + LoadBalancer
│
└── aks/                                # AKS-specific configurations
    ├── shoppingautoscale.yaml          # HPA for API + Client
    ├── mongo.yaml                      # AKS-tuned Mongo (right-sized)
    └── commands.txt                    # AKS setup reference commands
```

---

## Current Status

| Layer | Status | Notes |
|---|---|---|
| .NET 8 build | ✅ Working | Restore, build, correct SDK version |
| Docker build | ✅ Working | Correct build context (`Shopping/`) |
| Trivy scan | ✅ Working | Soft-fail, report published as artifact |
| Push to Docker Hub | ✅ Working | `hardik0811/shoppingapi` + `shoppingclient` |
| AKS cluster | ⏸️ Paused | Taken offline to manage costs — Azure RBAC + Entra ID configured |
| CD to AKS | ✅ Validated | Deploys successfully when cluster is running |
| HPA autoscaling | ✅ Configured | Scales on CPU utilization |
| Terraform for AKS | 🗓️ Planned | Next phase — infrastructure as code |

---
<img src="https://github.com/felicitousbeing999-h/Obsidian01/blob/main/Pasted%20image%2020260916192144.png">
## What I Deliberately Left Out (For Now)

| Item | Reason |
|---|---|
| Azure Container Registry | Cost for this scale wasn't justified |
| Full GitOps (ArgoCD / Flux) | Scope control — get one path working first |
| Azure Policy / Governance | Not the current learning goal |
| Terraform for AKS | Planned next, after the application path is stable |
| Helm charts | Keeping manifests simple while learning the fundamentals |
| Multi-environment (staging/prod) | Single `dev` environment is sufficient for learning |

I prefer to get one path working cleanly before adding the next layer.

---
<img src="https://github.com/felicitousbeing999-h/Obsidian01/blob/main/Pasted%20image%2020260916195802.png">

## DORA & Continuous Improvement

This project is named with [DORA](https://dora.dev/) in mind — not because I've implemented their full framework, but because the DORA community's way of thinking about software delivery resonates with how I want to learn.

The four key metrics they focus on:

| Metric | Where I See It in This Project |
|---|---|
| **Deployment Frequency** | Independent pipelines per service, automated trigger on push |
| **Lead Time for Changes** | Template reuse reduces time from commit to deployed artifact |
| **Change Failure Rate** | Trivy scanning catches vulnerabilities before they reach the cluster |
| **Time to Restore** | Environment approval gate, separate infra + app deployment |

I'm not measuring these formally yet. But the pipeline was designed with these ideas in the background — small, independent, automated, and observable.

The DORA community taught me that improving delivery isn't about adding more tools.  
It's about understanding where time and risk actually live in your system.

---

## Writing & Deeper Notes

I documented the messy parts — the ones that actually taught me something:

📝 **[My Breaking and Learning Journey with Azure DevOps CI](https://hardik0811arora.hashnode.dev/my-breaking-and-learningjourney-with-azure-devops)**  
Docker context issues, image name mismatches, Trivy soft-fail strategy, reusable templates, and why I temporarily removed Kubernetes.

📝 **[Right-Sizing the Setup — ACR → Docker Hub + OOMKilled Fix](https://hardik0811arora.hashnode.dev/right-sizing-my-microservice-based-kubernetes-deployment-escaping-acr-costs-and-fixing-oomkilled-pods)**  
The cost decision, MongoDB OOMKilled incident, and resource right-sizing on a constrained node pool.

📓 **[Obsidian Engineering Notes](https://github.com/felicitousbeing999-h/Obsidian01/tree/main/Azure-Devops-Microservice-App)**  
Raw debugging sessions, AKS deployment commands, incident post-mortem, and the full "Caveman War Log" of every failure and fix.

---

## Credits

This project is built on top of the original work by **aspnetrun**:

- Original repository: [aspnetrun/run-devops](https://github.com/aspnetrun/run-devops)

I kept the Shopping API / Client application structure and the overall architecture picture, then reworked the delivery path, registry choice, pipeline structure, deployment manifests, and operational decisions for a smaller, cost-conscious environment.

---

## A Note

```
I'm still early in my career, and I'm building in public because I believe 
the best way to learn is to show your work — including the failures.

What I care about is understanding the real trade-offs: 
cost, complexity, security gates, and operational reality.

If you're a senior engineer reading this - I would genuinely love your feedback.
If you're someone learning the same things - I hope this helps. 
Let's figure it out together.
```

**Connect:** [LinkedIn](https://www.linkedin.com/in/hardik0811arora/) · [Hashnode](https://hardik0811arora.hashnode.dev/) · [Twitter/X](https://x.com/HardikArora0811)

---
