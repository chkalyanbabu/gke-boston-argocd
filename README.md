# GKE + Ingress (external HTTP LB) + "Boston" pod, deployed via Argo CD

## Repo layout

```
apps/boston/          # what Argo CD syncs into the cluster
  namespace.yaml
  configmap.yaml      # the Boston index.html
  deployment.yaml     # nginx x2, readiness probe = LB health check
  service.yaml        # ClusterIP + NEG annotation (container-native LB)
  backendconfig.yaml  # health check / timeout tuning for the GCLB backend
  ingress.yaml        # kubernetes.io/ingress.class: gce  -> external ALB
argocd/
  boston-application.yaml   # the Argo CD Application (apply with kubectl, not synced)
```

## Quick run-through

```bash
# 0. vars
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export ZONE=us-central1-a
export CLUSTER=boston-cluster

gcloud services enable container.googleapis.com compute.googleapis.com

# 1. Standard cluster
gcloud container clusters create $CLUSTER \
  --zone $ZONE --num-nodes 2 --machine-type e2-medium \
  --enable-ip-alias --release-channel regular
gcloud container clusters get-credentials $CLUSTER --zone $ZONE

# 2. static IP for the LB
gcloud compute addresses create boston-ip --global
gcloud compute addresses describe boston-ip --global --format='value(address)'

# 3. Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.2/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
kubectl port-forward -n argocd svc/argocd-server 8080:443   # https://localhost:8080

# 4. point the Application at your fork, then
kubectl apply -f argocd/boston-application.yaml
argocd app sync boston      # or let auto-sync do it

# 5. watch the LB come up (5-10 min)
kubectl -n boston get ingress boston-ingress -w
curl http://$(gcloud compute addresses describe boston-ip --global --format='value(address)')
```

## Cleanup

```bash
kubectl delete -f argocd/boston-application.yaml   # finalizer prunes the LB first
gcloud container clusters delete $CLUSTER --zone $ZONE
gcloud compute addresses delete boston-ip --global
```
