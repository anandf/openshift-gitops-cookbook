
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

### Create a secret for Database Credentials

```shell
oc create secret generic keycloak-db-secret -n keycloak\
  --from-literal=username=postgres \
  --from-literal=password=testpassword
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
              valueFrom:
                secretKeyRef:
                  name: keycloak-db-secret
                  key: password
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
    host: postgres-db.keycloak.svc.cluster.local
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

### Annotate the keycloak service to generate TLS certs

```shell
oc annotate svc/argocd-keycloak-service -n keycloak service.beta.openshift.io/serving-cert-secret-name=argocd-keycloak-tls
```

### Wait for the Keycloak instance to be ready

```shell
oc wait keycloaks/argocd-keycloak -n keycloak \
  --for=jsonpath='{.status.conditions[?(@.type=="Ready")].status}=True' \
  --timeout=3m
```

### Get the Keycloak admin secret

```shell
export KC_ADMIN_USERNAME=$(oc get secret argocd-keycloak-initial-admin -n keycloak -o jsonpath='{.data.username}' | base64 -d)
export KC_ADMIN_PASSWORD=$(oc get secret argocd-keycloak-initial-admin -n keycloak -o jsonpath='{.data.password}' | base64 -d)
```

### Get the host address of Argo CD

```shell
export ARGOCD_HOSTNAME=$(oc get route -n openshift-gitops openshift-gitops-server -o jsonpath='{.spec.host}')
```

### Generate a client secret for Argo CD client

```shell
export CLIENT_SECRET=$(openssl rand -base64 48 | tr -dc 'A-Za-z0-9' | head -c 32)
export USERNAME_SECRET=$(openssl rand -base64 48 | tr -dc 'A-Za-z0-9' | head -c 12)
```

### Patch the argocd secret

```shell
oc patch secret argocd-secret -n openshift-gitops -p '{"stringData": {"oidc.keycloak.clientSecret":  "${CLIENT_SECRET}"}}' --type=merge
```

### Create the KeycloakRealmImport Custom Resource to create a realm

```shell
oc create -f - <<EOF
apiVersion: k8s.keycloak.org/v2alpha1
kind: KeycloakRealmImport
metadata:
  name: argocd-realm
  namespace: keycloak
spec:
  keycloakCRName: argocd-keycloak
  realm:
    id: argocd-realm
    realm: argocd-realm
    displayName: Argo CD Realm
    enabled: true
    clients:
      - clientId: argocd
        name: ArgoCD Client
        description: OIDC client for Argo CD Integration
        rootUrl: "https://${ARGOCD_HOSTNAME}"
        adminUrl: "https://${ARGOCD_HOSTNAME}"
        baseUrl: /applications
        surrogateAuthRequired: false
        alwaysDisplayInConsole: false
        enabled: true
        attributes:
          realm_client: "false"
          oidc.ciba.grant.enabled: "false"
          backchannel.logout.session.required: "true"
          standard.token.exchange.enabled: "false"
          post.logout.redirect.uris: "https://${ARGOCD_HOSTNAME}/applications"
          oauth2.device.authorization.grant.enabled": "false"
          backchannel.logout.revoke.offline.tokens": "false"
          dpop.bound.access.tokens": "false"
        fullScopeAllowed: true
        protocol: openid-connect
        clientAuthenticatorType: client-secret
        redirectUris:
          - "https://${ARGOCD_HOSTNAME}/auth/callback"
        webOrigins:
          - "https://${ARGOCD_HOSTNAME}"
        secret: "${CLIENT_SECRET}"
        serviceAccountsEnabled: true
        standardFlowEnabled: true
        implicitFlowEnabled: false
        directAccessGrantsEnabled: true
        publicClient: false
        frontchannelLogout: true
        notBefore: 0
        bearerOnly: false
        consentRequired: false
        defaultClientScopes:
          - openid
          - profile
          - email
          - groups
          - basic
          - roles
        optionalClientScopes:
          - address
          - phone
          - offline_access
          - organization
          - microprofile-jwt
          - web-origins
          - acr
    defaultDefaultClientScopes:
      - openid
      - roles
      - profile
      - email
      - roles
      - web-origins
      - acr
      - basic
      - groups
    defaultOptionalClientScopes:
      - address
      - phone
      - offline_access
      - organization
      - microprofile-jwt
    groups:
      - name: ArgoCDAdmins
        description: Argo CD Administrators,
        path: /ArgoCDAdmins
    users:
      - username: anandf
        firstName: Anand Francis
        lastName: Joseph
        email: anandfrancisjoseph@gmail.com
        emailVerified: true
        enabled: true
        groups:
          - ArgoCDAdmins
        credentials:
          - type: password
            value: "${USERNAME_SECRET}"
            temporary: false
