# Kong Helm OpenShift Parent

Parent Helm chart, which subcharts the Kong official one, for proper installation on OpenShift.

This chart wraps the [existing Kong Helm chart](https://github.com/Kong/charts/tree/main/charts/kong).

# Pre-Installation

**Before** installing the chart, you should follow the same instructions [in that chart's README](https://github.com/Kong/charts/blob/main/charts/kong/README.md), including setting up cluster certificates and licenses if necessary, and then return here for final steps.

Create a `kong-values.yaml` file as described in the Kong Helm chart's README.

# Installation

## Sub-Chart Values

This installation uses the Kong chart as a [subchart](https://helm.sh/docs/chart_template_guide/subcharts_and_globals/).

You need to alter your configuration values to have them applied to the subchart instead.

To do this:

### Inline Settings

For any inline value setting, e.g:

```sh
helm install --set cluster.enabled=true ........
```

you must prepend with `kong.`, like this:

```sh
helm install --set kong.cluster.enabled=true ........
```

### Values YAML

For all values in a values YAML file, e.g:

```yaml
cluster:
  enabled: true
```

you must indent the entire block, under the root object 'kong', like this:

```yaml
kong:
  cluster:
    enabled: true
```

## Install the Chart

> :warning: When installing on OpenShift or OKD, you **must disable custom resource definitions globally**. To do this, during an installation you just need to add the follow command arguments:
```sh
--skip-crds --set ingressController.installCRDs=false
```

**1. Login to OpenShift as a cluster administrator, or other privileged user:**
```sh
oc login ...
```

**2. Create a dedicated Kong project (namespace):**
```sh
oc new-project kong
```

**3. Grant that project's pods permission to run as any UID:**

```sh
oc adm policy add-scc-to-user privileged -z default -n kong
```

**4. Clone this chart:**
```sh
git clone https://github.com/ttyS0e/kong-helm-openshift-parent.git
```

**5. Change directory:**
```sh
cd kong-helm-openshift-parent
```

**6. Create/copy your configuration values file `kong-values.yaml` from the pre-installation into this directory**

**7. Install the chart:**
```sh
helm install -f kong-values.yaml --namespace kong --skip-crds --set ingressController.installCRDs=false --generate-name .
```

# Extra Configuration

There are OpenShift-specific objects in this parent chart, that help in the installation and management of Kong.

These have their own specific configuration values, which can be customised:

