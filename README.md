# eks-gitops
Here’s a clean, interview‑ready, **end‑to‑end story + commands** you can actually run, centered on your `eks-gitops` cluster and repo.

---

## 1. Prerequisites and context

**Goal you’ll describe:**

> “I built an EKS cluster named `eks-gitops` and wired it to a GitOps repo on GitHub using Flux CD, so all workloads and infra are managed declaratively from Git.”

**Tools installed locally:**

- **AWS CLI** configured with an IAM user/role that can manage EKS, EC2, IAM, CloudFormation  
- **kubectl** for Kubernetes interaction  
- **eksctl** to create and manage the EKS cluster  
- **flux CLI** to bootstrap Flux CD against the cluster and Git repo  

---

## 2. Create the EKS cluster with eksctl (CloudFormation under the hood)

You can say:

> “I used `eksctl` to provision the EKS cluster. Under the hood, it generates and manages CloudFormation stacks for the control plane, node groups, IAM roles, and networking, giving me reproducible infrastructure as code.”

### 2.1 Create the cluster

```bash
eksctl create cluster \
  --name eks-gitops \
  --region us-east-1 \
  --nodes 2 \
  --node-type t3.medium \
  --with-oidc \
  --managed
```

This:

- Creates the **EKS control plane**  
- Creates **managed node groups**  
- Enables **OIDC provider** for IRSA  
- Uses **CloudFormation stacks** to manage all resources  

### 2.2 Verify the cluster

```bash
aws eks describe-cluster \
  --name eks-gitops \
  --region us-east-1 \
  --query "cluster.status"

kubectl get nodes
kubectl get pods -A
```

You expect:

- Cluster status: `"ACTIVE"`  
- Nodes: `Ready`  

---

## 3. Prepare the GitOps repo (your existing `eks-gitops` repo)

You already have:

```text
https://github.com/arkinfotech24/eks-gitops.git
```

Structure (what you can describe):

> “I structured the GitOps repo with clear separation of concerns:  
> - `clusters/` for environment definitions  
> - `apps/` for workloads  
> - `infrastructure/` for shared components like the AWS ALB controller.”  

Example (your actual tree):

```text
apps/demo-app/base
apps/demo-app/overlays/prod
clusters/prod/...
infrastructure/ingress/aws-alb/...
```

This aligns with GitOps best practices and Flux’s model of **GitRepository + Kustomization**.

---

## 4. Bootstrap Flux CD against the cluster and GitHub repo

You can frame it like this:

> “I used Flux CD as the GitOps engine. Flux continuously reconciles the `eks-gitops` repo into the EKS cluster, so Git is the single source of truth for workloads and infra.”

### 4.1 Install Flux controllers into the cluster

From your local machine, with `kubectl` pointing at `eks-gitops`:

```bash
flux install
```

This installs the Flux controllers into the `flux-system` namespace.

### 4.2 Configure Flux to watch your GitHub repo

```bash
flux create source git eks-gitops \
  --url=https://github.com/arkinfotech24/eks-gitops.git \
  --branch=master \
  --interval=1m \
  --export > clusters/prod/git-source.yaml
```

Then apply:

```bash
kubectl apply -f clusters/prod/git-source.yaml
```

This defines a **GitRepository** source pointing at your repo.

---

## 5. Wire Flux to your app and infra via Kustomizations

You can say:

> “I used Flux Kustomizations to map Git paths to cluster state. Each Kustomization points at a folder in the repo and applies it into the cluster.”

### 5.1 Kustomization for the app

Example `clusters/prod/kustomization.yaml` (you already have something similar):

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: demo-app
  namespace: flux-system
spec:
  interval: 1m
  path: ./apps/demo-app/overlays/prod
  prune: true
  sourceRef:
    kind: GitRepository
    name: eks-gitops
  targetNamespace: prod
```

Apply:

```bash
kubectl apply -f clusters/prod/kustomization.yaml
```

This tells Flux:

- Watch `apps/demo-app/overlays/prod` in Git  
- Apply it into the `prod` namespace  
- Reconcile every minute  

### 5.2 Kustomization for infrastructure (ALB controller)

Example `infrastructure/ingress/aws-alb/kustomization.yaml` (pattern):

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: aws-alb
  namespace: flux-system
spec:
  interval: 1m
  path: ./infrastructure/ingress/aws-alb
  prune: true
  sourceRef:
    kind: GitRepository
    name: eks-gitops
  targetNamespace: kube-system
```

Apply:

```bash
kubectl apply -f infrastructure/ingress/aws-alb/kustomization.yaml
```

This deploys the AWS Load Balancer Controller and related resources via GitOps.

---

## 6. Deploy and manage workloads via GitOps

You can describe this flow:

> “Once Flux is wired to the repo, any change to the manifests—Deployments, Services, Ingress, HelmReleases—is done via pull request. When merged, Flux reconciles the new desired state into the cluster. No one is kubectl‑ing changes directly into prod.”

### 6.1 Verify Flux is reconciling

```bash
kubectl get pods -n flux-system
flux get kustomizations
flux get sources git
```

You should see:

- Flux controllers running  
- Kustomizations `Ready`  
- Git source `Ready`  

### 6.2 Verify the app and ingress

```bash
kubectl get deploy -n prod
kubectl get svc -n prod
kubectl get ingress -n prod
```

You can then hit the ALB DNS (like the nginx page you have open) to confirm end‑to‑end traffic.

---

## 7. (Optional) Add image automation with ECR

You can mention this as “in progress” or “next step”:

> “The next step I’ve been wiring in is Flux Image Automation. The idea is: GitHub Actions builds and pushes images to ECR, Flux watches the registry, updates the image tag in the manifests, and rolls out the new version automatically—still fully GitOps‑driven.”

High‑level steps (no need to go deep unless asked):

1. Create ECR repo.  
2. Configure GitHub Actions to build and push images.  
3. Add Flux `ImageRepository` and `ImagePolicy`.  
4. Add `ImageUpdateAutomation` to update manifests in Git.  

---

## 8. How to summarize this to the Director of DevOps

You can wrap it like this:

> “Recently I built an EKS cluster called `eks-gitops` using `eksctl`, which leverages CloudFormation under the hood for reproducible infrastructure. I then bootstrapped Flux CD and pointed it at a dedicated GitOps repo on GitHub (`arkinfotech24/eks-gitops`).  
> 
> The repo is structured with `clusters/` for environment definitions, `apps/` for workloads, and `infrastructure/` for shared components like the AWS ALB controller. Flux GitRepository and Kustomization objects map those paths into the cluster, so any change is done via pull request and reconciled automatically.  
> 
> The result is a fully GitOps‑driven EKS environment: no manual changes in prod, full auditability, easy rollbacks, and a clear separation between application code, infrastructure, and cluster configuration.”

If you want, I can next:

- Draft a **2–3 minute “walk me through this project”** answer  
- Or add the **GitHub Actions + ECR + Flux Image Automation** layer as a second phase of the story.

Resource Links:
https://knowledge.businesscompassllc.com/build-configure-deploy-full-eks-cluster-creation-tutorial-with-eksctl/?utm_source=copilot.com

https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html?utm_source=copilot.com

https://github.com/aws-samples/eks-fluxcd-bootstrap?utm_source=copilot.com
