# Kong Helm OpenShift Parent

Parent Helm chart, which subcharts the Kong official one, for proper installation on OpenShift.

This chart wraps the [existing Kong Helm chart](https://github.com/Kong/charts/tree/main/charts/kong).

> :warning: This has been tested exclusively with **Helm 3.7** - this is the recommended version unless it is otherwise impossible to change your Helm installation.

# Pre-Installation

**Before** installing the chart, you should follow the same instructions [in that chart's README](https://github.com/Kong/charts/blob/main/charts/kong/README.md), including setting up cluster certificates and licenses if necessary, and then return here for final steps.

Create a `kong-values.yaml` file as described in the Kong Helm chart's README, but bear in mind the following required changes...

## Changes to Sub-Chart Values

This installation uses the Kong chart as a [subchart](https://helm.sh/docs/chart_template_guide/subcharts_and_globals/).

You need to alter your configuration values to have them applied to the subchart instead. To do this:

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
  tls:
    enabled: true
    servicePort: 8005
    containerPort: 8005
...
...
```

you must indent the entire block, under the root object 'kong', like this:

```yaml
kong:
  cluster:
    enabled: true
    tls:
      enabled: true
      servicePort: 8005
      containerPort: 8005
  ...
  ...
```

# Install the Chart

> :warning: When installing on OpenShift or OKD, you **must disable custom resource definitions globally**. To do this, during an installation you just need to add the follow command arguments: `--skip-crds --set ingressController.installCRDs=false`

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

**6. Load the dependent charts**
```sh
helm dependency update
```

**7. Create/copy your configuration values file `kong-values.yaml` from the pre-installation into this directory**

**8. Install the chart:**
```sh
helm install -f kong-values.yaml --namespace kong --skip-crds --set ingressController.installCRDs=false --generate-name .
```

# Extra Configuration

There are OpenShift-specific objects in this parent chart, that help in the installation and management of Kong.

These have their own specific configuration values, which can be customised:

# Limitations

- If you want to run the Kong Admin API and the Kong Manager UI on the same router, but at different hostnames, in pass-through mode (e.g. https://manager.konghq.com and https://api.konghq.com) then **you need to provision separate certificates for each URL** - wildcards will **not work**
- The Kong Ingress Controller cannot be installed on OpenShift yet, because the service account is unable to get permissions to watch multiple projects (namespaces)
- UDP routes are not supported over the OpenShift Router - you'll have to manually create an OpenShift Service on a free NodePort, and manually provision some UDP load balancer product (AWS NLB for example)
