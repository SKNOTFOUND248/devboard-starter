# DevBoard on EKS — ArgoCD + Gateway API + Envoy Gateway

This is the same DevBoard app you already ran on Kind (`docs/kubernetes.md`),
now on a real AWS EKS cluster, with **ArgoCD** doing the `kubectl apply`s for
you by watching this git repo, and **Gateway API + Envoy Gateway** exposing
it — same concepts as the Kind Gateway setup, but with a real AWS Network
Load Balancer instead of a NodePort + Docker port mapping.

```
you push to git → ArgoCD notices → applies k8s/eks/* → EKS cluster
                                                            │
                            AWS NLB ← Envoy Gateway ← Gateway API ← frontend → backend → postgres (EBS)
```

## What's different from Kind, and why

| Kind | EKS | Why |
|---|---|---|
| `k8s/10-postgres-pv.yml` — static `hostPath` PV | `k8s/eks/04-postgres-storageclass.yml` — dynamic EBS `StorageClass` | `hostPath` ties data to one node's local disk. On EKS, nodes are separate EC2 instances — you need real, dynamically-provisioned block storage (EBS via the EBS CSI driver). |
| `k8s/04-frontend-service.yml` — `NodePort` + Kind's `extraPortMappings` | `k8s/eks/11-frontend-service.yml` — `ClusterIP`, fronted by Gateway API | Kind has no cloud load balancer, so NodePort + a Docker port mapping was the workaround. EKS has a real cloud provider — Gateway API + Envoy Gateway + the AWS Load Balancer Controller give you a proper external NLB. |
| manual `kubectl apply -f k8s/` | ArgoCD `Application` watching `k8s/eks/` in this repo | GitOps: push to `main`, ArgoCD reconciles the cluster automatically (and self-heals if someone `kubectl edit`s something by hand). |

`k8s/eks/` is a flat, self-contained folder (same numbering style as `k8s/`)
— no Kustomize, nothing to learn beyond what you already know from Kind.

## Prerequisites

- AWS CLI configured (`aws sts get-caller-identity` works)
- `eksctl`, `kubectl`, `helm` installed
- This repo pushed to GitHub, or your own fork of it (ArgoCD reads it over
  HTTPS) — if you're working from a fork, update `repoURL` in Step 6's
  `Application` manifest to point at your fork instead

**Region used in this walkthrough:** `ap-south-1` (Mumbai). Swap it for
whichever AWS region you prefer — just keep it consistent across every
command below, since `eksctl`/`aws` don't infer region from context.

**Cost:** this is real, billable AWS infra, not free-tier. Roughly (ap-south-1,
on-demand): EKS control plane ~$0.10/hr + 2× `t3.medium` ~$0.084/hr each +
one NLB ~$0.025/hr + LCUs + a 1Gi gp3 EBS volume (negligible) — call it
**~$0.30-0.40/hr** total while it's running. Tear it down (§8) when you're
done with it for the day.

---

## 1. Create the EKS cluster

Pinned to Kubernetes **1.36** — matches the Kind cluster's `v1.36.1`
(`kind/kind-config.yaml`). Confirmed available on EKS as of Sept 2026 (EKS
guarantees at least 4 production-ready versions at any time; 1.36 is
current, 1.37 is already out in EKS Distro). Unlike Kind, EKS tracks minor
versions only — AWS manages the patch/AMI level for you.

> EKS 1.33+ only ships **Amazon Linux 2023 (AL2023)** node AMIs (AL2 is
> retired from 1.33 onward) — since we're on 1.36, the managed nodegroup
> below uses AL2023 automatically, no extra flag needed.

```bash
eksctl create cluster \
  --name devboard-eks \
  --region ap-south-1 \
  --version 1.36 \
  --nodegroup-name devboard-workers \
  --node-type t3.medium \
  --nodes 2 --nodes-min 2 --nodes-max 3 \
  --managed
```

Takes ~15-20 min. Confirm:

```bash
kubectl get nodes
```

---

## 2. Storage — EBS CSI driver addon

Needed so `k8s/eks/06-postgres-statefulset.yml`'s PVC (via `ebs-gp3`
StorageClass) can dynamically provision a real EBS volume.

```bash
eksctl utils associate-iam-oidc-provider --cluster devboard-eks --region ap-south-1 --approve

eksctl create iamserviceaccount \
  --cluster devboard-eks --region ap-south-1 --name ebs-csi-controller-sa --namespace kube-system \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve --role-only --role-name AmazonEKS_EBS_CSI_DriverRole

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

eksctl create addon --cluster devboard-eks --region ap-south-1 --name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::${ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole
```

---

## 3. AWS Load Balancer Controller

Provisions the real AWS NLB behind Envoy Gateway's `Service` (see
`k8s/eks/13-envoyproxy.yml`, which sets `type: LoadBalancer` + the
`service.beta.kubernetes.io/aws-load-balancer-*` annotations this controller
reads).

```bash
curl -o /tmp/iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file:///tmp/iam-policy.json

POLICY_ARN="arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy"

eksctl create iamserviceaccount \
  --cluster devboard-eks --region ap-south-1 --namespace kube-system --name aws-load-balancer-controller \
  --attach-policy-arn ${POLICY_ARN} \
  --approve

helm repo add eks https://aws.github.io/eks-charts && helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system --set clusterName=devboard-eks --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller --set region=ap-south-1
```

> **Gotcha hit on this run:** `aws iam create-policy --policy-name
> AWSLoadBalancerControllerIAMPolicy ...` failed with `EntityAlreadyExists`
> — an older version of this policy already existed in the account from a
> prior, unrelated setup. Reusing it, the NLB never provisioned; the
> controller logged `AccessDenied ... not authorized to perform:
> elasticloadbalancing:DescribeListenerAttributes`. The freshly-downloaded
> `iam-policy.json` *does* grant that action — the account's existing v1
> policy just predates it. Fix: add a new policy version from the current
> JSON and set it default, then restart the controller so it picks up fresh
> permissions:
> ```bash
> aws iam create-policy-version --policy-arn $POLICY_ARN \
>   --policy-document file:///tmp/iam-policy.json --set-as-default
> kubectl -n kube-system rollout restart deploy/aws-load-balancer-controller
> ```
> If you're doing this fresh (no pre-existing policy in your account),
> `create-policy` will just succeed and you won't hit this.

---

## 4. Gateway API CRDs + Envoy Gateway

- Gateway API pinned to **v1.6.1** — current stable release on the
  `standard` channel (the Kind doc just says "v1 standard channel"; this
  resolves it to the actual latest tag).
- Envoy Gateway stays at **v1.9.1** — the exact version `docs/kubernetes.md`
  already pins for this project.

> **Gotcha hit on this run:** the `eg` helm chart *bundles and installs the
> Gateway API CRDs itself* (server-side apply). If you `kubectl apply` the
> standard-install.yaml CRDs first (as you'd naturally do, reading this
> top-to-bottom) and *then* `helm install eg`, the install fails with field
> manager conflicts (`conflict with "kubectl-client-side-apply"`). The
> `eg` v1.9.1 chart happens to bundle the exact same v1.6.1 CRDs, so there's
> no real version mismatch here — just two different appliers fighting over
> the same objects. **Skip the standalone `kubectl apply` entirely and let
> the helm chart install the CRDs** — that's what the commands below now do.

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.9.1 \
  -n envoy-gateway-system --create-namespace
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available

# confirm the bundled CRDs are the v1.6.1 you expect:
kubectl get crd gatewayclasses.gateway.networking.k8s.io \
  -o jsonpath='{.metadata.annotations.gateway\.networking\.k8s\.io/bundle-version}{"\n"}'
