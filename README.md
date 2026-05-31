# Cloudflare Access Operator for Kubernetes

This is a Kubernetes Operator that manages Cloudflare Zero Trust Access Applications natively from within your cluster. You define your SSO rules, identity providers, and IP bypasses in a simple Custom Resource, and the operator ensures Cloudflare matches it. If someone deletes the resource, the operator tears down the application in Cloudflare. 

By moving this logic directly into a Kubernetes Operator, there is no need to rely on third-party automation pipelines or external orchestration tools. Your cluster's etcd becomes the single source of truth, managing your edge security continuously alongside your deployments.

### Why this Operator?

* **Native & Self-Contained:** No external orchestration required. It uses pure Kubernetes reconciliation directly against the Cloudflare v4 REST API.
* **Perfect Companion:** Works exceptionally well alongside the [STRRL/cloudflare-tunnel-ingress-controller](https://github.com/STRRL/cloudflare-tunnel-ingress-controller) to provide a complete, declarative Cloudflare Edge-to-Pod routing and access solution.
* **Smart Secret Resolution:** You don't need to hardcode namespaces or map complex config maps. Provide a `secret_ref` (like `security/cf-api`) or just a `tenant` name, and the operator natively cascades through namespaces to find your API tokens. It even supports legacy Cloudflare Ingress Controller secret formats out-of-the-box.
* **Pure Helm, No Kustomize:** We stripped out all the Operator SDK Kustomize overhead. It deploys via a clean, standard OCI Helm chart.
* **Multi-Arch Native:** Built for both `amd64` and `arm64`. Runs flawlessly on standard cloud nodes, Apple Silicon, or lightweight ARM test clusters.
* **Deep JSON Payload Mapping:** Supports highly complex, deeply nested Cloudflare Access policies (Azure AD groups, IP blocks, etc.) without rigid YAML validation getting in your way.

## Installation

The operator is packaged as an OCI artifact. You do not need to clone this repository to install it.

By default, the operator will watch all namespaces, but you can restrict it to a specific namespace for strict RBAC environments.

```bash
# Install the operator via OCI
helm upgrade --install cloudflare-operator oci://ghcr.io/k8stooling/charts/cloudflare-operator 
```

(Optional) To restrict the operator to a single namespace:

```bash
helm upgrade --install cloudflare-operator oci://ghcr.io/k8stooling/charts/cloudflare-operator \
  --set watchNamespace="cfdev"
```

## Usage

## The Credentials Secret
The operator needs a Cloudflare API Token and Account ID. You can drop this secret in the namespace where your Custom Resource lives, or let the operator fall back to its own system namespace.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cf-credentials-xyz
stringData:
  API_TOKEN: "your-super-secret-api-token"
  ACCOUNT_ID: "your-cloudflare-account-id"
```


## The Custom Resource

Here is a very simple MVP application. This creates an Access Application that bypasses SSO and only allows traffic from a specific IP block.

```yaml
apiVersion: cloudflare.k8stooling.io/v1alpha1
kind: AccessApplication
metadata:
  name: echo-test-app
spec:
  tenant: xyz # Resolves to secret: cf-credentials-xyz
  domain: "echo.xksh.org"
  destinations:
    - type: "public"
      uri: "echo.xksh.org"
      zone_name: "xksh.org"
  allowed_idps: [] # Empty because we use a bypass policy below
  policies:
    - name: "Allow My Test IP Only"
      decision: "bypass"
      precedence: 1
      include:
        - ip: { ip: "203.0.113.50/32" }
```

Once applied, check the status of your resource to see the live Cloudflare App ID, the sync state, and the last run timestamp:

```bash
kubectl get accessapplication echo-test-app -o yaml
```

In order to be able easily identify applications on Cloudflare the generated application will hold the name {{ tenant }}-{{ metadate.namespace }}-{{ metadata.name }}.

# Development

This operator is built on the Operator SDK using the Ansible plugin. It relies on standard Ansible action modules and the kubernetes.core collection.

To build and push a new multi-arch image locally:

```bash
make docker-buildx IMG=ghcr.io/k8stooling/cloudflare-operator:latest
```

(Note: The GitHub Actions pipeline is configured to automatically cross-compile and publish both the container image and the OCI Helm chart to GHCR).

# Acknowledgments

The architecture and design of this operator were created by the author, with Google Gemini serving as an intelligent pair-programmer/StackOverflow replacement during the build process.