---
layout: post
title: Let Kubernetes Pod create a Deployment (Giving a Pod access to the Kubernetes API)
---

If you are running code inside of a Kubernetes `Pod` which needs to create a Kubernetes `Deployment`, Service, etc., by default, you will get the following error.

```
Unhandled exception. Microsoft.Rest.HttpOperationException: Operation returned an invalid status code 'Forbidden'
   at k8s.Kubernetes.SendRequestRaw(String requestContent, HttpRequestMessage httpRequest, CancellationToken cancellationToken)
   at k8s.Kubernetes.CreateNamespacedDeploymentWithHttpMessagesAsync(V1Deployment body, String namespaceParameter, String dryRun, String fieldManager, String fieldValidation, Nullable`1 pretty, IDictionary`2 customHeaders, CancellationToken cancellationToken)
   at k8s.KubernetesExtensions.CreateNamespacedDeploymentAsync(IKubernetes operations, V1Deployment body, String namespaceParameter, String dryRun, String fieldManager, String fieldValidation, Nullable`1 pretty, CancellationToken cancellationToken)
   at Program.<Main>$(String[] args) in /src/Program.cs:line 68
   at Program.<Main>(String[] args)
```

This is otherwise known as giving a [pod access to the Kubernetes API](https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/#accessing-the-api-from-within-a-pod). This will break down exactly what you need to do to give your pod the right permissions.

See [the examples repo](https://github.com/jkotalik/examples/tree/main/examples/let-pod-create-deployment) for a working example demonstrating how to create a deployment within a pod.

## Step 1: Create a Kubernetes Service Account

See https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/ for more information on Service Accounts.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: manager
  namespace: default
```

## Step 2: Create a Kubernetes Role

This will give permissions for the ClusterRole to create deployments.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: manager-role
rules:
- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
```

## Step 3: Create a Kubernetes Role Binding

See https://kubernetes.io/docs/reference/access-authn-authz/rbac/#clusterrolebinding-example for more info on Role Bindings.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: manager-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: manager-role
subjects:
- kind: ServiceAccount
  name: manager
  namespace: default
```

## Step 4: Modify your Kubernetes Deployment to use the Service Account

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: app
  labels:
    app.kubernetes.io/name: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: app
  template:
    metadata:
      labels:
        app.kubernetes.io/name: app
    spec:
      serviceAccountName: manager # This is the part to modify
      containers:
      - name: hello-world
        image: jkotalik/let-pod-create-deployment:latest
        env:
        - name: DOTNET_LOGGING__CONSOLE__DISABLECOLORS
          value: 'true'
        - name: ASPNETCORE_URLS
          value: 'http://*'
        ports:
        - containerPort: 80
```
