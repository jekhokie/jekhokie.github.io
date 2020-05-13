---
layout: post
title:  "Open Policy Agent as k8s Admission Controller"
date:   2020-05-08 20:12:00 -0400
categories: k8s opa admission-controller
---

[Open Policy Agent](https://openpolicyagent.org/) (OPA) enables multi-platform policy control of resources. There are many use cases and
applications for OPA (Terraform, Envoy, Kubernetes, etc.) but this post will be focused on enabling OPA as an admission controller to define
allowed/denied policies related to resource requests within Kubernetes. There is a more first-class integration between OPA and Kubernetes
known as [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper), which is out of scope for this particular tutorial, which will
focus on native integration between OPA and k8s.

### Prerequisites

This tutorial assumes you've followed [this]({% post_url 2020-04-21-vagrant-k8s-cluster %}) previous post and have a local
k8s cluster set up using Vagrant (or any k8s cluster that is reachable from your workstation - but the instructions are
geared towards the existing setup referenced in the previous post).

### Validating Required Admission Controllers

The OPA integration expects that the `ValidatingAdmissionWebhook` is enabled on your cluster. This Admission Webhook is normally enabled
by default on k8s clusters, but to validate this, you can inspect the command-line arguments present from the `kube-apiserver-master` pod
(assuming you followed the previously-referenced tutorial for creating the cluster):

```bash
$ kubectl describe pod kube-apiserver-master -n kube-system
# should include output indicating ValidatingAdmissionWebhook is enabled
```

If the above does not show `ValidatingAdmissionWebhook` as enabled in the `--enable-admission-plugins=` switch, you can enable it by
editing the `/etc/kubernetes/manifests/kube-apiserver.yaml` file and adding it to the list of enabled plugins, and then deleting the
API Server pod via `kubectl delete pod kube-apiserver-master` (at which point the pod will re-generate and use the new command-line
arguments in the configuration file).

### Deploying OPA

Deploying OPA requires a few steps, which will be covered in this section:

1. A k8s namespace for logical organization of the OPA resources.
2. TLS is required for communication between k8s and OPA - generate and store the required components as a Secret resource in k8s
which is accessible by OPA.
3. Deployment of OPA with an associated `ConfigMap` resource to configure it readies the functionality.
4. Disable OPA for `kube-system` and `opa` namespaces by labeling the namespaces.
5. Enabling OPA as an Admission Controller.

If you wish to simply execute the required steps to get OPA up and running, you can simply clone and use the contents in
[this repository folder](https://github.com/jekhokie/scriptbox/tree/master/multiple--vagrant-istio-k8s-cluster/shared/opa) by following
these steps and then skipping all sub-headings in this section:

```bash
$ cd shared/opa/

# create opa namespace
$ kubectl create namespace opa

# generate CA
$ openssl genrsa -out ssl/ca.key 2048
$ openssl req -x509 -new -nodes \
              -key ssl/ca.key \
              -days 365 \
              -out ssl/ca.crt \
              -subj "/CN=admission_ca"

# generate certificate/key pair for opa
$ openssl genrsa -out ssl/server.key 2048
$ openssl req -new -key ssl/server.key \
              -out ssl/server.csr \
              -subj "/CN=opa.opa.svc" \
              -config req.cnf
$ openssl x509 -req -in ssl/server.csr \
               -CA ssl/ca.crt \
               -CAkey ssl/ca.key \
               -CAcreateserial \
               -out ssl/server.crt \
               -days 365 \
               -extensions v3_req \
               -extfile req.cnf

# store the certificate/key pair for opa as a secret
$ kubectl create secret tls opa-server \
                 --cert=ssl/server.crt \
                 --key=ssl/server.key \
                 --namespace=opa

# deploy opa
$ kubectl apply -f opa.yaml --namespace=opa

# label kube-system and opa namespaces to exclude OPA management
$ kubectl label namespace kube-system openpolicyagent.org/webhook=ignore
$ kubectl label namespace opa openpolicyagent.org/webhook=ignore

# edit the opa-webhook.yaml to ensure the base64-encoded version of the
# ca.crt is within scope for the config
$ vim opa-webhook.yaml

# register opa as admission controller
$ kubectl apply -f opa-webhook.yaml --namespace=opa

# watch webhook requests
$ kubectl logs -l app=opa -c opa --namespace=opa -f

# attempt to launch an image from a repo that is not whitelisted
$ kubectl run busybox --image=busybox --image-pull-policy=Always

# error should return, indicating the policy is enforcing
```

If you wish to understand what the commands above are doing, proceed to the following sub-sections.

#### Create OPA Namespace

To deploy OPA into k8s, it's best to deploy into its own namespace - create a namespace for OPA: `kubectl create namespace opa`

#### Generate TLS Components

As stated, OPA requires TLS-based communication with k8s. We'll generate the components required for TLS communication first:

```bash
# generate CA
$ openssl genrsa -out ca.key 2048
$ openssl req -x509 -new -nodes -key ca.key -days 365 -out ca.crt -subj "/CN=admission_ca"

# generate certificate/key pair for opa
$ cat >server.conf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, serverAuth
EOF

$ openssl genrsa -out server.key 2048
$ openssl req -new -key server.key \
              -out server.csr \
              -subj "/CN=opa.opa.svc" \
              -config server.conf
$ openssl x509 -req -in server.csr \
               -CA ca.crt \
               -CAkey ca.key \
               -CAcreateserial \
               -out server.crt \
               -days 365 \
               -extensions v3_req \
               -extfile server.conf
```

Next, store the components in a k8s Secret for OPA to access:

```bash
$ kubectl create secret tls opa-server \
                 --cert=server.crt \
                 --key=server.key \
                 --namespace=opa
```

#### Deploy OPA and ConfigMap

Deployment of OPA takes several resources, including various role-based permissions for access. Create a file `opa.yaml` with
the following contents. Note that the Service created is one of `type: NodePort` in order to easily enable us to access the service since we do
not have an actual ingress defined as part of our cluster:

```yaml
# Grant OPA/kube-mgmt read-only access to resources. This lets kube-mgmt
# replicate resources into OPA so they can be used in policies.
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: opa-viewer
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: Group
  name: system:serviceaccounts:opa
  apiGroup: rbac.authorization.k8s.io
---
# Define role for OPA/kube-mgmt to update configmaps with policy status.
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: opa
  name: configmap-modifier
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["update", "patch"]
---
# Grant OPA/kube-mgmt role defined above.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: opa
  name: opa-configmap-modifier
roleRef:
  kind: Role
  name: configmap-modifier
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: Group
  name: system:serviceaccounts:opa
  apiGroup: rbac.authorization.k8s.io
---
kind: Service
apiVersion: v1
metadata:
  name: opa
  namespace: opa
spec:
  selector:
    app: opa
  type: NodePort
  ports:
  - name: https
    protocol: TCP
    port: 443
    targetPort: 443
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: opa
  namespace: opa
  name: opa
spec:
  replicas: 1
  selector:
    matchLabels:
      app: opa
  template:
    metadata:
      labels:
        app: opa
      name: opa
    spec:
      containers:
        # WARNING: OPA is NOT running with an authorization policy configured. This
        # means that clients can read and write policies in OPA. If you are
        # deploying OPA in an insecure environment, be sure to configure
        # authentication and authorization on the daemon. See the Security page for
        # details: https://www.openpolicyagent.org/docs/security.html.
        - name: opa
          image: openpolicyagent/opa:latest
          args:
            - "run"
            - "--server"
            - "--tls-cert-file=/certs/tls.crt"
            - "--tls-private-key-file=/certs/tls.key"
            - "--addr=0.0.0.0:443"
            - "--addr=http://127.0.0.1:8181"
            - "--log-format=json-pretty"
            - "--set=decision_logs.console=true"
          volumeMounts:
            - readOnly: true
              mountPath: /certs
              name: opa-server
          readinessProbe:
            httpGet:
              path: /health?plugins&bundle
              scheme: HTTPS
              port: 443
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              scheme: HTTPS
              port: 443
            initialDelaySeconds: 3
            periodSeconds: 5
        - name: kube-mgmt
          image: openpolicyagent/kube-mgmt:0.8
          args:
            - "--replicate-cluster=v1/namespaces"
            - "--replicate=extensions/v1beta1/ingresses"
            - "--replicate=apps/v1/deployments"
            - "--replicate=v1/pods"
      volumes:
        - name: opa-server
          secret:
            secretName: opa-server
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: opa-default-system-main
  namespace: opa
  labels:
    app: opa
data:
  main: |
    package system

    import data.kubernetes.admission

    main = {
      "apiVersion": "admission.k8s.io/v1beta1",
      "kind": "AdmissionReview",
      "response": response,
    }

    default response = {"allowed": true}

    response = {
        "allowed": false,
        "status": {
            "reason": reason,
        },
    } {
        reason = concat(", ", admission.deny)
        reason != ""
    }
```

Apply the file resources to deploy OPA via: `kubectl apply -f opa.yaml --namespace=opa`.

#### Excluding OPA from Namespaces

Some system-level namespaces and, in general, the `opa` namespace we created likely need to be excluded from scope. To do this,
use labels on the namespace the way the OPA configuration expects (as defined in the previous sub-section):

```bash
$ kubectl label namespace kube-system openpolicyagent.org/webhook=ignore
$ kubectl label namespace opa openpolicyagent.org/webhook=ignore
```

#### Enable OPA as Admission Controller

Finally, enabling OPA as an admission controller. Create a file `opa-webhook.yaml` with the following contents, replacing the `<REPLACE_WITH_B64_CA_CRT>`
variable with the output of the command `base64 ca.crt | tr -d '\n'`, where `ca.crt` is the certificate authority certificate created earlier:

```yaml
kind: ValidatingWebhookConfiguration
apiVersion: admissionregistration.k8s.io/v1beta1
metadata:
  name: opa-validating-webhook
webhooks:
  - name: validating-webhook.openpolicyagent.org
    namespaceSelector:
      matchExpressions:
      - key: openpolicyagent.org/webhook
        operator: NotIn
        values:
        - ignore
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: ["*"]
        apiVersions: ["*"]
        resources: ["*"]
    clientConfig:
      caBundle: <REPLACE_WITH_B64_CA_CRT>
      service:
        namespace: opa
        name: opa
```

Apply the changes, and you should now be configured with OPA as a valid admission controller: `kubectl apply -f opa-webhook.yaml`.

You can monitor logs for OPA to see what webhook requests/communication is occurring via the following command:

```bash
$ kubectl logs -l app=opa -c opa --namespace=opa -f
```

### Configuring Policy for OPA

As stated previously, OPA expects any policies to be defined using `ConfigMap` resources in the namespace where OPA is running
(for this tutorial, specifically in the `opa` namespace). Let's create a policy that ensures container images deployed into the
k8s cluster are coming from a valid/trusted repository in Google Container Registry (`gcr.io`). This is done by defining a Rego-style
policy. Create a file named `image-repo-check.rego` with the following contents (or you can get this file with its contents straight
out of [this repository folder](https://github.com/jekhokie/scriptbox/tree/master/multiple--vagrant-istio-k8s-cluster/shared/opa)):

```
package kubernetes.admission

deny[msg] {
    input.request.kind.kind == "Pod"
    some i
    image := input.request.object.spec.containers[i].image
    not startswith(image, "gcr.io/")
    msg := sprintf("image '%v' comes from an untrusted registry", [image])
}
```

Next, store this policy as a `ConfigMap` resource in the `opa` namespace:

```bash
$ kubectl create configmap image-repo-check \
                 --from-file=image-repo-check.rego \
                 --namespace=opa
```

### Testing OPA Policy

Now that we have a policy in place to restrict based on container image repository, let's test it. Try to launch the `busybox` image
as a pod to the cluster while at the same time (in a different window) watching the logs for the OPA agent:

```bash
# keep a terminal window open with the following to watch
# logs and inspect that opa responded with deny for launching
# a container image from an uncertified repo
$ kubectl logs -l app=opa -c opa --namespace=opa -f

# open a new terminal window and attempt to deploy an instance
# of the busybox image
$ kubectl run busybox --image=busybox --image-pull-policy=Always
```

If the above worked successfully, you should see an error similar to the following, indicating that our policy is working!

`Error from server (image 'busybox' comes from untrusted registry): admission webhook "validating-webhook.openpolicyagent.org" denied the request: image 'busybox' comes from untrusted registry`

In addition, you should see a response object in the OPA logs similar to the following:

```json
  ...
  "result": {
    "apiVersion": "admission.k8s.io/v1beta1",
    "kind": "AdmissionReview",
    "response": {
      "allowed": false,
      "status": {
        "reason": "image 'busybox' comes from untrusted registry"
      }
    }
  },
  ...
```

### Troubleshooting

A few things can occur when setting up the OPA functionality. To troubleshoot, here are a few things that might occur and how you
can possibly resolve them:

#### OPA Error Reporting Default Policy Missing

Error reported: `document missing or undefined: data.system.main`

To troubleshoot this, inspect the logs in `/var/log/containers/opa-*kube-mgmt*.log` as it is likely there is an issue with the `kube-mgmt`
sidecar communicating with the `kube-apiserver`. If this is the case, it's possible that the firewall on the k8s node is blocking
connectivity. As a quick workaround (not recommended for production use cases), you can simply `sudo systemctl stop firewalld` on the
k8s nodes and the issue should resolve.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [OPA - Kubernetes Overview & Architecture](https://www.openpolicyagent.org/docs/latest/kubernetes-introduction/)
* [OPA - Tutorial: Ingress Validation](https://www.openpolicyagent.org/docs/latest/kubernetes-tutorial/)
