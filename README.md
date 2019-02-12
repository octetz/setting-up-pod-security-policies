# Setting Up Pod Security Policies

Kubernetes, by default, allows anything capable of creating a Pod to run a fairly privileged container that can compromise a system. Pod Security Policies protect clusters from privileged pods by ensuring the requester is authorized to create a pod as configured.

<iframe width="1206" height="678" src="https://www.youtube.com/embed/dhPy3lWWhoU" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## How It Works

A PodSecurityPolicy is a Kubernetes API object. You can create them without any modifications to Kubernetes. However, the policies created are not enforced by default. Enforcement requires an admission controller, kube-controller-manager modification, and RBAC additions. To demonstrate how this works, we'll add each piece sequentially.

## Admission Controller

Admission controllers intercept requests to the `kube-apiserver`. The interception occurs before a requested object is persisted but after the request is authenticated and authorized. This enables us to see who or what the requested object came from and validate whether what's being asking for is appropriate. Admission controllers are enabled by adding them to the `kube-apiserver` flag `--enable-admission-plugins`. Prior to 1.10, the order of admission controllers mattered when using the, now deprecated, `--admission-control` flag.

Add the `PodSecurityPolicy` to the `--enabled-admission-plugins` flag on the `kube-apiserver`.

```
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,PodSecurityPolicy
```

