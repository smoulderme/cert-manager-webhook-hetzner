# ACME webhook for Hetzner DNS API

This solver can be used when you want to use cert-manager with Hetzner DNS API. API documentation is [here](https://dns.hetzner.com/api-docs)

## Requirements
-   [go](https://golang.org/) >= 1.13.0
-   [helm](https://helm.sh/) >= v3.0.0
-   [kubernetes](https://kubernetes.io/) >= v1.14.0
-   [cert-manager](https://cert-manager.io/) >= 0.12.0

## Installation

### cert-manager

Follow the [instructions](https://cert-manager.io/docs/installation/) using the cert-manager documentation to install it within your cluster.

### Webhook

#### Using public helm chart
```bash
helm repo add cert-manager-webhook-hetzner https://smoulderme.github.io/cert-manager-webhook-hetzner/
helm upgrade -i --namespace cert-manager cert-manager-webhook-hetzner cert-manager-webhook-hetzner/cert-manager-webhook-hetzner
```

#### From local checkout

```bash
helm install --namespace cert-manager cert-manager-webhook-hetzner deploy/cert-manager-webhook-hetzner
```
**Note**: The kubernetes resources used to install the Webhook should be deployed within the same namespace as the cert-manager.

To uninstall the webhook run
```bash
helm uninstall --namespace cert-manager cert-manager-webhook-hetzner
```

## Issuer

Create a `ClusterIssuer` or `Issuer` resource.
The **groupName** must be the same as in the Helm Chart (see [deploy/cert-manager-webhook-hetzner/values.yaml](deploy/cert-manager-webhook-hetzner/values.yaml) for the details):  
```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # The ACME server URL for staging:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # If you want to use production ACME server URL: 
    # server: https://acme-v02.api.letsencrypt.org/directory

    # Email address used for ACME registration
    email: mail@example.com # REPLACE THIS WITH YOUR EMAIL!!!

    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging

    solvers:
      - dns01:
          webhook:
            groupName: acme.yourdomain.here
            solverName: hetzner
            config:
              secretName: hetzner-secret
              zoneName: example.com
              apiUrl: https://dns.hetzner.com/api/v1
```

### Credentials
In order to access the Hetzner API, the webhook needs an API token. You can get the API token in the [Hetzner DNS Web Console](https://dns.hetzner.com/settings/api-token).  
If you choose another name for the secret than `hetzner-secret`, ensure you modify the value of `secretName` in the `[Cluster]Issuer` and change it in the Helm Chart values.  
The secret for the example above will look like this:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hetzner-secret
type: Opaque
data:
  api-key: your-key-base64-encoded
```

### Create a certificate

Finally you can create certificates, for example:

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: example-cert
  namespace: cert-manager
spec:
  commonName: example.com
  dnsNames:
    - example.com
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  secretName: example-cert
```

## Development

### Running the test suite

All DNS providers **must** run the DNS01 provider conformance testing suite,
else they will have undetermined behaviour when used with cert-manager.

**It is essential that you configure and run the test suite when creating a
DNS01 webhook.**

First, you need to have Hetzner account with access to DNS control panel. You need to create API token and have a registered and verified DNS zone there.
Then you need to replace `zoneName` parameter at `testdata/hetzner/config.json` file with actual one.
You also must encode your api token into base64 and put the hash into `testdata/hetzner/hetzner-secret.yml` file.

You can then run the test suite with:

```bash
# first install necessary binaries (only required once)
./scripts/fetch-test-binaries.sh
# then run the tests
TEST_ZONE_NAME=example.com. make verify
```
