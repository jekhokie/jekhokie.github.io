---
layout: post
title:  "Dex OIDC Auth with LDAP and Gangway for k8s"
date:   2020-04-28 19:56:00 -0400
categories: k8s dex ldap auth gangway oidc
logo: dex.jpg
---

Authentication and self-service for users to interact with k8s is generally fairly common, and enabling this in a very self-service,
automated, and maintainable way can be challenging. This post attempts to provide a jumping off point for enabling self-service provisioning
and management of user-based permissions in interacting with a k8s cluster using [Dex](https://github.com/dexidp/dex) as an OpenID
Connect (OIDC) Identity and [Gangway](https://github.com/heptiolabs/gangway) to enable auth flows using OIDC for a k8s cluster. The use
case for this configuration would be for infrastructure and platform teams to enable development and other teams to self-provision
tokens that enable them to interact with a k8s cluster based on roles and permissions defined by the managing team using k8s RBAC.

### Warning

Note that this tutorial is NOT recommended for a production-grade OIDC setup using Dex. There are many missing security-related
components (client secrets, secrets in general, etc.) - this tutorial is focused heavily on how to use Dex and Gangway as an
academic exercise to provide a jumping off point for more production-grade configurations.

### Prerequisites

This tutorial assumes you've followed [this]({% post_url 2020-04-21-vagrant-k8s-cluster %}) previous post and have a local
k8s cluster set up using Vagrant (or any k8s cluster that is reachable from your workstation - but the instructions are
geared towards the existing setup referenced in the previous post).

### OpenLDAP

OpenLDAP will be used as our LDAP directory service and we will install it right into the k8s cluster for simplicity and ease
of communication with the various other components we will focus on in this post.

#### Install and Configure OpenLDAP

First, let's deploy OpenLDAP to our k8s cluster using the [OpenLDAP Helm Chart](https://github.com/helm/charts/tree/master/stable/openldap)
and a few configuration parameters we wish to use to expose the services easier and set a known password `abc1234` for
the admin and config users:

```bash
$ helm install openldap stable/openldap \
               --set service.type=NodePort \
               --set adminPassword=abcd1234 \
               --set configPassword=abcd1234
```

Once you run the above, it will inform you that you can retrieve the admin and config user passwords using a couple of
commands listed below (in case you forget the passwords you set):

```bash
$ kubectl get secret openldap -o jsonpath="{.data.LDAP_ADMIN_PASSWORD}" | base64 --decode; echo
$ kubectl get secret openldap -o jsonpath="{.data.LDAP_CONFIG_PASSWORD}" | base64 --decode; echo
```

In addition, the output will also give you a way to test the OpenLDAP installation using the `ldapsearch` command. Let's
run a quick test - you'll need to use the IP address of the k8s node to which the service has been deployed, and the NodePort
specified for the service (included in the instructions below):

```bash
# get the node that the particular openldap pod is deployed to
$ kubectl get pods --output=wide
# based on the "NODE" column, determine the IP address of the node to use
# if you are using the previously configured cluster, this is likely one of the following:
#   - master: 10.11.12.13 (this is only possible if you removed the taint to enable master scheduling of pods)
#   - node1:  10.11.12.14
#   - node2:  10.11.12.15

# get the NodePort for the running pod
$ kubectl get service openldap
# capture the second port number after the ':' under PORT(S) - depending on how
# you decided to deploy, you can either use the encryption-enabled endpoint or the
# clear-text endpoint port (636 or 389, respectively) - we will use the clear-text
# port for simplicity

# use the <NODE_IP> and <NODE_PORT> captured to perform a search
$ ldapsearch -x -H ldap://<NODE_IP>:<NODE_PORT> \
                -b dc=example,dc=org \
                -D "cn=admin,dc=example,dc=org" \
                -w $LDAP_ADMIN_PASSWORD
```

If all goes well, you should get an LDIF response similar to the following:

```
# extended LDIF
#
# LDAPv3
# base <dc=example,dc=org> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# example.org
dn: dc=example,dc=org
objectClass: top
objectClass: dcObject
objectClass: organization
o: Example Inc.
dc: example

# admin, example.org
dn: cn=admin,dc=example,dc=org
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: LDAP administrator
userPassword:: e1NTSEF9RzI2UkRjSXNuQzFLemRNSnVVM25WM0dQWlo0YWY3SVM=

# search result
search: 2
result: 0 Success

# numResponses: 3
# numEntries: 2
```

#### Add Groups and Users to OpenLDAP

You're now ready to add some content to the LDAP server to use for authentication in the k8s cluster. Note that if you had built
your cluster with and are referring to [this repository folder](https://github.com/jekhokie/scriptbox/tree/master/multiple--vagrant-istio-k8s-cluster),
you can simply execute the following command sequence and skip the remainder of this section for OpenLDAP. Otherwise (or if you're
interested in what is happening), continue down the path of the remainder of this section.

```bash
# add the test user
$ ldapadd -x -H ldap://<NODE_IP>:<NODE_PORT> \
                -D "cn=admin,dc=example,dc=org" \
                -w $LDAP_ADMIN_PASSWORD \
                -f shared/openldap/config-user.ldif

# add the group
$ ldapadd -x -H ldap://<NODE_IP>:<NODE_PORT> \
                -D "cn=admin,dc=example,dc=org" \
                -w $LDAP_ADMIN_PASSWORD \
                -f shared/openldap/config-group.ldif

# examine results
$ ldapsearch -x -H ldap://<NODE_IP>:<NODE_PORT> \
                -b dc=example,dc=org \
                -D "cn=admin,dc=example,dc=org" \
                -w $LDAP_ADMIN_PASSWORD

# test auth works as expected
$ ldapwhoami -x -H ldap://<NODE_IP>:<NODE_PORT> \
                -D "uid=jsmith,ou=users,dc=example,dc=org" \
                -w abcd1234

# ensure the user has a memberOf response indicating their group
$ ldapsearch -x -H ldap://<NODE_IP>:<NODE_PORT> \
                -b "ou=users,dc=example,dc=org" \
                -D "cn=admin,dc=example,dc=org" \
                -w abcd1234 \
                "(uid=jsmith)" memberOf
```

First, let's add our test user and associate a password that we can use to actually test auth with. To generate a SHA-formatted
password having clear-text of `abcd1234`, use the command `slappasswd -s abcd1234`, which should generate a SHA-formatted output
string. Copy this string, and create a file named `config-user.ldif` with the following contents, replacing <SHA_PASSWORD> with
the string you copied from the result of the `slappasswd` command. Note that this LDIF file will also generate the `ou=users` OU
in the directory structure for gathering/organizing users:

```
# config users
dn: ou=users,dc=example,dc=org
objectClass: top
objectClass: organizationalUnit
ou: users

# config jsmith
dn: uid=jsmith,ou=users,dc=example,dc=org
objectClass: top
objectClass: posixAccount
objectClass: inetOrgPerson
description: Test user
cn: jsmith
sn: Smith
uid: jsmith
userPassword: {SSHA}FB4s71qplmiQZW7Rbqf0+WlPExBjaG8f
uidNumber: 1001
gidNumber: 1001
homeDirectory: /home/jsmith
mail: jsmith@test.com
loginShell: /bin/bash
gecos: John Smith
```

Run the `ldapadd` command to add the `users` OU and the associated user belonging to the group/directory:

```bash
$ ldapadd -x -H ldap://<NODE_IP>:<NODE_PORT> \
                -D "cn=admin,dc=example,dc=org" \
                -w $LDAP_ADMIN_PASSWORD \
                -f config-user.ldif
```

We'll now create a group to which the user belongs. Create another LDIF file named `config-group.ldif` with the following contents,
which will add the `jsmith` test user `uid` to the group:

```
dn: ou=groups,dc=example,dc=org
objectClass: top
objectClass: organizationalUnit
ou: groups

dn: cn=testgrp,ou=groups,dc=example,dc=org
objectClass: groupOfUniqueNames
cn: testgrp
uniqueMember: uid=jsmith,ou=users,dc=example,dc=org
```

Next, add the group to your OpenLDAP directory:

```bash
$ ldapadd -x -H ldap://<NODE_IP>:<NODE_PORT> \
                -D "cn=admin,dc=example,dc=org" \
                -w $LDAP_ADMIN_PASSWORD \
                -f config-group.ldif
```

We should now be able to see our additional `OU`s, the group, and the user we created via running the previous `ldapsearch` command:

```bash
$ ldapsearch -x -H ldap://<NODE_IP>:<NODE_PORT> \
                -b dc=example,dc=org \
                -D "cn=admin,dc=example,dc=org" \
                -w $LDAP_ADMIN_PASSWORD
```

Finally, we'll test that the user password works as expected by attempting to bind to the LDAP directory using the user and associated
credential:

```bash
$ ldapwhoami -x -H ldap://<NODE_IP>:<NODE_PORT> \
                -D "uid=jsmith,ou=users,dc=example,dc=org" \
                -w abcd1234
```

If the above command returns the DN for the user (specifically, `dn:uid=jsmith,ou=users,dc=example,dc=org`), it means your user has been
successfully provisioned in the OpenLDAP directory and can also authenticate against the directory using the password set.

Finally, check that the member reports the groups that they belong to when querying:

```bash
$ ldapsearch -x -H ldap://<NODE_IP>:<NODE_PORT> \
                -b "ou=users,dc=example,dc=org" \
                -D "cn=admin,dc=example,dc=org" \
                -w abcd1234 \
                "(uid=jsmith)" memberOf
```

The response to the above should return 2 results, one of which is the `memberOf` property indicating that they belong to the `testgrp`
group.

### Dex

Now that we have an LDAP directory service running with a test user account, let's configure permissions for the user to itneract with the
k8s cluster to get pod information (just as an example). As a note, much of the following content is taken from a popular example on the
Dex repository site and adapted for the needs of this tutorial (see "Credits" at the end of this post), and all files can be found in
[this repository folder](https://github.com/jekhokie/scriptbox/tree/master/multiple--vagrant-istio-k8s-cluster/shared/dex). Note that you'll
need to replace the references to `<NODE_IP>` and `<NODE_PORT>` as referenced in this part of the tutorial with the ones specific to your
cluster.

Again, if you wish to skip the section/steps and jump through the install, you can use the files provided in the previous repository which
instantiates a k8s cluster using Vagrant. Note that the contents of the YAML files are very specific in configuration - namely, they assume
that your namespace and all associated resources exist and are run from the `master` node (IP address `10.11.12.13` according to the example).
You can simply run the below steps and then skip the rest of this section, or follow along with the detailed steps within this section.

```bash
# handle SSL components
$ cd shared/dex/
$ openssl genrsa -out ssl/ca-key.pem 2048
$ openssl req -x509 -new -nodes -key ssl/ca-key.pem \
                                -days 10 \
                                -out ssl/ca.pem \
                                -subj "/CN=kube-ca"
$ openssl genrsa -out ssl/key.pem 2048
$ openssl req -new -key ssl/key.pem \
                   -out ssl/csr.pem \
                   -subj "/CN=kube-ca" \
                   -config req.cnf
$ openssl x509 -req -in ssl/csr.pem \
                    -CA ssl/ca.pem \
                    -CAkey ssl/ca-key.pem \
                    -CAcreateserial \
                    -out ssl/cert.pem \
                    -days 10 \
                    -extensions v3_req \
                    -extfile req.cnf

$ sudo vim /etc/kubernetes/manifests/kube-apiserver.conf
# in the configuration file under the 'command' property,
# add the following options:
#    - --oidc-issuer-url=https://10.11.12.13:30000/dex
#    - --oidc-client-id=oidc-auth-client
#    - --oidc-ca-file=/etc/ssl/certs/ca.pem
#    - --oidc-username-claim=email
#    - --oidc-groups-claim=groups
# then write/quit the editor

# create namespace and all associated resources for Dex
$ kubectl apply -f shared/dex/dex.yaml

# perform a query against the Dex well known endpoint to
# ensure it's responding as expected
$ curl --cacert shared/dex/ssl/ca.pem https://10.11.12.13:30000/dex/.well-known/openid-configuration
```

If you want to proceed with the manual/detailed steps, you can now continue here - otherwise, jump to the next section to install and configure
Gangway.

#### TLS Configuration for Dex

First, we'll create a configuration for SSL certificates and components - create a file `req.cnf` under the with the
following contents:

```bash
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1 = 10.11.12.13
```

The most important part of the configuration from a naming perspective is the IP SAN, which if wrong/missing will result in the OAuth flow to
fail. The `IP.1` SAN specified in the above configuration is the IP address of the master node in the Vagrant k8s cluster.

Next, we need to create the respective keys and certificates that are expected. 

```bash
$ mkdir ssl
$ openssl genrsa -out ssl/ca-key.pem 2048

$ openssl req -x509 -new -nodes -key ssl/ca-key.pem \
                                -days 10 \
                                -out ssl/ca.pem \
                                -subj "/CN=kube-ca"

$ openssl genrsa -out ssl/key.pem 2048

$ openssl req -new -key ssl/key.pem \
                   -out ssl/csr.pem \
                   -subj "/CN=kube-ca" \
                   -config req.cnf

$ openssl x509 -req -in ssl/csr.pem \
                    -CA ssl/ca.pem \
                    -CAkey ssl/ca-key.pem \
                    -CAcreateserial \
                    -out ssl/cert.pem \
                    -days 10 \
                    -extensions v3_req \
                    -extfile req.cnf
```

Once the above certs and keys are generaetd, copy the `ca.pem`, `cert.pem`, and `key.pem` filesto the `/etc/ssl/certs/` directory on the master k8s node.

Finally update the `kube-apiserver` configuration to configure the dex endpoint and use the certificate authority certificate for validation of the TLS
bits of Dex. Note that doing the following does not cause issues with the k8s cluster - the logs will simply indicate that attempting to access the Dex
endpoint specified fails as there is no endpoint listening:

```bash
$ sudo vim /etc/kubernetes/manifests/kube-apiserver.conf
# in the configuration file under the 'command' property,
# add the following options:
#    - --oidc-issuer-url=https://10.11.12.13:30000/dex
#    - --oidc-client-id=oidc-auth-client
#    - --oidc-ca-file=/etc/ssl/certs/ca.pem
#    - --oidc-username-claim=email
#    - --oidc-groups-claim=groups

# save and write the file
```

Immediately after saving the file, you should see the API server attempt to re-read the configuration. You can specifically inspect the logs to see
that the `kube-apiserver` pod attempts to access the OIDC endpoint as configured, but fails as Dex is not yet deployed:

```bash
$ kubectl get pods -n kube-system
# look for a `kube-apiserver-master` pod

$ kubectl logs kube-apiserver-master -n kube-system
# should see logs about failure to communicate with
# configured OIDC endpoint - this is expected
```

You're now ready to deploy and configure Dex.

#### Deploy and Configure Dex

There are several components needing to be deployed for Dex to function. Specifically, a namespace will be created into which both Dex and Gangway
will be instantiated, and configurations for each, including a deployment and service definition.

First, let's create the namespace YAML `dex-ns.yaml`:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
    name: auth-system
...
```

And then create the namespace: `kubectl apply -f dex-ns.yaml`.

Next we'll create the ConfigMap object which contains how Dex will be configured - name this `dex-cm.yaml` with the following. Again note the
very specific IP address and static port declaration, and the fact that the values for the LDAP lookups are specific in the NodePort used as
well as the structure that we created in the LDAP directory earlier:

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: dex
  namespace: auth-system
data:
  config.yaml: |
    issuer: https://10.11.12.13:30000/dex

    storage:
      type: kubernetes
      config:
        inCluster: true

    web:
      https: 0.0.0.0:5554
      tlsCert: /etc/ssl/certs/cert.pem
      tlsKey: /etc/ssl/certs/key.pem

    logger:
      level: "debug"
      format: text

    staticClients:
    - id: oidc-auth-client
      redirectURIs:
      - 'http://10.11.12.13:30001/callback'
      name: 'oidc-auth-client'

    connectors:
    - type: ldap
      id: ldap
      name: LDAP
      config:
        host: 10.11.12.13:30595
        insecureNoSSL: true
        insecureSkipVerify: true
        bindDN: CN=admin,dc=example,dc=org
        bindPW: abcd1234
        userSearch:
          baseDN: ou=users,dc=example,dc=org
          filter: "(objectClass=posixAccount)"
          username: uid
          idAttr: uid
          emailAttr: mail
          nameAttr: uid
        groupSearch:
          baseDN: ou=groups,dc=example,dc=org
          filter: "(objectClass=groupOfUniqueNames)"
          userAttr: DN
          groupAttr: uniqueMember
          nameAttr: cn
...
```

Apply the resources: `kubectl apply -f dex-cm.yaml`.

Next we'll create a service account and associated RBAC permissions for Dex to function as expected in `dex-rbac.yaml`:

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dex
  namespace: auth-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: dex
  namespace: auth-system
rules:
- apiGroups: ["dex.coreos.com"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: dex
  namespace: auth-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: dex
subjects:
- kind: ServiceAccount
  name: dex
  namespace: auth-system
...
```

Apply the resources: `kubectl apply -f dex-rbac.yaml`.

The deployment spec is going to define how the bits of configuration and secrets come together - store this in `dex-deploy.yaml`. In the
spec for the deployment, you'll notice a host path mount - this is to expose the certificates we created earlier to the Dex Pod from the
`master` k8s node, where they are stored:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: dex
  name: dex
  namespace: auth-system
spec:
  selector:
    matchLabels:
      app: dex
  replicas: 1
  template:
    metadata:
      labels:
        app: dex
    spec:
      serviceAccountName: dex
      containers:
      - image: quay.io/coreos/dex:v2.9.0
        name: dex
        command: ["dex", "serve", "/etc/dex/cfg/config.yaml"]
        ports:
        - name: http
          containerPort: 5556
        volumeMounts:
        - name: config
          mountPath: /etc/dex/cfg
        - name: etc-ssl-certs
          mountPath: /etc/ssl/certs
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: dex
          items:
          - key: config.yaml
            path: config.yaml
      - name: etc-ssl-certs
        hostPath:
          path: /etc/ssl/certs
          type: DirectoryOrCreate
```

And apply the resources: `kubectl apply -f dex-deploy.yaml`.

Finally, we'll create the service which exposes Dex on a NodePort for access via a `dex-service.yaml` file:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: dex
  namespace: auth-system
spec:
  selector:
    app: dex
  type: NodePort
  ports:
  - name: dex
    port: 5554
    nodePort: 30000
    protocol: TCP
    targetPort: 5554
...
```

Create the service resource: `kubectl apply -f dex-service.yaml`.

At this point, you should have Dex up and running! Test that Dex responds to the well-known path to ensure things are looking good:

`curl --cacert ./ssl/ca.pem https://10.11.12.13:30000/dex/.well-known/openid-configuration`

If things don't respond the way you expect, or if you're curious if there are any misconfigurations, you can inspect various logs:

```bash
# check the kube-apiserver logs to ensure errors have subsided
# indicating the `kube-apiserver` has finally connected with dex
$ kubectl logs kube-apiserver-master -n kube-system

# if you can't get great logs from the apiserver pod, you can also
# access the apiserver logs via the master k8s node itself
$ tail -f /var/log/containers/kube-apiserver...log

# inspect the dex logs to ensure there are no errors showing up
# get the pod name and insert it into the logs command
$ kubectl get pods -n auth-system
$ kubectl logs <DEX_POD_NAME> -n auth-system
```

Additionally, if you notice errors in the logs, a few things may have gone wrong. Here are some common errors that might be found and what
you can do about resolving them:

1. `issuer did not match the issuer returned by the provider`: Indicates there is a mismatch between the issuer defined in the dex
ConfigMap configuration and the kube-apiserver configuration for `--oidc-issuer-url`.
2. `x509: cannot validate certificate for 10.11.12.13 because it doesn't contain any IP SANs`: Indicates the `req.cnf` file used for the
SSL certificates did not have the correct `IP.1` configuration for the `[alt_names]` configuration section (should be set to `10.11.12.13`
for this tutorial).
3. `http: server gave HTTP response to HTTPS client`: You have configured the `--oidc-issuer-url` in the kube-apiserver config to use an
HTTPS endpoint when Dex is listening on an http (non-TLS) endpoint for the address.

### Gangway

Our OIDC proxy is now in place, so let's deploy Gangway. In the same spirit, you can simply execute the following commands using the directory
in [this repository folder](https://github.com/jekhokie/scriptbox/tree/master/multiple--vagrant-istio-k8s-cluster/shared/gangway), or follow
along in the details below the following commands:

```bash
$ kubectl apply -f shared/gangway/gangway.yaml

# visit the gangway URL in a browser and use the OpenLDAP
# user created `jsmith` to log in
#   http://10.11.12.13:30001/
```

#### Deploy and Configure Gangway

Gangway is a fairly straightforward deploy as compared to Dex and most of the resources are well defined. As a note, we'll re-use the
`auth-system` namespace created during the Dex deploy to configure Gangway, so we need not create the namespace resource.

First, we'll create the configuration for Gangway in `gw-cm.yaml`:

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: gangway
  namespace: auth-system
data:
  gangway.yaml: |
    clusterName: "k8s-gangway"
    authorizeURL: "https://10.11.12.13:30000/dex/auth"
    tokenURL: "https://10.11.12.13:30000/dex/token"
    redirectURL: "http://10.11.12.13:30001/callback"
    clientID: "oidc-auth-client"
    allowEmptyClientSecret: true
    usernameClaim: "email"
    emailClaim: "email"
    apiServerURL: "https://10.11.12.13:6443"
    trustedCAPath: "/cacerts/ca.pem"
    scopes: ["groups", "openid", "profile", "email", "offline_access"]
```

Apply the ConfigMap: `kubectl apply -f gw-cm.yaml`.

Next, we'll create a secret that will contain the auth session key in `gw-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gangway-key
  namespace: auth-system
type: Opaque
stringData:
  sessionkey: supertestsecret
```

Create the secret: `kubectl apply -f gw-secret.yaml`.

Next comes the deployment in `gw-deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: gangway
  name: gangway
  namespace: auth-system
spec:
  selector:
    matchLabels:
      app: gangway
  replicas: 1
  template:
    metadata:
      labels:
        app: gangway
    spec:
      containers:
      - image: gcr.io/heptio-images/gangway:v3.1.0
        name: gangway
        command: ["gangway", "-config", "/gangway/gangway.yaml"]
        env:
        - name: SESSION_SECURITY_KEY
          valueFrom:
            secretKeyRef:
              name: gangway-key
              key: sessionkey
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        volumeMounts:
        - name: gangway
          mountPath: /gangway/
        - name: oidc-tls-ca
          mountPath: /cacerts
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 20
          timeoutSeconds: 1
          periodSeconds: 60
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          timeoutSeconds: 1
          periodSeconds: 10
          failureThreshold: 3
      volumes:
        - name: gangway
          configMap:
            name: gangway
        - name: oidc-tls-ca
          hostPath:
            path: /vagrant_data/dex/ssl
            type: Directory
...
```

Apply the deploy: `kubectl apply -f gw-deploy.yaml`.

Finally, the service definition in gw-service.yaml`:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: gangway
  name: gangwaysvc
  namespace: auth-system
spec:
  selector:
    app: gangway
  type: NodePort
  ports:
  - name: "http"
    protocol: TCP
    nodePort: 30001
    port: 80
    targetPort: "http"
...
```

Apply the service: `kubectl apply -f gw-service.yaml`.

At this point, Gangway should be up and communicating with Dex. You can visit the Gangway endpoint by navigating your browser to the following
URL: `http://10.11.12.13:3001`.

### Authenticate and Retrieve Configuration

Now that Gangway, Dex, and the k8s cluster are all talking appropriately, attempt to log into the Gangway UI at `http://10.11.12.13:30001` using
the `jsmith` credentials we created in OpenLDAP - you should walk through the OAuth flow and eventually be presented with a `kubectl` configuration.
Download the configuration and attempt to use it to interact with the k8s cluster (spoiler, you won't be able to do much yet):

`kubectl --kubeconfig=kubeconf.txt get nodes`

This will undoubtedly return an error similar to the following:

`Error from server (Forbidden): nodes is forbidden: User "jsmith@test.com" cannot list resource "nodes" in API group "" at the cluster scope`

So what happened here? While the user authenticated, there are no roles or permissions associated with the user to enable them to interact with
the cluster. We will add permissions in the next section to grant users access to perform specific tasks.

You can also check to see what Dex pulled back from the LDAP server by monitoring the logs of the Dex pod during the login process via Gangway.
Run a `kubectl logs -f <DEX_POD> -n auth-system` command to watch the logs as you attempt to log in as `jsmith` via the Gangway UI - you should
see the `jsmith@test.com` user successfully authenticated and the group to which they belong returned as part of the query response, which sets
up the foundation for enabling group-based RBAC in the following sections.

#### Testing Permissions

As a small deviation, you can use the `can-i` feature of `kubectl auth` to check whether you can perform certain tasks using your environment
configuration/identity. Sampling from the previous section, running the following will show you which granular permissions you have access to
vs. which you do not (hint: you will not have access to perform any of these tasks as there is no RBAC configured for the test user/group yet):

```bash
# test if user can create deployments in the default namespace
$ kubectl --kubeconfig=kubeconf.txt auth can-i create deployments --namespace default

# test if user can get pods in the auth-system namespace
$ kubectl --kubeconfig=kubeconf.txt auth can-i get pods --namespace auth-system
```

Again, all above commands should return `no` in their response - we will set up some permissions to enable `jsmith` to do certain things in the
next section.

### Creating Permissions via Roles

Now that we have a successful OIDC flow enabling self-service for any authenticated users in our LDAP directory, we need to add permissions for
those users to actually be able to interact with the cluster.

To skip this section and jump straight to the content, you can run the following short sequence of steps using the existing repository contents:

```bash
# apply role and role binding
$ kubectl apply -f shared/rbac/pod-viewer.yaml

# attempt can-i for the permission - should return 'yes'
$ kubectl --kubeconfig=kubeconf.txt auth can-i get pods --namespace default
```

#### Create Role and Role Binding

First, let's create a Role which enables view/read permissions on pods in the default
namespace (you can easily see how you could scope permissions to specific namespaces, etc. via the configuration) via the file `pod-viewer.yaml`:

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: default
subjects:
- kind: Group
  name: "testgrp"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
...
```

Apply the role and associated binding: `kubectl apply -f pod-viewer.yaml`.

Next, re-test your `can-i` permission (and even attempt to get the pods in the namespace):

```bash
# test can-i permission
$ kubectl --kubeconfig=kubeconf.txt auth can-i get pods --namespace default

# actually list the pods in the default namespace
$ kubectl --kubeconfig=kubeconf.txt get pods --namespace default
```

You now have the basis for managing users via groups and k8s RBAC!

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [OpenLDAP Helm Chart](https://www.thegeekstuff.com/2015/02/openldap-add-users-groups/)
* [Add OpenLDAP Users/Groups](https://www.thegeekstuff.com/2015/02/openldap-add-users-groups/)
* [Dex - Authentication through LDAP](https://github.com/dexidp/dex/blob/master/Documentation/connectors/ldap.md)
* [Gangway](https://github.com/heptiolabs/gangway)
* [Connecting Gangway to Dex](https://github.com/heptiolabs/gangway/blob/master/docs/dex.md)
* [Kubernetes Authentication On-Prem Cluster with LDAP](https://medium.com/@ahmed.ebaya/kubernetes-authentication-in-private-cluster-13c44115bf3)
