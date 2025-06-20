# Running Red Hat OpenShift Service Mesh (OSSM) 2 and OSSM 3 side by side
This section describes how to run OSSM 2 and OSSM 3 side by side in one cluster without interfering with each other.
> **_NOTE:_** It's not OSSM 2 -> OSSM 3 migration guide.

To better understand the steps described here, it's recommended to read the OSSM 2 [deployment models](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/service_mesh/service-mesh-2-x#ossm-deployment-models) and upstream Istio [Multiple Control Planes](https://istio.io/latest/docs/setup/install/multiple-controlplanes/) documentation. This document assumes OSSM 2.6 is already running and the goal is to install OSSM 3.0 on the same cluster without interfering with the OSSM 2.6 installation.

Follow the steps described below for your OSSM 2 deployment model (Multi-tenant or Cluster-Wide).

## Multitenant (default) deployment model
There are no changes required on OSSM 2 side. For OSSM 3 installation, see  [OSSM 3 installation](#ossm-3-installation) below.

## Cluster-Wide (Single Tenant) mesh deployment model
### Required OSSM 2 changes for Cluster-Wide model
To be able to run OSSM 2 in cluster-wide mode with OSSM 3, it's required to make configuration changes on OSSM 2 side as well.
As the control plane with the cluster-wide model watches all namespaces, it's required to restrict that to namespaces only belonging to the OSSM 2 mesh. Otherwise it would conflict with OSSM 3. Restricting your control plane can be done either with [discovery selectors](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/service_mesh/service-mesh-2-x#ossm-excluding-namespaces-from-cluster-wide-mesh-console_ossm-deployment-models) or by switching to a multi-tenant deployment model.
1. configure discovery selectors in your SMCP CR:
    ```yaml
    apiVersion: maistra.io/v2
    kind: ServiceMeshControlPlane
    metadata:
      name: basic
      namespace: istio-system
    spec:
      policy:
        type: Istiod
      addons:
        grafana:
          enabled: false
        kiali:
          enabled: true
        prometheus:
          enabled: true
      telemetry:
        type: Istiod
      version: v2.6
      mode: ClusterWide
      meshConfig:
        discoverySelectors:
          - matchExpressions:
            - key: maistra.io/member-of
              operator: Exists
      runtime:
        components:
          pilot:
            container:
              env:
                ENABLE_ENHANCED_RESOURCE_SCOPING: 'true'
    ```

### OSSM 3 installation
See OSSM [installation](https://docs.openshift.com/service-mesh/3.0.0tp1/install/ossm-installing-openshift-service-mesh.html) documentation for more information.
1. install `Red Hat OpenShift Service Mesh 3` operator
1. create `IstioCNI` resource in `istio-cni` namespace
1. create following `Istio` resource in `istio-system3` (must be a different namespace than a namespace running OSSM 2). Make sure to use discovery selectors which are ignoring OSSM 2 namespaces and NOT to use `default` name for `Istio` resource. You can optionally restrict discovered namespaces even more. Configuration shown in the example only ignores OSSM 2 namespaces but all other namespaces will be part of OSSM 3 mesh. See upstream Istio documentation for [discoverySelectors](https://istio.io/v1.19/docs/setup/install/multiple-controlplanes/) usage.
    ```yaml
    kind: Istio
    apiVersion: sailoperator.io/v1alpha1
    metadata:
      name: ossm3
    spec:
      namespace: istio-system3
      values:
        meshConfig:
          discoverySelectors:
            - matchExpressions:
              - key: maistra.io/member-of
                operator: DoesNotExist
      updateStrategy:
        type: InPlace
      version: v1.23.0
    ```
1. Deploy your workloads and label the namespaces with `istio.io/rev=ossm3` label
    > **_NOTE:_** Do NOT use `istio-injection=enabled` label for your OSSM 3 workload namespaces unless you have changed `spec.memberSelectors` in `ServiceMeshMemberRoll`.
1. You can use the `istioctl ps` command to confirm that the application workloads are managed by their respective control plane:
    ```sh
    $ istioctl ps -i istio-system
    NAME                                          CLUSTER        CDS        LDS        EDS        RDS        ECDS         ISTIOD                                          VERSION
    details-v1-7f46897b-88x4l.bookinfo            Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-install-istio-system-bd58bdcd5-2htkf     1.20.8
    mongodb-v1-6cf7dc9885-7nlmq.bookinfo          Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-install-istio-system-bd58bdcd5-2htkf     1.20.8
    mysqldb-v1-7c4c44b9b4-22b57.bookinfo          Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-install-istio-system-bd58bdcd5-2htkf     1.20.8
    productpage-v1-6f9c6589cb-l6rvg.bookinfo      Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-install-istio-system-bd58bdcd5-2htkf     1.20.8
    ratings-v1-559b64556-f6b4l.bookinfo           Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-install-istio-system-bd58bdcd5-2htkf     1.20.8
    ratings-v2-8ddc4d65c-bztrg.bookinfo           Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-install-istio-system-bd58bdcd5-2htkf     1.20.8
    ratings-v2-mysql-cbc957476-m5j7w.bookinfo     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-install-istio-system-bd58bdcd5-2htkf     1.20.8
    reviews-v1-847fb7c54d-7dwt7.bookinfo          Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-install-istio-system-bd58bdcd5-2htkf     1.20.8
    reviews-v2-5c7ff5b77b-5bpc4.bookinfo          Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-install-istio-system-bd58bdcd5-2htkf     1.20.8
    reviews-v3-5c5d764c9b-mk8vn.bookinfo          Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-install-istio-system-bd58bdcd5-2htkf     1.20.8
    ```
    ```sh
    $ istioctl ps -i istio-system3
    NAME                                          CLUSTER        CDS                LDS                EDS                RDS                ECDS        ISTIOD                            VERSION
    details-v1-57f6466bdc-5krth.bookinfo2         Kubernetes     SYNCED (2m40s)     SYNCED (2m40s)     SYNCED (2m34s)     SYNCED (2m40s)     IGNORED     istiod-ossm3-5b46b6b8cb-gbjx6     1.23.0
    productpage-v1-5b84ccdddf-f8d9t.bookinfo2     Kubernetes     SYNCED (2m39s)     SYNCED (2m39s)     SYNCED (2m34s)     SYNCED (2m39s)     IGNORED     istiod-ossm3-5b46b6b8cb-gbjx6     1.23.0
    ratings-v1-fb764cb99-kx2dr.bookinfo2          Kubernetes     SYNCED (2m40s)     SYNCED (2m40s)     SYNCED (2m34s)     SYNCED (2m40s)     IGNORED     istiod-ossm3-5b46b6b8cb-gbjx6     1.23.0
    reviews-v1-8bd5549cf-xqqmd.bookinfo2          Kubernetes     SYNCED (2m40s)     SYNCED (2m40s)     SYNCED (2m34s)     SYNCED (2m40s)     IGNORED     istiod-ossm3-5b46b6b8cb-gbjx6     1.23.0
    reviews-v2-7f7cc8bf5c-5rvln.bookinfo2         Kubernetes     SYNCED (2m40s)     SYNCED (2m40s)     SYNCED (2m34s)     SYNCED (2m40s)     IGNORED     istiod-ossm3-5b46b6b8cb-gbjx6     1.23.0
    reviews-v3-84f674b88c-ftcqg.bookinfo2         Kubernetes     SYNCED (2m40s)     SYNCED (2m40s)     SYNCED (2m34s)     SYNCED (2m40s)     IGNORED     istiod-ossm3-5b46b6b8cb-gbjx6     1.23.0
    ```