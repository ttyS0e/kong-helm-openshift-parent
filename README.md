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

# Extra OpenShift Features

> :warning: Remember that if you are exposing the admin API, manager UI, portal, or portal API using OpenShift Routes, you need to set their corresponding "kong.env.*" parameter too! See the table for reference.

There are OpenShift-specific objects in this parent chart, that help in the installation and management of Kong.

These have their own specific configuration values, which can be customised:

| Value                                | Description                                                                                                                                                                                           | Type    | Default | Example                       |
|--------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------|---------|-------------------------------|
| openshift.routes.{i}.enabled         | Creates an OpenShift route for the given service. By default, this uses the router's hostname and certificate, and terminates TLS, passing through to the equivalent plaintext port on the container. | boolean | `false` | `true`                        |
| openshift.routes.{i}.tls.enabled     | Whether TLS is enabled on the OpenShift route. This is mostly `true` by default, but in some cases can be changed to force plaintext HTTP connections.                                                | boolean | `true`  | `true`                        |
| openshift.routes.{i}.tls.termination | OpenShift TLS termination strategy - for most routes, this can be customised. Choices are `edge`, `passthrough`, `reencrypt`.                                                                         | string  | `edge`  | `passthrough`                 |
| openshift.routes.{i}.hostname        | Custom hostname to use for the route.                                                                                                                                                                 | string  | `""`    | `"portal-api.mydomain.mytld"` |

## Example Block

To configure custom route hostnames and certificates to manage Kong API, set up the following in the values YAML file:

```yaml
openshift:
  routes:
    manager:
      enabled: true
      hostname: "manager.kong.mydomain.tld"
      tls:
        termination: "passthrough"
    admin:
      enabled: true
      hostname: "admin-api.kong.mydomain.tld"
      tls:
        termination: "passthrough"
kong:
  env:
    admin_api_uri: "https://admin-api.kong.mydomain.tld"
    admin_gui_url: "https://manager.kong.mydomain.tld"
    admin_ssl_cert: "/etc/secrets/kong-ssl-cert/api-cert"
    admin_ssl_cert_key: "/etc/secrets/kong-ssl-cert/api-key"
    admin_gui_ssl_cert: "/etc/secrets/kong-ssl-cert/manager-cert"
    admin_gui_ssl_cert_key: "/etc/secrets/kong-ssl-cert/manager-key"
  secretVolumes:
  - kong-ssl-cert
```

Ensure you have these custom certificates created in the `kong` Project already:

```sh
oc create secret generic kong-ssl-cert --from-file=api-cert=./admin-api.kong.mydomain.tld.pem --from-file=api-key=./admin-api.kong.mydomain.tld.key --from-file=manager-cert=./manager.kong.mydomain.tld.pem --from-file=manager-key=./manager.kong.mydomain.tld.key
```

Then, you can access the Kong Manager UI at: `https://manager.kong.mydomain.tld` (if your DNS points to the OpenShift Router load balancer).

# Limitations

- If you want to run the Kong Admin API and the Kong Manager UI on the same router, but at different hostnames, in pass-through mode (e.g. https://manager.konghq.com and https://api.konghq.com) then **you need to provision separate certificates for each URL** - wildcards will **not work**
- The Kong Ingress Controller cannot be installed on OpenShift yet, because the service account is unable to get permissions to watch multiple projects (namespaces)
- UDP routes are not supported over the OpenShift Router - you'll have to manually create an OpenShift Service on a free NodePort, and manually provision some UDP load balancer product (AWS NLB for example)
