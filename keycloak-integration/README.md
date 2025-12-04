
## Introduction
This document describes the steps required to integrate Keycloak for SSO authentication in OpenShift GitOps v1.16.0 or later.

## Prerequisites
- OpenShift Cluster with v4.16 or later
- The OpenShift CLI binary (oc) is installed and available in your system's PATH.
- gomplate binary is installed and available in your system's PATH. If not it can be installed via `brew install gomplate`.

## Installation of OpenShift GitOps

### Create the namespace

```shell
oc create namespace openshift-gitops-operator
```

### Create the OperatorGroup

```shell
oc create -f - <<EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-gitops-og
  namespace: openshift-gitops-operator
spec:
  upgradeStrategy: Default
EOF
```

### Create the Subscription

```shell
oc create -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/openshift-gitops-operator.openshift-gitops-operator: ""
  name: openshift-gitops-operator
  namespace: openshift-gitops-operator
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: openshift-gitops-operator.v1.18.1
EOF
```
### Wait for the installation to complete

```shell
oc wait subscription openshift-gitops-operator -n openshift-gitops-operator \
  --for=jsonpath='{.status.installedCSV}' \
  --timeout=3m
```
### Wait for the operator deployment to be ready
```shell
until oc get deploy/openshift-gitops-operator-controller-manager -n openshift-gitops-operator &> /dev/null; do \
  sleep 10; \
done && \
oc rollout status deploy/openshift-gitops-operator-controller-manager -n openshift-gitops-operator
```

### Wait for the ArgoCD instance to be available
```shell
oc wait argocd/openshift-gitops -n openshift-gitops \
  --for=jsonpath='{.status.phase}'=Available \
  --timeout=3m
```

## Installation of Red Hat Build of Keycloak Operator

### Create the namespace

```shell
oc create ns keycloak
```

### Create the OperatorGroup

```shell
oc create -f - <<EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: keycloak-og
  namespace: keycloak
spec:
  targetNamespaces:
  - keycloak
  upgradeStrategy: Default
EOF
```

### Create the subscription for RedHat Build of Keycloak operator

```shell
oc create -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/rhbk-operator.keycloak: ""
  name: rhbk-operator
  namespace: keycloak
spec:
  channel: stable-v26.4
  installPlanApproval: Automatic
  name: rhbk-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### Create the Database for Keycloak

```shell
oc create -f - <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql-db
  namespace: keycloak
spec:
  serviceName: postgresql-db-service
  selector:
    matchLabels:
      app: postgresql-db
  replicas: 1
  template:
    metadata:
      labels:
        app: postgresql-db
    spec:
      containers:
        - name: postgresql-db
          image: postgres:latest
          volumeMounts:
            - mountPath: /data
              name: cache-volume
          env:
            - name: POSTGRES_PASSWORD
              value: testpassword
            - name: PGDATA
              value: /data/pgdata
            - name: POSTGRES_DB
              value: keycloak
      volumes:
        - name: cache-volume
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
  namespace: keycloak
spec:
  selector:
    app: postgresql-db
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
EOF
```
### Expose the DB service as a route

```shell
oc expose service postgres-db -n keycloak --name=postgres-db-route
```

### Get the Postgres route host address

```shell
POSTGRES_ADDRESS=$(oc get route -n keycloak postgres-db-route -o jsonpath='{.spec.host}')
```
### Create a self signed certificate

```shell
openssl req -subj '/CN=argocd.<domain_name>/O=Test Keycloak./C=US' -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
```

### Create the secret with the cert credentials
```shell
oc create secret tls argocd-keycloak-tls -n keycloak --cert certificate.pem --key key.pem 
```

[OR]

### Annotate the keycloak service
```shell
oc annotate svc/argocd-keycloak-service -n keycloak service.beta.openshift.io/serving-cert-secret-name=argocd-keycloak-tls
```

### Create DB secret
```shell
oc create secret generic keycloak-db-secret -n keycloak\
  --from-literal=username=postgres \
  --from-literal=password=testpassword
```

### Create the Keycloak Custom Resource

```shell
oc create -f - <<EOF
apiVersion: k8s.keycloak.org/v2alpha1
kind: Keycloak
metadata:
  name: argocd-keycloak
  namespace: keycloak
spec:
  instances: 1
  db:
    vendor: postgres
    host: postgres-db
    usernameSecret:
      name: keycloak-db-secret
      key: username
    passwordSecret:
      name: keycloak-db-secret
      key: password
  http:
    tlsSecret: argocd-keycloak-tls
    insecureSkipVerify: true
  ingress:
    className: openshift-default
EOF
```

### Wait for the Keycloak instance to be ready

```shell
oc wait keycloaks/argocd-keycloak -n keycloak \
  --for=jsonpath='{.status.conditions[?(@.type=="Ready")].status}=True' \
  --timeout=3m
```
### Get the ingresses internal hostname

```shell
INTERNAL_HOSTNAME=$(oc get ingress argocd-keycloak-ingress -n keycloak -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
INTERNAL_HOSTIP=$(dig +short ${INTERNAL_HOSTNAME})
sudo echo "${INTERNAL_HOSTIP}\t test.keycloak.org" >> /etc/hosts
```

### Get the Keycloak admin secret 

```shell
oc get secret argocd-keycloak-initial-admin -n keycloak -o jsonpath='{.data.username}' | base64 -d
oc get secret argocd-keycloak-initial-admin -n keycloak -o jsonpath='{.data.password}' | base64 -d
```

### Create the KeycloakRealmImport Custom Resource to create a realm
```shell
oc create -f - <<EOF
apiVersion: k8s.keycloak.org/v2alpha1
kind: KeycloakRealmImport
metadata:
  name: argocd-realm-kc
  namespace: keycloak
spec:
  keycloakCRName: argocd-keycloak
  realm:
    id: argocd-realm
    realm: argocd-realm
    displayName: Argo CD Realm
    enabled: true
EOF
```

### Wait for the realm to be initialized
```shell
oc wait keycloakrealmimport/argocd-realm-kc -n keycloak \
  --for=jsonpath='{.status.conditions[?(@.type=="Done")].status}=True' \
  --timeout=3m
```

### Patch the argocd secret
```shell
oc patch secret argocd-secret -n openshift-gitops -p {"oidc.keycloak.clientSecret":  "5J5jEOYtkqTVC5aQI0MvEC7TmlTSVR5S"} --type=merge
```

### Patch the ArgoCD with the OIDC Config
```shell
  spec.oidcConfig: |
    name: Keycloak
    issuer: https://argocd.<domain_name>/realms/argocd-realm
    clientID: argocd
    clientSecret: $oidc.keycloak.clientSecret
    requestedScopes: ["openid", "profile", "email", "groups"]

  spec.sso: null
```
