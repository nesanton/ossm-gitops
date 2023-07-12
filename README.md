# ossm-gitops

Example App-of-Apps repo for Red Hat Openshift GitOps (ArgoCD) to roll out and perform Lifecycle Management of Red Hat Openshift Service Mesh.

>**IMPORTANT:** This is a potential starting point, but definitely not a production setup. Feel free to contribute.

## Lifecycle Management

### TL&DR

* Use the [version: X.Y](https://github.com/nesanton/ossm-gitops/blob/ac07b486544349ff76eb9a5f2595b7bb5a0fa923/smcp/smcp.yml#L30) of SMCP instance as the main and only control of service mesh versioning. Follow [release notes](https://docs.openshift.com/container-platform/4.13/service_mesh/v2x/servicemesh-release-notes.html#component-versions-included-in-red-hat-openshift-service-mesh-version-2-4) and [support cycle](https://access.redhat.com/support/policy/updates/openshift#ossm) docs for decision making on OSSM upgrades.
* Keep the Operators' update strategy at `Automatic` to ensure seamless Z-stream patches to the Service Mesh control plane and version compatibility across components.
* In order for the User Workloads to use the new envoy sidecar images that came with a Z- or Y-stream updates the `Deployments` of these Workloads need a restart.

### More details

All operators of the productized version of OSSM are quite advanced in their [capability levels](https://sdk.operatorframework.io/docs/overview/operator-capabilities/) which means that keeping them at Automatic upgrade strategy is highly recommended and beneficial, reducing the LCM efforts dramatically. Every time an operator updates is becomes aware of the new versions of operands' images and seamlessly performs necessary changes. In case of OSSM, the X.Y version of the Service Mesh Controll Plane will stay the same (as specified in its CR), while Z-stream patches will be automatically applied to the Control PLane components.

Organizations often maintain a cadence or some well-defined process for patches and upgrades. Patching would include Z-stream number changes, while upgrades are changing the Y and potentially X.

#### Patching

If the Operators are set for Automatic updates, no additional action is required for the control plane.

For the data plane (sidecars in the user workloads), the containers need to be restarted, e.g. by means of restarting the Deployments. For example, this can be performed at a patching cadence, say every two weeks unconditionally. Alternatively such restarts can be a result of monitoring events for the Servicemesh operator updates. 

#### Upgrades

Decisions for Upgrades are often made based on interest in new features or End-of-Life of a particulat X.Y combination approaching. Follow [release notes](https://docs.openshift.com/container-platform/4.13/service_mesh/v2x/servicemesh-release-notes.html#component-versions-included-in-red-hat-openshift-service-mesh-version-2-4) and [support cycle](https://access.redhat.com/support/policy/updates/openshift#ossm) docs for decision making on OSSM upgrades.

## How to use this repo

### Install the Openshift GitOps operator

Install the operator as usual

### Add custom HealthChecks

Add a couple of custom HealthChecks for the CRDs that ArgoCD isn't aware of. We'll create one for the `Subscription` object and one for the `ServiceMeshControlPlane`.
Edit/Patch the `argocd/openshift-gitops` object in `openshift-gitops` namespace to have:

>**NOTE**: You may get a warning that this object is managed and your changes won't stick. You can safely ignore it, because the `extraConfig` field is reconsiled as-is by design and is not paved over. This is a means of allowing new features to be added to the productized version of ArgoCD due to its naturally slower update cadence. See [GITOPS-1964](https://issues.redhat.com/browse/GITOPS-1964) for details.

```yaml
...
spec:
...
  extraConfig:
    resource.customizations: |
      operators.coreos.com/Subscription:
        health.lua: |
          hs = {}
          hs.status = "Progressing"
          hs.message = ""
          if obj.status ~= nil then
            if obj.status.state ~= nil then
              if obj.status.state == "AtLatestKnown" then
                hs.message = obj.status.state .. " - " .. obj.status.currentCSV
                hs.status = "Healthy"
              end
            end
          end
          return hs
      maistra.io/ServiceMeshControlPlane:
        health.lua: |
          hs = {}
          if obj.status ~= nil then
            if obj.status.conditions ~= nil then
              for i, condition in ipairs(obj.status.conditions) do
                if condition.type == "Ready" and condition.status == "False" then
                  hs.status = "Degraded"
                  hs.message = condition.message
                  return hs
                end
                if condition.type == "Ready" and condition.status == "True" then
                  hs.status = "Healthy"
                  hs.message = condition.message
                  return hs
                end
              end
            end
          end

          hs.status = "Progressing"
          hs.message = "Waiting for certificate"
          return hs
```

### Add permissions

ArgoCD needs to have permissions to modify cluster resources. For simplicity we'll add the `cluster-admin` cluster role to its service account. In production you may want to setup a more granular set of permissions.

`oc adm policy add-cluster-role-to-user cluster-admin -z openshift-gitops-argocd-application-controller -n openshift-gitops`

### Create an App in Openshift GitOps

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ossm
  namespace: openshift-gitops
spec:
  destination:
    server: 'https://kubernetes.default.svc'
  project: default
  source:
    path: apps
    repoURL: 'https://github.com/nesanton/ossm-gitops'
    targetRevision: HEAD
  syncPolicy:
    automated:
      selfHeal: true
```

Confirm that the App has synced the entire App-of-Apps and the Service Mesh Control PLane.

## Points of attention

* There are no `finalizers` set on the Apps and `prune` is set to `false`. OLM (Operator Lifecycle Manager) today does not have a high level object that corresponds to Operators. Instead, a `Subscription` needs to be templated for operator installatiion. Removal is a somewhat more complex process. To keep things clean, it's better to consider operator installation as a one-off task in a clusters' lifetime. That said, one can freely change the `Subscription` object in the code when needed and have it propagated into the clusters. E.g. `StartingCSV` may need a change once in a while.
* App of Apps is rolled out in waves to ensure that the operators are healthy, namespaces (mainly openshift-distributed-tracing) exist, etc. before the SMCP is instanciated.
* The `StartingCSV` filed may need adjustment for your version of OpenShift. Operators will have no problem updating to the current version from an older StartingCSV. But depending on when you're reading this, it could be that the versions in this repo are already obsolete enough to be removed from the catalogue sources. Generally speaking, its a good practice to instanciate Operators from a version that isn't too old for the versions of OpenShift clusters you are running. Follow the [Installing from OperatorHub using the CLI](https://docs.openshift.com/container-platform/4.13/operators/admin/olm-adding-operators-to-cluster.html#olm-installing-operator-from-operatorhub-using-cli_olm-adding-operators-to-a-cluster) and [Installing a specific version of an Operator](https://docs.openshift.com/container-platform/4.13/operators/admin/olm-adding-operators-to-cluster.html#olm-installing-specific-version-cli_olm-adding-operators-to-a-cluster) for how-to.

## Links

* [Red Hat Developer Guide: OSSM heading to production and Day 2](https://github.com/redhat-developer-demos/ossm-heading-to-production-and-day-2/)
* [Example of Production SMCP sizing](https://github.com/redhat-developer-demos/ossm-heading-to-production-and-day-2/blob/main/scenario-1-kick-off-meeting/README.adoc#prod-environment-service-mesh-ossm-architecture)
* [Custom HealthChecks with ArgoCD](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/#custom-health-checks)
* [An alternative solution for ArgoCD resource health verification](https://blog.stderr.at/openshift/2023-03-20-operator-installation-with-argocd/)
