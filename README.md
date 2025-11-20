
## Introduction
This document describes the steps required to enable the Source Hydrator feature in OpenShift GitOps v1.16.0 or later.

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
  startingCSV: openshift-gitops-operator.v1.18.0
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
oc rollout status deploy/openshift-gitops-operator-controller-manager -n openshift-gitops-operator -w
```

### Wait for the ArgoCD instance to be available
```shell
oc wait argocd/openshift-gitops -n openshift-gitops \
  --for=jsonpath='{.status.phase}'=Available \
  --timeout=3m
```

## Enable the source hydrator feature

```shell
oc patch argocd openshift-gitops -n openshift-gitops  -p '{"spec": {"extraConfig": {"hydrator.enabled": "true"}}}' --type merge
```

## Install the Source Hydrator (commit-server)

```shell
gomplate -f ${PWD}/hydrator-install/kustomization.yaml.tmpl -o ${PWD}/hydrator-install/kustomization.yaml --datasource argocd=${PWD}/hydrator-install/argocd.yaml && oc create -k ${PWD}/hydrator-install -n openshift-gitops
```

Note: If you need to use a different Argo CD version or Argo CD instance, override the values in `values.yaml` before running the above command.

## Create secrets containing the credentials required to connect to git repository for push/pull operations

### For Github App based connection
```
gomplate -f ${PWD}/auth/templates/github-app.yaml.tmpl -o ${PWD}/auth/github-app.yaml --datasource config=${PWD}/auth/data/config.yaml && oc apply -f  ${PWD}/auth/github-app.yaml -n openshift-gitops
```

### For HTTPS based connection
```
gomplate -f ${PWD}/auth/templates/git-https.yaml.tmpl -o ${PWD}/auth/git-https.yaml --datasource config=${PWD}/auth/data/config.yaml && oc apply -f  ${PWD}/auth/git-https.yaml -n openshift-gitops
```

### For SSH based connection
```
gomplate -f ${PWD}/auth/templates/git-ssh.yaml.tmpl -o ${PWD}/auth/git-ssh.yaml --datasource config=${PWD}/auth/data/config.yaml && oc apply -f  ${PWD}/auth/git-ssh.yaml -n openshift-gitops
```
