# Add a CAPx (CAPI) Workload Cluster release (cluster App based)

- [Example](#example)
- [Choose bases](#choose-bases)
- [Create shared cluster template base (optional)](#create-shared-cluster-template-base-optional)
- [Create versioned base (optional)](#create-versioned-base-optional)

Our CAPx (CAPI provider-specific clusters) are delivered by Giant Swarm as a set of two applications. First one
is an App with Cluster instance definition, second one is an App with all the default applications a new cluster
needs in order to run correctly. As such, creating a CAPx cluster means that you need to deliver two correctly
configured App CRs to the Management Cluster.

You can follow the instructions below to store CAPx cluster templates in the repository. The instructions respect the
[repository structure](https://github.com/giantswarm/gitops-template/blob/main/docs/repo_structure.md).

Adding definitions can be done on two levels: shared cluster template and version specific template, see
[create shared template base](#create-shared-template-base-optional)
and [create versioned base](#create-versioned-base-optional).

**IMPORTANT**, CAPx configuration utilizes the
[App Platform Configuration Levels](https://docs.giantswarm.io/app-platform/app-configuration/#levels),
in the following manner:

- cluster templates provide default configuration via App' `config` field,
- cluster instances provide custom configuration via App' `userConfig` field, that is overlaid on top of `config`.

See more about this approach [here](https://github.com/giantswarm/rfc/tree/main/merging-configmaps-gitops).

## Example

An example of a WC cluster template created using the CAPx/CAPI is available in [release_templates/capi/capo](/release_templates/capi/capo/).

## Choose bases

In order to avoid code duplication, it is advised to utilize the
[bases and overlays](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#bases-and-overlays)
of Kustomize in order to configure the cluster.

If the desired base is not there, you can add it. Reference the next sections to get a general idea of how to do it.

## Create shared cluster template base (optional)

*Note: if a template base for your provider is already here, you most probably want to skip this part.*

**IMPORTANT:** cluster release template base cannot serve as a standalone base for cluster creation, it is there only to
abstract App CRs that are common to all clusters versions, and to provide basic configuration for default apps App.
It is then used as a base to other bases, which provide an overlay with a specific configuration. This is to avoid
code duplication across bases.

**Bear in mind**, this is not a complete guide of how to create a perfect base, but rather a mere summary of basic
steps needed to move forward. Hence, instructions here will not always be precise in telling you what to change,
as this can strongly depend on resources involved, how much of them you would like to include into a base, etc.

1. Export provider's and CAPx names you are about to create, for example `capo` and `openstack`:

    ```sh
    export CAPX=capo
    export PROVIDER=openstack
    ```

1. Create a directory structure:

    ```sh
    mkdir -p release_templates/capi/${CAPX}/template
    ```

1. Create cluster App CR template:

    ```sh
    cat <<EOF > release_templates/capi/${CAPX}/template/cluster.yaml
    apiVersion: application.giantswarm.io/v1alpha1
    kind: App
    metadata:
      name: \${cluster_name}
      namespace: org-\${organization}
    spec:
      catalog: giantswarm
      kubeConfig:
        inCluster: true
      name: cluster-${PROVIDER}
      namespace: org-\${organization}
      version: \${cluster_release}
    EOF
    ```

1. Create a default apps App CR template, note the version of the App is set to `0.1.0` by default
   (check with the provider team which version of default apps should you use for your provider):

    ```sh
    cat <<EOF > release_templates/capi/${CAPX}/template/default_apps.yaml
    apiVersion: application.giantswarm.io/v1alpha1
    kind: App
    metadata:
      name: \${cluster_name}-default-apps
      namespace: org-\${organization}
    spec:
      catalog: giantswarm
      config:
        configMap:
          name: \${cluster_name}-default-apps-config
          namespace: org-\${organization}
      kubeConfig:
        inCluster: true
      name: default-apps-${PROVIDER}
      namespace: org-\${organization}
      version: \${default_apps_release:=0.1.0}
    EOF
    ```

1. Create default Apps config, note we do not yet put values in a ConfigMap to have YAML syntax highlighting:

    ```sh
    cat <<EOF > release_templates/capi/${CAPX}/template/default_apps_config.yaml
    clusterName: \${cluster_name}
    organization: \${organization}
    EOF
    ```

1. Create the template's `kustomization.yaml`, note usage of
[ConfigMap Generator](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/configGeneration.md)
for turning config from the previous step into a ConfigMap and placing it under the
[values](https://docs.giantswarm.io/app-platform/app-configuration/#values-format) key:

    ```sh
    cat <<EOF > release_templates/capi/${CAPX}/template/kustomization.yaml
    apiVersion: kustomize.config.k8s.io/v1beta1
    configMapGenerator:
      - files:
        - values=default_apps_config.yaml
        name:  \${cluster_name}-default-apps-config
        namespace: org-\${organization}
    generatorOptions:
      disableNameSuffixHash: true
    kind: Kustomization
    resources:
      - cluster.yaml
      - default_apps.yaml
    EOF
    ```

1. Create the `readme.md` listing variables supported and expected values:

    ```sh
    cat <<EOF > readme.md
    # Input Variables

    Expected variables are in the table below.

    | Variable | Expected Value |
    | :--: | :--: |
    | \`cluster_name\` | Unique name of the Workload Cluster, MUST comply with the [Kubernetes Names](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#names) |
    | \`organization\` | Organization name, the \`org-\` prefix MUST not be part of it and MUST comply with the [Kubernetes Names](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#names) |
    | \`cluster_release\` | Cluster App version, reference the [Cluster Openstack](https://github.com/giantswarm/cluster-openstack/releases) for more insight on releases |
    | \`default_apps_release\` | Defaults Apps App version, reference the [Default Apps for Openstack](https://github.com/giantswarm/default-apps-openstack/releases) for more insight on releases |
    EOF
    ```

## Create versioned base (optional)

**IMPORTANT**, versioned cluster template bases use a shared cluster template base and overlay it with a preferably
generic configuration for a given cluster version. Versioning comes from the fact that `values.yaml` schema may change
over multiple releases,
and although minor differences can be handled on the `userConfig` level, it is advised for the bases to follow major
`values.yaml` schema versions to avoid confusion.

For example, both CAPO bases in this repository, the [v0.5.0](/bases/clusters/capo/<=v0.5.0) and
the [v0.6.0](/bases/clusters/capo/>=v0.6.0), are product of the major changes introduced to the `values.yaml` in
the [cluster-openstack v0.6.0 release](https://github.com/giantswarm/cluster-openstack/releases/tag/v0.6.0).

**IMPORTANT**, despite the below instructions referencing `kubectl-gs` for templating configuration, `kubectl-gs`
generates configuration for the most recent schema only. If you configure a base for older versions of cluster app,
it is advised to check what is generated against the version-specific `values.yaml`.

1. Export CAPx name, provider name and cluster App version you are about to create, for example `capo`, `openstack` and `v0.8.0`:

    ```sh
    export CAPX=capo
    export PROVIDER=openstack
    export CLUSTER_VERSION=v0.8.0
    export DEFAULT_APPS_VERSION=v0.2.0
    ```

1. Create a directory structure:

    ```sh
    mkdir -p release_templates/capi/${CAPX}/${VERSION}
    ```

1. Use the [kubectl gs template cluster](https://docs.giantswarm.io/ui-api/kubectl-gs/template-cluster/) to template
cluster resources, see example for the `openstack` provider below. Use arbitrary values for the mandatory fields, we
will configure them later in our process:

    ```sh
    kubectl gs template cluster \
    --bastion-image aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa \
    --bastion-machine-flavor=n1.tiny \
    --cloud openstack \
    --cloud-config cloud-config \
    --cluster-catalog giantswarm \
    --control-plane-image cccccccc-cccc-cccc-cccc-cccccccccccc \
    --control-plane-machine-flavor n1.small \
    --default-apps-catalog giantswarm \
    --external-network-id bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb \
    --name mywcl \
    --node-cidr 10.6.0.0/24 \
    --organization myorg \
    --provider openstack \
    --worker-failure-domain gb-lon-1 \
    --worker-image cccccccc-cccc-cccc-cccc-cccccccccccc \
    --worker-machine-flavor n1.small \
    --worker-replicas 3 > release_templates/capi/${CAPX}/${VERSION}/cluster.tmp.yaml
    ```

1. Split up the `cluster.yaml` into multiple files:

    ```sh
    COUNT=$(grep -e '---' release_templates/capi/${CAPX}/${VERSION}/cluster.tmp.yaml | wc -l | tr -d ' ')
    csplit release_templates/capi/${CAPX}/${VERSION}/cluster.tmp.yaml /---/ "{$((COUNT-2))}"
    rm release_templates/capi/${CAPX}/${VERSION}/cluster.tmp.yaml
    ```

1. Discard everything except `mywcl-cluster-userconfig` and move it to the base:

    ```sh
    for f in $(ls xx*)
    do
        name=$(yq eval '.metadata.name' $f | tr '[:upper:]' '[:lower:]')
        if [[ "$name" == "mywcl-cluster-userconfig" ]]
        then
            yq eval '.data.values' $f > release_templates/capi/${CAPX}/${VERSION}/cluster_config.yaml
        else
            rm $f
        fi
    done
    ```

1. Replace `mywcl`, `myorg` values from the previous step with variables:

    ```sh
    sed -i "s/myorg/${organization}/g" release_templates/capi/${CAPX}/${VERSION}/cluster_config.yaml
    sed -i "s/mywcl/${cluster_id}/g" release_templates/capi/${CAPX}/${VERSION}/cluster_config.yaml
    ```

1. Compare `cluster_config.yaml` against the version-specific `values.yaml`, and tweak it if necessary to match the
expected schema. At this point you may also provide extra configuration, like additional availability zones, node
pools, etc.:

    ```sh
    wget https://github.com/giantswarm/cluster-openstack/archive/refs/tags/${CLUSTER_VERSION}.tar.gz
    tar -xvf ${CLUSTER_VERSION}.tar.gz cluster-${PROVIDER}-${CLUSTER_VERSION:1}/helm/cluster-${PROVIDER}/values.yaml
    vim cluster-openstack-${CLUSTER_VERSION:1}/helm/cluster-${PROVIDER}/values.yaml
    ```

1. Create a patch for the cluster App CR to provide the newly created configuration:

    ```sh
    cat <<EOF > release_templates/capi/${CAPX}/${VERSION}/patch_config.yaml
    apiVersion: application.giantswarm.io/v1alpha1
    kind: App
    metadata:
      name: \${cluster_name}
      namespace: org-\${organization}
    spec:
      config:
        configMap:
          name: \${cluster_name}-config
          namespace: org-\${organization}
    EOF
    ```

1. Create the `kustomization.yaml`, referencing the template, and generating the ConfigMap out of `cluster_config.yaml`:

    ```sh
    cat <<EOF > release_templates/capi/${CAPX}/${VERSION}/kustomization.yaml
    apiVersion: kustomize.config.k8s.io/v1beta1
    configMapGenerator:
      - files:
        - values=cluster_config.yaml
        name: \${cluster_name}-config
        namespace: org-\${organization}
    generatorOptions:
      disableNameSuffixHash: true
    kind: Kustomization
    patchesStrategicMerge:
      - patch_config.yaml
    resources:
      - ../template
    EOF
    ```

1. Copy `readme.md` from the template base:

    ```sh
    cp release_templates/capi/${CAPX}/template/readme.md release_templates/capi/${CAPX}/${VERSION}/readme.md
    ```