EOF
```

### Wait for the realm to be initialized

```shell
oc wait keycloakrealmimport/argocd-realm -n keycloak \
  --for=jsonpath='{.status.conditions[?(@.type=="Done")].status}=True' \
  --timeout=3m
```


### Get the Keycloak hostname

```shell
export KEYCLOAK_HOSTNAME=$(oc get route -n keycloak | grep argocd-keycloak-service | awk '{print $2}')
```

### Add ClientScope Group to the Client

#### Login to the Keycloak server

```shell
kcadm.sh config credentials --server ${KEYCLOAK_HOSTNAME} --realm master --user ${KC_ADMIN_USERNAME} --password ${KC_ADMIN_PASSWORD}
```

**Note:** if you are getting cert related errors, refer this solution https://access.redhat.com/solutions/7076894

#### Create the ClientScope called 'groups'

```shell
kcadm.sh create client-scopes -r argocd-realm -s '
{
  "name": "groups",
  "description": "",
  "protocol": "openid-connect",
  "attributes": {
    "include.in.openid.provider.metadata": "true",
    "include.in.token.scope": "false",
    "display.on.consent.screen": "true",
    "gui.order": "",
    "consent.screen.text": ""
  },
  "protocolMappers": [
    {
      "name": "groups",
      "protocol": "openid-connect",
      "protocolMapper": "oidc-group-membership-mapper",
      "consentRequired": false,
      "config": {
        "full.path": "false",
        "introspection.token.claim": "true",
        "userinfo.token.claim": "true",
        "id.token.claim": "true",
        "lightweight.claim": "false",
        "access.token.claim": "true",
        "claim.name": "groups"
      }
    }
  ]
}
'
```
#### Get the Client and ClientScope UUIDs

```shell
# Get the ID of the Client (e.g., "my-custom-client")
export CLIENT_UUID=$(kcadm.sh get clients -r argocd-realm -q clientId=argocd --fields id --format csv --noquotes)

# Get the ID of the Scope (e.g., "groups")
export SCOPE_UUID=$(kcadm.sh get client-scopes -r argocd-realm -q name=groups --fields id --format csv --noquotes)
```

#### Update the client to include the client scope

```shell
kcadm.sh update clients/$CLIENT_UUID/default-client-scopes/$SCOPE_UUID -r argocd-realm
```
### Patch the ArgoCD with the OIDC Config and remove SSO

```shell
oc patch argocd openshift-gitops -n openshift-gitops --type='json' -p '
[
  {
    "op": "add",
    "path": "/spec/oidcConfig",
    "value": "name: Keycloak\nissuer: https://${KEYCLOAK_HOSTNAME}/realms/argocd-realm\nclientID: argocd\nclientSecret: $oidc.keycloak.clientSecret\nrequestedScopes: [\"openid\", \"profile\", \"email\", \"groups\"]"
  },
  {
    "op": "replace",
    "path": "/spec/sso",
    "value": null
  },
]
'
```

**Note:** If you are using self-signed certificate tell Argo CD to ignore the errors in cert verification

```shell
oc patch argocd openshift-gitops -n openshift-gitops --type=merge -p '{"spec": {"extraConfig": {"oidc.tls.insecure.skip.verify": "true"}}}'
```
### Restart the Argo CD Server

Restart the Argo CD Server pod to ensure that the configuration changes are in effect.

```shell
oc scale deployment openshift-gitops-server -n openshift-gitops --replicas=0
oc scale deployment openshift-gitops-server -n openshift-gitops --replicas=1
```