The other plugins come from the [list of recommended plugins in the Kubernetes documentation](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#is-there-a-recommended-set-of-admission-controllers-to-use). `PodSecurityPolicy` has been appended to that list above. Now that Pod Security Policies are enforced and our cluster is absent of policies, new pod creations (including re-creating pods from a scheduling event) **will fail**.

Create an nginx deployment.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.4
```

Check the available pods, replicasets, and deployments in the namespace. Then delete the deployment.

```
kubectl get po,rs,deploy

NAME                                                            DESIRED   CURRENT   READY   AGE
replicaset.extensions/nginx-hostnetwork-deployment-811c4cff45   1         0         0       9s

NAME                                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/nginx-hostnetwork-deployment   0/1     0            0           9s
```

```
kubectl delete deploy nginx-deployment
```

This demonstrates that the deployment and replicaset were created but the pod could not be created by the replicaset controller. This is where service accounts come in.

## Service Accounts: Controller Manager 

Pods are rarely created by users. Typically a user creates a Deployment, StatefulSet, Job, or Daemonset. This in turn relies on a controller to create the pod. With this in mind, the `kube-controller-manager` should be configured to use individual service accounts for each controller it contains. 

This can be accomplished by adding the following flag to the command args. 

```
--use-service-account-credentials=true
```

This flag is the default for most installers and tooling such as [kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm).

When the `kube-controller-manager` starts with this flag, it'll make use of the following service accounts, automatically generated by Kubernetes.

```
kubectl get serviceaccount -n kube-system | egrep -o '[A-Za-z0-9-]+-controller'

attachdetach-controller
certificate-controller
clusterrole-aggregation-controller
cronjob-controller
daemon-set-controller
deployment-controller
disruption-controller
endpoint-controller
job-controller
namespace-controller
node-controller
pv-protection-controller
pvc-protection-controller
replicaset-controller
replication-controller
resourcequota-controller
service-account-controller
service-controller
statefulset-controller
ttl-controller
```

These service accounts provide granularity in specifying what controller can resolve which policies.

## Policies

The `PodSecurityPolicy` object provides a declarative means for expressing what we allow users and service accounts to create within our clusters. See the [Policy Reference](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#policy-reference) for documentation on available settings. In this example, we'll create 2 policies. The first is a "default" policy that provides restrictive access. Ensuring pods can't be created with privileged settings such as using `hostNetwork`. The second is an "elevated" permissive policy that allows for privileged settings to be used for certain pods, such as those created in the `kube-system` namespace.

Start by creating a restrictive policy that will act as the default.

```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restrictive
spec:
  privileged: false
  hostNetwork: false
  allowPrivilegeEscalation: false
  defaultAllowPrivilegeEscalation: false
  hostPID: false
  hostIPC: false
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  volumes:
  - 'configMap'
  - 'downwardAPI'
  - 'emptyDir'
  - 'persistentVolumeClaim'
  - 'secret'
  - 'projected'
  allowedCapabilities:
  - '*'
```

While restrictive access is adequate for most pod creations, a `permissive` policy is required for pods that require elevated access. For example, `kube-proxy` requires [hostNetwork](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#policy-reference) enabled.

```
kubectl get po kube-proxy-4769h -n kube-system -o yaml | grep 'hostNetwork'

hostNetwork: true
```

Create the permissive policy we'll use for elevated creation rights.

```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: permissive
spec:
  privileged: true
  hostNetwork: true
  hostIPC: true
  hostPID: true
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  hostPorts:
  - min: 0
    max: 65535
  volumes:
  - '*'
```

Now that everything is in place, we need to hook into Kubernetes authorization to determine whether a user or service account requesting pod creation resolves the restrictive or permissive policy. This is where RBAC comes in.

## RBAC

RBAC can be a source of confusion when enabling Pod Security Policies. It determines what policy an account can use. Using a cluster-scoped `ClusterRoleBinding` we can give a service account, such as the `replicaset-controller`, access to the `restrictive` policy.  Using a namespace-scoped RoleBinding, you can enable access to the permissive policy for operating in selective namespaces such as `kube-system`. The diagram below demonstrates a hypothetical resolution path for creating a `kube-proxy` pod on behalf of the `daemonset-controller`.

<center><img src="imgs/rbac-flow.png" width="700"></center>

The flow above exists to help conceptually model policy resolution. Don't expect it to be accurate to the code/execution path.

Start by creating `ClusterRole`s that allow the `use` of the `restrictive` policy and `permissive` policy.

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: psp-restrictive
rules:
- apiGroups:
  - extensions
  resources:
  - podsecuritypolicies
  resourceNames:
  - restrictive
  verbs:
  - use
```

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: psp-permissive
rules:
- apiGroups:
  - extensions
  resources:
  - podsecuritypolicies
  resourceNames:
  - permissive
  verbs:
  - use
```

With the `ClusterRole`s in place, let's start by creating access to use the "default" restrictive policy.

Create a `ClusterRoleBinding` that grants the `psp-restrictive` `ClusterRole` to all controller (system) service accounts.

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: psp-default
subjects:
- kind: Group
  name: system:serviceaccounts
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: psp-restrictive
  apiGroup: rbac.authorization.k8s.io
```

Create the nginx deployment again.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.4
```

Get the pods, replicasets, and deployments for the namespace.

```
kubectl get po,rs,deploy

pod/nginx-hostnetwork-deployment-7c74c7d654-tl4v4   1/1     Running   0          3s

NAME                                                            DESIRED   CURRENT   READY   AGE
replicaset.extensions/nginx-hostnetwork-deployment-7c74c7d654   1         1         1       3s

NAME                                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/nginx-hostnetwork-deployment   1/1     1            1           3s
```

Pods can be created again! However, if we attempt to do something the policy doesn't allow, it should be rejected.

Delete the nginx deployment.

```
kubectl delete deploy nginx-deployment
```

Create the nginx-deployment with `hostNetwork: true`.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-hostnetwork-deployment
  namespace: default
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.4
      hostNetwork: true
```

Get the pods, replicasets, and deployments for the namespace.

```
kubectl get po,rs,deploy

NAME                                                            DESIRED   CURRENT   READY   AGE
replicaset.extensions/nginx-hostnetwork-deployment-597c4cff45   1         0         0       9s

NAME                                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/nginx-hostnetwork-deployment   0/1     0            0           9s
```

Here we can see the pod is no longer able to be created by the replicaset.

Describe the replicaset to better understand why it's unable to create the pod.`

```
Events:
  Type     Reason        Age                 From                   Message
  ----     ------        ----                ----                   -------
  Warning  FailedCreate  35s (x14 over 76s)  replicaset-controller  Error creating: pods "nginx-hostnetwork-deployment-597c4cff45-" is forbidden: unable to validate against any pod security policy: [spec.securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used]

```

In some cases, such as creating pods in `kube-system` we want the ability to use settings such as `hostNetwork: kube-system`. Let's set that up now.

Create a `RoleBinding` for `kube-system` that grants the `psp-permissive` `ClusterRole` to relevant controller service accounts.

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: psp-permissive
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: psp-permissive
subjects:
- kind: ServiceAccount
  name: daemon-set-controller
  namespace: kube-system
- kind: ServiceAccount
  name: replicaset-controller
  namespace: kube-system
- kind: ServiceAccount
  name: job-controller
  namespace: kube-system
```

Now you can create a pod with `hostNetwork: true` in the `kube-system` namespace! An example deployment is as follows.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-hostnetwork-deployment
  namespace: kube-system
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.4
      hostNetwork: true
```

Use `kubectl get pod` to verify this deployment/replicaset pod creation request was admitted.

### RBAC: Admit Based on Workload's Service Account

What if you want to enforce the restrictive policy in a namespace but make an exception for one workload to use the permissive policy? With the current model, we only have cluster-level and namespace-level resolution. To provide workload-level resolution to the permissive policy, we can provide the workload's `ServiceAccount` the ability to `use` the `psp-permissive` `ClusterRole`.

Create a `specialsa` `ServiceAccount` in the `default` namespace.

```
kubectl create serviceaccount specialsa
```

Create a `RoleBinding` in the default namespace binding `specialsa` to the `psp-permissive` `ClusterRole`.

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: specialsa-psp-permissive
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: psp-permissive
subjects:
- kind: ServiceAccount
  name: specialsa
  namespace: default
```

Create the nginx-deployment in the default namespace with the specialsa service account.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-hostnetwork-deployment
  namespace: kube-system
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.4
      hostNetwork: true
	  serviceAccount: specialsa
```

## Troubleshooting / Tricks

### Revealing the Chosen Policy

When the PodSecurityPolicy matches an admission controller to a Pod being requested, it selects that policy and moves on. When debugging, it can be helpful to see which policy was chosen. Kubernetes annotates the pod with the selected PSP so you can see just that.

Search for the psp annotation on the nginx-deployment

```
kubectl get po $(kubectl get po | egrep -o nginx[A-Za-z0-9-]+) -o yaml | grep -i psp

kubernetes.io/psp: permissive
```

## Final Thoughts

I hope this demonstrated how to set up Pod Security Policies in your cluster. **Order of operations matter** When setting up automation to do this work, consider putting your RBAC and Policies in place **before** enabling the `PodSecurityPolicy` admission controller.
