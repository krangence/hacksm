# kube-system

## cifs-pv

Using the [CIFS volume driver](https://k8scifsvol.juliohm.com.br/)

cifs-based persistent mounts for pod volumes

* [cifs-pv/](cifs-pv/)

## ingress-nginx

![](https://i.imgur.com/b21MHEE.png)

ingress-nginx controller leveraging cert-manager as the central cert store for the wildcard certificate

* [ingress-nginx/](ingress-nginx/)

## metallb

[Run your own on-prem LoadBalancer](https://metallb.universe.tf/)

* [metallb/metallb.yaml](metallb/metallb.yaml)

## oauth2-proxy

[OAuth2 authenticating proxy](https://github.com/pusher/oauth2_proxy) leveraging Auth0

* [oauth2-proxy/](oauth2-proxy/)

## vault

[vault-helm chart](https://github.com/hashicorp/vault-helm) deployed in HA mode leveraging consul as the storage backend

### Vault HA server

* [vault/vault.yaml](vault/vault.yaml)

See the [vault/vault.yaml](vault/vault.yaml) & [../setup/bootstrap-vault.sh](../setup/bootstrap-vault.sh) files for reference on how these are implemented in this cluster.  The server leverages the Google KMS keystore for automatic unseal as needed.  Further information about Google KMS for unsealing are [Vault GCPKMS Documentation](https://www.vaultproject.io/docs/configuration/seal/gcpckms.html), [Autounseal with GCP KMS](https://learn.hashicorp.com/vault/operations/autounseal-gcp-kms), & [Authenticating to Google Cloud Platform](https://cloud.google.com/kubernetes-engine/docs/tutorials/authenticating-to-cloud-platform) for passing service account json via a secret.

## vault-secrets-operator

[vault-secrets-operator](https://github.com/ricoberger/vault-secrets-operator)

* [vault-secrets-operator/vault-secrets-operator.yaml](vault-secrets-operator/vault-secrets-operator.yaml)

### Setup

The setup is automatically handled during cluster bootstrapping.  See [bootstrap-vault.sh](../setup/bootstrap-vault.sh) for more detail.

If configuring manually, follow the [vault-secrets-operator guide](https://github.com/ricoberger/vault-secrets-operator/blob/master/README.md) which is mostly the following:

```shell
# if not logged in to vault already:
kubectl -n kube-system port-forward svc/vault 8200:8200 &
export VAULT_ADDR='http://127.0.0.1:8200'
vault login <root token>

# enable kv secrets type
vault secrets enable -path=secrets -version=1 kv

# create read-only policy for kubernetes
cat <<EOF | vault policy write vault-secrets-operator -
path "secrets/*" {
  capabilities = ["read"]
}
EOF

export VAULT_SECRETS_OPERATOR_NAMESPACE=$(kubectl -n kube-system get sa vault-secrets-operator -o jsonpath="{.metadata.namespace}")
export VAULT_SECRET_NAME=$(kubectl -n kube-system get sa vault-secrets-operator -o jsonpath="{.secrets[*]['name']}")
export SA_JWT_TOKEN=$(kubectl -n kube-system get secret $VAULT_SECRET_NAME -o jsonpath="{.data.token}" | base64 --decode; echo)
export SA_CA_CRT=$(kubectl -n kube-system get secret $VAULT_SECRET_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)
export K8S_HOST=$(kubectl -n kube-system config view --minify -o jsonpath='{.clusters[0].cluster.server}')

# Verify the environment variables
env | grep -E 'VAULT_SECRETS_OPERATOR_NAMESPACE|VAULT_SECRET_NAME|SA_JWT_TOKEN|SA_CA_CRT|K8S_HOST'

vault auth enable kubernetes

# Tell Vault how to communicate with the Kubernetes cluster
vault write auth/kubernetes/config \
  token_reviewer_jwt="$SA_JWT_TOKEN" \
  kubernetes_host="$K8S_HOST" \
  kubernetes_ca_cert="$SA_CA_CRT"

# Create a role named, 'vault-secrets-operator' to map Kubernetes Service Account to Vault policies and default token TTL
vault write auth/kubernetes/role/vault-secrets-operator \
  bound_service_account_names="vault-secrets-operator" \
  bound_service_account_namespaces="$VAULT_SECRETS_OPERATOR_NAMESPACE" \
  policies=vault-secrets-operator \
  ttl=24h
```