```

---

## 5. Install ArgoCD

> **Gotcha hit on this run:** a plain `kubectl apply -f install.yaml` fails
> on the `applicationsets.argoproj.io` CRD with
> `metadata.annotations: Too long: may not be more than 262144 bytes` — the
> CRD is too large for client-side-apply's "last applied config" annotation.
> Fix: use `--server-side --force-conflicts` instead (server-side apply has
> no such annotation-size limit).

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side --force-conflicts
kubectl -n argocd rollout status deploy/argocd-server

# initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# expose the UI (simplest for a teaching setup — Ctrl+C to stop)
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Open **https://localhost:8080** — user `admin`, password from above.

---

## 6. Point ArgoCD at this repo

Apply this directly (it's cluster bootstrapping, not app state, so it isn't
one of the files in `k8s/eks/`):

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: devboard-eks
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/LondheShubham153/devboard-starter.git
    targetRevision: main
    path: k8s/eks
  destination:
    server: https://kubernetes.default.svc
    namespace: devboard-ns
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

ArgoCD will now clone the repo, apply everything under `k8s/eks/`, and keep
re-syncing on every push to `main` (and self-heal if anything drifts).

---

## 7. Verify

```bash
kubectl get pods -n devboard-ns
kubectl get pvc -n devboard-ns                # postgres PVC should be Bound (real EBS volume)
kubectl get gateway,httproute -n devboard-ns   # Gateway ADDRESS = the NLB hostname, PROGRAMMED=True
kubectl get svc -n envoy-gateway-system        # the "envoy-<ns>-<gateway>-..." Service, EXTERNAL-IP = same hostname

NLB=$(kubectl get gateway devboard-gateway -n devboard-ns -o jsonpath='{.status.addresses[0].value}')
curl http://$NLB/
curl "http://$NLB/api/tasks?project_id=1"
```

Also check the ArgoCD UI (or `kubectl get application devboard-eks -n argocd
-o jsonpath='{.status.sync.status} {.status.health.status}'`) — should read
**Synced Healthy**. Push a small change to `k8s/eks/` on `main` and watch it
auto-sync — that's the GitOps loop working, not just a one-time apply.

**If `curl` to the NLB hangs or times out**, check target health directly —
this is where the IAM permission gotcha above actually shows up:
```bash
kubectl logs -n kube-system deploy/aws-load-balancer-controller --tail=50 | grep -i error
aws elbv2 describe-target-groups --query 'TargetGroups[?contains(LoadBalancerArns[0], `k8s-envoygat`)].TargetGroupArn' --output text | \
  xargs -I{} aws elbv2 describe-target-health --target-group-arn {}
```

### Example of a working run

```
$ kubectl get pods,pvc,svc,gateway,httproute -n devboard-ns
pod/backend-deployment-...    1/1   Running
pod/frontend-deployment-...   1/1   Running
pod/postgres-0                1/1   Running
persistentvolumeclaim/pgdata-postgres-0   Bound   ebs-gp3   1Gi
gateway/devboard-gateway   envoy-gateway   <NLB hostname>   PROGRAMMED=True
httproute/devboard-route

$ curl http://<NLB hostname>/api/tasks?project_id=1
{"source":"database","tasks":[{"id":1,"title":"Design the task schema",...
```
10 seeded tasks returned — full frontend → backend → postgres chain
confirmed over the real AWS NLB, same golden path as Kind.
`kubectl get application devboard-eks -n argocd` → `Synced Healthy`.

(You'll likely see the backend pod restart 2-3 times early on — it's
retrying its Postgres connection while `postgres-0` is still starting.
Nothing to fix; it self-heals once Postgres is ready, same as Kind.)

---

## 8. Teardown

```bash
helm uninstall aws-load-balancer-controller -n kube-system
helm uninstall eg -n envoy-gateway-system
kubectl delete namespace argocd
eksctl delete cluster --name devboard-eks
```

(`eksctl delete cluster` also removes the nodegroup, VPC, and — since the
`ebs-gp3` StorageClass here uses `reclaimPolicy: Delete` — the dynamically
provisioned EBS volume backing Postgres. No manual EBS cleanup needed.)

**Not removed by the above** — `AWSLoadBalancerControllerIAMPolicy` is a
standalone IAM policy, not part of any eksctl-managed CloudFormation stack,
so it survives cluster teardown. That's arguably fine (reusable next time
you spin this cluster back up), but if you want it fully gone:
```bash
aws iam list-entities-for-policy --policy-arn $POLICY_ARN   # confirm nothing else is attached first
aws iam delete-policy --policy-arn $POLICY_ARN
```
