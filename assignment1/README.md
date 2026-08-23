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

## Before you run this

I could not compile or run this code myself — this sandbox has no access to
the Go module proxy (`proxy.golang.org`), so `go mod tidy` fails here. Run
these steps on your own laptop where you have normal internet access.

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

Ask me when you're ready to do this — I'll generate the Dockerfile and RBAC
YAML for you. It's genuinely useful groundwork for Assignment 3+ where your
controller will need its own RBAC anyway.

## 5. Write-up (the assignment explicitly asks for this)

Answer these in your own words, a few sentences each, and drop them in this
README or your PR description:

- What's the biggest ergonomic difference you personally felt between the
  two libraries while writing the same CRUD logic twice?
- What did `scheme.Scheme` actually do — what breaks if you don't register
  a type with it?
- Read one blog/doc page about controller-runtime's caching client and
  informers. Summarize in 2-3 sentences, in your own words, what problem
  the cache solves that plain client-go doesn't.
- Any utility function or helper (from either library) that surprised you
  or saved you time?

## Commit message template

```
assignment1: client-go and controller-runtime CRUD on ConfigMap/Pod/Deployment

- implemented CRUD for 3 resources using client-go typed clientset
- implemented same CRUD using controller-runtime's unified client.Client
- verified against local kind cluster
- write-up: <one-line summary of your key takeaway>
```
