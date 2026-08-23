# Assignment 1 — client-go vs controller-runtime CRUD

Resources used: **ConfigMap**, **Pod**, **Deployment**

## Folder layout

```
assignment1/
  client-go-crud/            # raw client-go version
    go.mod
    main.go
  controller-runtime-crud/   # controller-runtime version
    go.mod
    main.go
  README.md
```

## Before we run

## 1. Make sure your kind cluster is up

```bash
kind get clusters
# if empty:
kind create cluster --name dev-training
kubectl cluster-info --context kind-dev-training
```

## 2. Run the client-go version

```bash
cd assignment1/client-go-crud
go mod tidy          # downloads k8s.io/client-go, k8s.io/api, k8s.io/apimachinery
go run main.go
```

Expected: you'll see `[info] not running in-cluster, falling back to local
kubeconfig`, then a log line for every create/read/update/delete across the
three resources. Watch it happen live in another terminal:

```bash
kubectl get configmap,pod,deployment -w
```

## 3. Run the controller-runtime version

```bash
cd ../controller-runtime-crud
go mod tidy           # also pulls in sigs.k8s.io/controller-runtime
go run main.go
```

Same behavior, different plumbing. Compare the code side by side — notice
`c.Get(ctx, key, obj)` looks identical no matter which of the three types
`obj` is, whereas client-go needed `CoreV1().ConfigMaps()`,
`CoreV1().Pods()`, `AppsV1().Deployments()` — three different "sub-clients".

## 4. (Stretch) Actually run it in-cluster, as the assignment literally asks

Right now both programs fall back to your laptop's kubeconfig. To satisfy
"using in-cluster config" for real:

1. Build a container image with this Go binary (`Dockerfile` with
   `FROM golang:1.22 AS build` → `FROM gcr.io/distroless/static` pattern).
2. `kind load docker-image <your-image> --name dev-training` (kind can't
   pull from your local Docker daemon otherwise).
3. Create a ServiceAccount + RBAC Role/RoleBinding granting it
   get/list/create/update/delete on configmaps, pods, deployments.
4. Run it as a one-shot Pod using that ServiceAccount. Inside the Pod,
   `rest.InClusterConfig()` will succeed instead of falling through.
