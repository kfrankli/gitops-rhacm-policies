# Red Hat Advanced Cluster Management for Kubernetes GitOps Example

> [!NOTE]
> If you found this repo helpful or you found a bug, please feel free to submit a PR, drop me an email kfrankli@redhat.com, or even just star the repo. I welcome any and all feedback. 

##  Overview

This repository consists of two simple example GitOps patterns for deploying policies in Red Hat Advanced Cluster Management for Kubernete (RHACM) using the [policy-generator-plugin](https://github.com/open-cluster-management-io/policy-generator-plugin) for kustomize via the preferred as of RHACM 2.9 [ArgoCD based OpenShift GitOps Operator](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.10/html/gitops/index) or via the [RHACM native Application Model](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.10/html/applications/managing-applications). There are five policies that will be deployed:

* `operator-configuration` - Deploy the Red Hat [web-terminal](https://github.com/redhat-developer/web-terminal-operator) Operator to all clusters. This is the "simplest" manifest to study and learn.
* `console-banner` - Deploys a simple console banner unique to each environment. In doing so demonstrates how to handle basic environment specific variables in this GitOps model.
* `migrate-workloads-to-infra-nodes` - "Migrates" the Monitoring and Default Router to infrastructure nodes by adding the node-selector and tolerations per [Red Hat Documentation: Moving Resources to Infrastructure Machinesets](https://docs.openshift.com/container-platform/4.14/machine_management/creating-infrastructure-machinesets.html#moving-resources-to-infrastructure-machinesets)
* `oauth-ldap` - Deploys a ldap based oauth config to all clusters managed by RHACM to secure console access to OpenShift
* `f5-cis-configurations` - Deploys [OpenShift OVN-Kubernetes using F5 BIG-IP HA with NO Tunnels](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/ovn-kubernetes-ha#step-4-configure-egress-from-openshift-cluster-to-big-ip)

Thank you to [Alberto Gonzalez de Dios](https://github.com/albertogd) for his phenomenal GitOps series.

## Assumptions

* A Red Hat OpenShift cluster at 4.14+
* Red Hat Advanced Cluster Management for Kubernetes installed at 2.10+
* The OpenShift `oc` client utility installed locally 
* You have Infrastructure node machinesets in place [Red Hat Documentation: Creating Infrastructure Machinesets](https://docs.openshift.com/container-platform/4.14/machine_management/creating-infrastructure-machinesets.html)

## Using the ArgoCD based OpenShift GitOps Operator (Recommended)

[OpenShift GitOps (ArgoCD) Instructions](argocd.md)

## Using the RHACM Native Application Model

[RHACM Native Application Model Instructions](acm-native-gitops.md)

## Why use the policy-generator-plugin?

Writing the manifest for RHACM policies, placements, and placement bindings can quickly become rather complex. The [policy-generator-plugin](https://github.com/open-cluster-management-io/policy-generator-plugin) for kustomize can simplify a significant portion of this effort as it provides the following capabilities:

* Convert Kubernetes manifest files to RHACM configuration policies
* Patch Kubernetes manifests before they are inserted into a generated RHACM policy
* Run locally to allow the RHACM policies to be stored in git
* Alternatively, run by GitOps tools so that only the manifests and the policy generator configuration need to be stored in Git as is the case for this repo

Consider the equivalent of the object we have defined in `policies/manifests/operators/web-terminal.yaml` is the following policy, and we can see the advantages of using this plugin with regards to simplicity, ease of writing, and brevity:

```
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement-operator-configuration
  namespace: policies
spec:
  predicates:
  - requiredClusterSelector:
      labelSelector:
        matchExpressions: []
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: binding-operator-configuration
  namespace: policies
placementRef:
  apiGroup: cluster.open-cluster-management.io
  kind: Placement
  name: placement-operator-configuration
subjects:
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: operator-configuration
---
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  annotations:
    policy.open-cluster-management.io/categories: CM Configuration Management
    policy.open-cluster-management.io/controls: CM-2 Baseline Configuration
    policy.open-cluster-management.io/description: ""
    policy.open-cluster-management.io/standards: NIST SP 800-53
  name: operator-configuration
  namespace: policies
spec:
  disabled: false
  policy-templates:
  - objectDefinition:
      apiVersion: policy.open-cluster-management.io/v1
      kind: ConfigurationPolicy
      metadata:
        name: operator-configuration
      spec:
        object-templates:
        - complianceType: musthave
          objectDefinition:
            apiVersion: operators.coreos.com/v1alpha1
            kind: Subscription
            metadata:
              labels:
                operators.coreos.com/web-terminal.openshift-operators: ""
              name: web-terminal
              namespace: openshift-operators
            spec:
              channel: fast
              installPlanApproval: Automatic
              name: web-terminal
              source: redhat-operators
              sourceNamespace: openshift-marketplace
        remediationAction: enforce
        severity: low
  remediationAction: enforce
```

Of course it isn't perfect for all situations, as it does somewhat simplify the available possible configurations for more consult [Policy Generator Reference](https://github.com/open-cluster-management-io/policy-generator-plugin/blob/main/docs/policygenerator-reference.yaml)

## Debugging/Dryruns

If you're trying to make your own modifications to this repo, it's quite helpful to run locally to ensure the objects are coming together as expected. First install the [policy-generator-plugin](https://github.com/open-cluster-management-io/policy-generator-plugin) for kustomize.

Then it is as simple as running the following in the root of this repository(it is assumed you have previously installed the OpenShift Client):

```console
oc kustomize ./policies/ --enable-alpha-plugins
```

You can also further chain this command via a pipe, `|`, to dry run it:

```console
oc kustomize ./policies/ --enable-alpha-plugins | oc create -f - --dry-run
```

## Important Note on Secrets

This repository does in fact have kubernetes secrets committed to it since this is only a POC. As a reminder, the kubernetes secrets object is not itself secret. The contents of the object are by default a base64 encoding and are not encrypted. This is *not* something you should do as part of a production deployment strategy. Instead the secrets should be externalized via a different systems such as [SOPS](https://www.redhat.com/en/blog/a-guide-to-gitops-and-secret-management-with-argocd-operator-and-sops), [Hashicorp Vault](https://www.redhat.com/en/blog/openshift-gitops-with-argo-cd), [CyberArk](https://demo.openshift.com/en/latest/cyberark-for-openshift/), etc...

[Red Hat Blog: A Holistic approach to encrypting secrets, both on and off your OpenShift clusters](https://www.redhat.com/en/blog/holistic-approach-to-encrypting-secrets-both-on-and-off-your-openshift-clusters)


## References

* [Red Hat Solutions: Examples when Subscription Admin needs to be enabled in RHACM-Gitops scenarios](https://access.redhat.com/solutions/6010251)
* [Product Documentation: RHACM: Granting subscription administrator privilege](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.10/html-single/applications/index#granting-subscription-admin-privilege)
* [Product Documentation: RHACM: Configuring application channel and subscription for a secure Git connection](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.10/html-single/applications/index#configuring-git-channel)
* [Red Hat Blog: Generating Governance Policies Using Kustomize and GitOps](https://www.redhat.com/en/blog/generating-governance-policies-using-kustomize-and-gitops)
* [Red Hat Blog: GitOps for organizations: provisioning and configuring OpenShift clusters automatically](https://www.redhat.com/en/blog/gitops-for-organizations-provisioning-and-configuring-openshift-clusters-automatically)
* [Red Hat Blog: A Holistic approach to encrypting secrets, both on and off your OpenShift clusters](https://www.redhat.com/en/blog/holistic-approach-to-encrypting-secrets-both-on-and-off-your-openshift-clusters)
* [Policy Generator](https://github.com/open-cluster-management-io/policy-generator-plugin/tree/main)
* [Policy Generator Reference](https://github.com/open-cluster-management-io/policy-generator-plugin/blob/main/docs/policygenerator-reference.yaml)
* [F5 Container Ingress Services User Guides and More: OpenShift OVN-Kubernetes using F5 BIG-IP HA with NO Tunnels](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/ovn-kubernetes-ha#step-4-configure-egress-from-openshift-cluster-to-big-ip)