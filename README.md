# Multicluster-Service-Account

Multicluster-service-account makes it easy for pods in a cluster to call the Kubernetes APIs of other clusters. It imports remote service account tokens into local secrets, and automounts them inside annotated pods.

Multicluster-service-account can be used to run any Kubernetes client from another cluster. It can also be used to build operators that control Kubernetes resources across multiple clusters, e.g., with [multicluster-controller](https://github.com/admiraltyio/multicluster-controller).

Why? Check out [Admiralty's blog post introducing multicluster-service-account](https://admiralty.io/blog/introducing-multicluster-service-account).

## How it Works

Multicluster-service-account consists of:

1. A binary, `kubemcsa`, to bootstrap clusters, allowing them to import service account secrets from one another;
    - After [installing](#step-1-installation) multicluster-service-account in cluster1, allowing cluster1 to import service account secrets from cluster2 is as simple as running
        ```sh
        kubemcsa bootstrap --target-context cluster1 --source-context cluster2
        ```
      if you work with multiple contexts in a single default kubeconfig file, or
        ```sh
        kubemcsa bootstrap --target-kubeconfig cluster1 --source-kubeconfig cluster2
        ```
      if you work with multiple kubeconfig files.
1. a ServiceAccountImport custom resource definition (CRD) and controller to import remote service account secrets;
    - Here is a sample service account import object:
        ```yaml
        apiVersion: multicluster.admiralty.io/v1alpha1
        kind: ServiceAccountImport
        metadata:
          name: cluster2-default-pod-lister
        spec:
          clusterName: cluster2
          namespace: default # source and target namespaces can be different
          name: pod-lister
        ```
    - which would generate a secret like this:
        ```yaml
        apiVersion: v1
        kind: Secret
        metadata:
          name: cluster2-default-pod-lister-token-6456p
          ... # owner reference, etc.
        type: Opaque
        data:
          ca.crt: ...
          namespace: ...
          token: ...
          server: ...
          config: ...
        ```
      `ca.crt`, `namespace` and `token` are copied from the source service account secret. `server` is added to locate the remote Kubernetes API (FYI, regular service accounts simply call their namespace's `kubernetes` Service). Those four fields contain enough information to call a remote Kubernetes API, but require custom code to be loaded as a Kubernetes client configuration (cf. 4., below). Therefore, a fifth field is provided as a convenience: the `config` field combines the four first fields as a standard [kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) file, to make it even easier to use multicluster-service-account, without any code change, with clients written in any language.
1. a dynamic admission webhook to automount service account import secrets inside annotated pods, the same way regular service accounts are automounted inside Pods;
    - To automount the sample secret from above inside a pod, you would annotate the pod with `multicluster.admiralty.io/service-account-import.name=cluster2-default-pod-lister`. To mount multiple service account imports inside a single pod, append their names to the pod annotation, separated by commas. The pod and service account imports must be in the same namespace, and that namespace must itself be annotated with `multicluster-service-account=enabled`.
    - The sample secret would be mounted at `/var/run/secrets/admiralty.io/serviceaccountimports/cluster2-default-pod-lister`. Most Kubernetes clients accept a `--kubeconfig` option or a `KUBECONFIG` environment variable, which you would set to `/var/run/secrets/admiralty.io/serviceaccountimports/cluster2-default-pod-lister/config`.
1. **(optional)** Go helper functions (in the `pkg/config` package) to list and load mounted service account imports. They are particularly useful when you need to load multiple service account imports, when you need the remote namespaces as well, and/or when you'd like to automatically fallback to the default kubeconfig or local service account.

## Getting Started

We assume that you are a cluster admin on two clusters, associated with, e.g., the contexts "context1" and "context2" in your kubeconfig, and the clusters "cluster1" and "cluster2", respectively. We're going to install multicluster-service-account and run a multi-cluster client example in cluster1, listing pods in cluster2.

```bash
CONTEXT1=context1 # change me
CLUSTER1=cluster1 # change me

CONTEXT2=context2 # change me
CLUSTER2=cluster2 # change me
```

Note: you can also use multiple kubeconfig files, see below.

### Step 1: Installation

Install multicluster-service-account in cluster1:

```bash
RELEASE_URL=https://github.com/admiraltyio/multicluster-service-account/releases/download/v0.5.1
MANIFEST_URL=$RELEASE_URL/install.yaml
kubectl apply -f $MANIFEST_URL --context $CONTEXT1
```

Cluster1 is now able to import service accounts, but it hasn't been given permission to import them from cluster2 yet. This is a chicken-and-egg problem: cluster1 needs a token from cluster2, before it can import service accounts from it. To solve this problem, download the kubemcsa binary and run the bootstrap command:

```bash
OS=linux # or darwin (i.e., OS X) or windows
ARCH=amd64 # if you're on a different platform, you must know how to build from source
BINARY_URL="$RELEASE_URL/kubemcsa-$OS-$ARCH"
curl -Lo kubemcsa $BINARY_URL
chmod +x kubemcsa
sudo mv kubemcsa /usr/local/bin

kubemcsa bootstrap --target-context $CONTEXT1 --source-context $CONTEXT2
```

Note: if you access your clusters using different kubeconfig files, you can bootstrap them with the `--target-kubeconfig`/`--source-kubeconfig` options. Also, if you don't like your kubeconfig cluster names (or if they aren't valid DNS-1123 subdomains), you can rename them, as far as multicluster-service-account is concerned, with the `--target-name`/`--source-name` options.

### Step 2: Example

The `multicluster-client` example includes:

- in cluster2:
  - a service account named `pod-lister` in the default namespace, bound to a role that can only list pods in its namespace;
  - a dummy NGINX deployment (to have pods to list);
- in cluster1:
  - a new label on the `default` namespace, `multicluster-service-account=enabled`, to instruct multicluster-service-account to automount service account import secrets inside annotated pods;
  - a service account import named `cluster2-default-pod-lister`, importing `pod-lister` from the default namespace of cluster2;
  - a `multicluster-client` job, whose pod is annotated to automount `cluster2-default-pod-lister`'s secret—it will list the pods in the default namespace of cluster2, and stop without restarting (we'll check the logs).

```bash
kubectl config use-context $CONTEXT2
kubectl create serviceaccount pod-lister
kubectl create role pod-lister --verb=list --resource=pods
kubectl create rolebinding pod-lister --role=pod-lister \
  --serviceaccount=default:pod-lister
kubectl run nginx --image nginx

kubectl config use-context $CONTEXT1
kubectl label namespace default multicluster-service-account=enabled
cat <<EOF | kubectl create -f -
apiVersion: multicluster.admiralty.io/v1alpha1
kind: ServiceAccountImport
metadata:
  name: $CLUSTER2-default-pod-lister
spec:
  clusterName: $CLUSTER2
  namespace: default
  name: pod-lister
---
apiVersion: batch/v1
kind: Job
metadata:
  name: multicluster-client
spec:
  template:
    metadata:
      annotations:
        multicluster.admiralty.io/service-account-import.name: $CLUSTER2-default-pod-lister
    spec:
      restartPolicy: Never
      containers:
      - name: multicluster-client
        image: multicluster-service-account-example-multicluster-client:latest
EOF
```

In cluster1, check that:

1. The service account import controller created a secret for the `cluster2-default-pod-lister` service account import, containing a kubeconfig file populated with the token and namespace of the remote service account, and the URL and root certificate of the remote Kubernetes API:
    ```bash
    kubectl get secret -l multicluster.admiralty.io/service-account-import.name=$CLUSTER2-default-pod-lister -o jsonpath={.items[0].data.config} | base64 -D
    # the data is base64-encoded
    ```
1. The service account import secret was mounted inside the `multicluster-client` pod by the service account import admission controller:
    ```bash
    kubectl get pod -l job-name=multicluster-client -o yaml
    # look at volumes and volume mounts
    ```
1. The `multicluster-client` pod was able to list pods in the default namespace of cluster2:
    ```bash
    kubectl logs job/multicluster-client
    ```

## Service Account Imports

Service account imports tell the service account import controller to maintain a secret in the same namespace, containing the remote service account's namespace and token, as well as the URL and root certificate of the remote Kubernetes API, which are all necessary data to configure a Kubernetes client. The secret is also conveniently formatted as a standard [kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) file, which most Kubernetes clients can understand. If a pod needs to call several clusters, it will use several service account imports, e.g.:

```yaml
apiVersion: multicluster.admiralty.io/v1alpha1
kind: ServiceAccountImport
metadata:
  name: cluster2-default-pod-lister
spec:
  clusterName: cluster2
  namespace: default
  name: pod-lister
---
apiVersion: multicluster.admiralty.io/v1alpha1
kind: ServiceAccountImport
metadata:
  name: cluster3-default-pod-lister
spec:
  clusterName: cluster3
  namespace: default
  name: pod-lister
```

## Annotations

In namespaces labeled with `multicluster-service-account=enabled`, the `multicluster.admiralty.io/service-account-import.name` annotation on a pod (or pod template) tells the service account import admission controller to automount the corresponding secrets inside it. If a pod needs several service account imports, separate their names with commas, e.g.:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multicluster-client
  annotations:
    multicluster.admiralty.io/service-account-import.name: cluster2-default-pod-lister,cluster3-default-pod-lister
spec:
  # ...
```

Note: just like with local service accounts, there is a race condition if a service account import and a pod requesting it are created at the same time: the service account import admission controller will likely reject the pod because the secret to automount won't be ready. Luckily, if the pod is controlled by another object, such as a deployment, job, etc., pod creation will be retried.

## (Optional) Client Configuration

Multicluster-service-account includes a Go library (cf. [`pkg/config`](pkg/config)) to facilitate the creation of [client-go `rest.Config`](https://godoc.org/k8s.io/client-go/rest#Config) instances from service account imports. From there, you can create [`kubernetes.Clientset`](https://godoc.org/k8s.io/client-go/kubernetes#NewForConfig) instances as usual. The namespaces of the remote service accounts are also provided:

```go
cfg, ns, err := NamedServiceAccountImportConfigAndNamespace("cluster2-default-pod-lister")
// ...
clientset, err := kubernetes.NewForConfig(cfg)
// ...
pods, err := clientset.CoreV1().Pods(ns).List(metav1.ListOptions{})
```

Usually, however, you don't want to hardcode the name of the mounted service account import. If you only expect one, you can get a Config for it and its remote namespace like this:

```go
cfg, ns, err := ServiceAccountImportConfigAndNamespace()
```

If several service account imports are mounted, you can get Configs and namespaces for all of them by name as a `map[string]ConfigAndNamespaceTuple`:

```go
all, err := AllServiceAccountImportConfigsAndNamespaces()
// ...
for name, cfgAndNs := range all {
  cfg := cfgAndNs.Config
  ns := cfgAndNs.Namespace
  // ...
}
```

### Generic Client Configuration

The true power of multicluster-service-account's `config` package is in its generic functions, that can fall back to kubeconfig contexts or regular service accounts when no service account import is mounted:

```go
cfg, ns, err := ConfigAndNamespace()
```

```go
all, err := AllNamedConfigsAndNamespaces()
```

The service account import controller uses `AllNamedConfigsAndNamespaces()` internally. The [generic client example](examples/generic-client/) uses `ConfigAndNamespace()`.

## API Reference

For more details on the `config` package, or to better understand how the service account import controller and admission control work, please refer to the API documentation:

https://godoc.org/admiralty.io/multicluster-service-account/

or

```bash
go get admiralty.io/multicluster-service-account
godoc -http=:6060
```

then http://localhost:6060/pkg/admiralty.io/multicluster-service-account/
