# Install gitops on hub cluster

https://access.redhat.com/documentation/en-us/openshift_container_platform/4.15/html/cicd/gitops
https://docs.openshift.com/gitops/1.12/understanding_openshift_gitops/about-redhat-openshift-gitops.html

[Product Documentation: Red Hat OpenShift GitOps: Installing GitOps](https://docs.openshift.com/gitops/1.12/installing_gitops/installing-openshift-gitops.html#installing-gitops-operator-using-cli_installing-openshift-gitops)

```
$ oc create ns openshift-gitops-operator
$ cat gitops-operator-group.yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-gitops-operator
  namespace: openshift-gitops-operator
spec:
  upgradeStrategy: Default
$ oc apply -f gitops-operator-group.yaml
$ cat openshift-gitops-sub.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-gitops-operator
spec:
  channel: latest 
  installPlanApproval: Automatic
  name: openshift-gitops-operator 
  source: redhat-operators 
  sourceNamespace: openshift-marketplace
$ oc apply -f openshift-gitops-sub.yaml
$ oc get pods -n openshift-gitops
$ oc get pods -n openshift-gitops-operator
```

# Configure ArgoCD to allow the integrate with the policy generator

[Product Documentation: RHACM: Managing policy definitions with OpenShift Container Platform GitOps (Argo CD)](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.10/html/gitops/gitops-overview#policy-gen-install-on-openshift-gitops)

Error in docs. Need `argoproj.io/v1beta1` but docs say `argoproj.io/v1alpha1`

```
$ cat argocd-patch.yaml
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata:
  name: openshift-gitops
  namespace: openshift-gitops
spec:
  kustomizeBuildOptions: --enable-alpha-plugins
  repo:
    env:
    - name: KUSTOMIZE_PLUGIN_HOME
      value: /etc/kustomize/plugin
    - name: POLICY_GEN_ENABLE_HELM
      value: "true"
    initContainers:
    - args:
      - -c
      - cp /policy-generator/PolicyGenerator-not-fips-compliant /policy-generator-tmp/PolicyGenerator
      command:
      - /bin/bash
      image: '{{ (index (lookup "apps/v1" "Deployment" "open-cluster-management" "multicluster-operators-hub-subscription").spec.template.spec.containers 0).image }}'
      name: policy-generator-install
      volumeMounts:
      - mountPath: /policy-generator
        name: policy-generator-tmp
    volumeMounts:
    - mountPath: /etc/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator
      name: policy-generator
    volumes:
    - emptyDir: {}
      name: policy-generator
$ oc get argocd openshift-gitops -n openshift-gitops -o yaml > openshift-gitops.yaml
$ oc -n openshift-gitops patch argocd openshift-gitops --type merge --patch "$(cat argocd-patch.yaml)" 
$ cat openshift-gitops-policy-admin.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: openshift-gitops-policy-admin
rules:
  - verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete
    apiGroups:
      - policy.open-cluster-management.io
    resources:
      - policies
      - policysets
      - placementbindings
  - verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete
    apiGroups:
      - apps.open-cluster-management.io
    resources:
      - placementrules
  - verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete
    apiGroups:
      - cluster.open-cluster-management.io
    resources:
      - placements
      - placements/status
      - placementdecisions
      - placementdecisions/status
$ oc apply -f openshift-gitops-policy-admin.yaml
$ cat openshift-gitops-policy-admin-clusterolebinding.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: openshift-gitops-policy-admin
subjects:
  - kind: ServiceAccount
    name: openshift-gitops-argocd-application-controller
    namespace: openshift-gitops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-gitops-policy-admin
$ oc apply -f openshift-gitops-policy-admin-clusterolebinding.yaml
```

# Create rbac stuff

```
$ oc adm groups new argocdadmins
$ oc adm groups add-users argocdadmins $(oc whoami)
$ oc edit argocd -n openshift-gitops
[...]
  rbac:
    defaultPolicy: ""
    policy: |
      g, system:cluster-admins, role:admin
      g, cluster-admins, role:admin
      g, argocdadmins, role:admin
    scopes: '[groups]'
[...]
```

# Generate policies using OpenShift GitOps

```
$ oc create namespace policies
$ cat gitops-policies-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-policies
  namespace: openshift-gitops
spec:
  destination:
    namespace: policies
    server: https://kubernetes.default.svc
  project: default
  source:
    repoURL: https://github.com/kfrankli/gitops-rhacm-policies.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
$ oc get route openshift-gitops-server -n openshift-gitops

```