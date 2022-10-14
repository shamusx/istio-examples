# Demo PKI Infra with Vault+Cert-Manager

## Scenario

Vault will act as the existing PKI infrastructure the cert-manager has to talk to in order to obtain intermediate CA cert for istio to sign workload certs

## Summary

What is being done

- Setup Vault as root CA
- Setup Certmanager to acquire istio intermediate ca
- Setup istio 

## Setup Vault

### Install Vault with Helm

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# External vault for multi-cluster
# helm upgrade -i vault hashicorp/vault --set "ui.enabled=true" --set "ui.serviceType=LoadBalancer" --set "server.service.type=LoadBalancer"  --set="server.dev.enabled=true"

helm install vault hashicorp/vault --set="server.dev.enabled=true" -n vault-demo --create-namespace
```

Once install take note of IP for vaul svc(if)


### Configure Vault
1. Enter the vault shell
```bash 
# enter vault shell
kubectl exec -it vault-0 -n vault-demo -- /bin/sh

#Prompt should change to "/ $" 
/ $ 
```

2. Setup selfsigned CA (Demo)
```bash vault
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki

vault write pki/root/generate/internal \
    common_name="rootca-demo" \
    issuer_name="rootca-demo" \
    ttl=8760h

# vault.vault-demo is svc.namespace
vault write pki/config/urls \
    issuing_certificates="http://vault.vault-demo/v1/pki/ca" \
    crl_distribution_points="http://vault.vault-demo:8200/v1/pki/crl"

vault write pki/roles/cert-manager \
    allowed_domains="svc" \
    require_cn=false \
    allow_subdomains=true \
    max_ttl=720h

# this path will permit intermediate ca cert signing
vault policy write cert-manager - <<EOF
path "pki/root/sign-intermediate" { capabilities = ["create", "update"] }
EOF

vault write auth/approle/role/cert-manager token_policies="cert-manager" token_ttl=1h token_max_ttl=2h
```

Collect following details

### role-id
```bash
vault read auth/approle/role/cert-manager/role-id

Key        Value
---        -----
role_id    1b5e46af-047b-eb54-f2b3-7c1c869a1b9f
```
### secret_id

```bash
vault write -force auth/approle/role/cert-manager/secret-id

Key                   Value
---                   -----
secret_id             7f280b87-3ac5-6d52-98d2-a2bb2bef82fc
secret_id_accessor    cf804056-44aa-a0fe-d028-0cf26c707084
secret_id_ttl         0s
```

## Install Cert Manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.yaml
```

## Configure Issuer and Certs
<br>

### Setup Secret using `secret_id`

1. Place `secret_id` in a secret
```bash
kubectl create ns istio-system
# Will use my secret_id collected above as example
echo 7f280b87-3ac5-6d52-98d2-a2bb2bef82fc | base64 

#Returns the following
N2YyODBiODctM2FjNS02ZDUyLTk4ZDItYTJiYjJiZWY4MmZjCg==
```
2. Take secret and create a secret with it
```bash
cat << EOF | kubectl apply -n istio-system -f -
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: cert-manager-vault-approle
  namespace: istio-system
data:
  secretId: N2YyODBiODctM2FjNS02ZDUyLTk4ZDItYTJiYjJiZWY4MmZjCg==
EOF
```
<br>

### Create cert-manager Issuer

1. Will use `role_id` collected above as `roleID` below
Note: if following this demo properties should be similar
```bash
cat << EOF | kubectl apply -f -n istio-system -f -
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: vault-issuer
  namespace: istio-system
spec:
  vault:
    path: pki/root/sign-intermediate
    server: http://vault.vault-demo:8200
    auth:
      appRole:
        path: approle
        roleId: 1b5e46af-047b-eb54-f2b3-7c1c869a1b9f
        secretRef:
          name: cert-manager-vault-approle
          key: secretId
EOF
```

2. Verify Vault is in a good status

```bash 
kubectl get issuer -o wide -n istio-system
NAME           READY   STATUS           AGE
vault-issuer   True    Vault verified   7m28s
```

### Create istiod cacert

```bash
cat << EOF | kubectl apply -f -
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cacerts
  namespace: istio-system
spec:
  secretName: cacerts
  duration: 720h 
  renewBefore: 360h
  commonName: istiod.istio-system.svc
  isCA: true
  usages:
    - digital signature
    - key encipherment
    - cert sign
  dnsNames:
    - istiod.istio-system.svc
  issuerRef:
    name: vault-issuer
    kind: Issuer
    group: cert-manager.io
EOF
```

## Install Istio

Note: recommend using version 1.14.2+ as istiod can read `kubernetes.io/tls` Secrets after this release

```bash
cat << EOF | istioctl apply --skip-confirmation -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: demo-istio-install
  namespace: istio-system
spec:
  profile: demo
  components:
    pilot:
      k8s:
        env:
        - name: AUTO_RELOAD_PLUGIN_CERTS
          value: "true"
EOF
```

Looking at istiod logs should reveal 2 things
1. tls secret type is used (which is what cert-maanger uses)
2. rootca-demo we created in vault is the CA that signed the istiod intermediate CA
```bash
kubectl logs -l app=istiod -n istio-system -f

2022-10-14T20:31:28.354369Z     info    Using kubernetes.io/tls secret type for signing ca files
2022-10-14T20:31:28.354391Z     info    Use plugged-in cert at etc/cacerts/tls.key
2022-10-14T20:31:28.354634Z     info    x509 cert - Issuer: "CN=istiod.istio-system.svc", Subject: "", SN: 352b46ad9241a50c1eae296a895d84ac, NotBefore: "2022-10-14T20:29:28Z", NotAfter: "2032-10-11T20:31:28Z"
2022-10-14T20:31:28.354682Z     info    x509 cert - Issuer: "CN=rootca-demo", Subject: "CN=istiod.istio-system.svc", SN: 4fac40d5a1e2f88a8aae6e075474e349adf2f4dc, NotBefore: "2022-10-14T20:17:42Z", NotAfter: "2022-11-13T20:18:12Z"
2022-10-14T20:31:28.354717Z     info    x509 cert - Issuer: "CN=rootca-demo", Subject: "CN=rootca-demo", SN: 6c6f85ac4cf3727b0e0429b3ade92c1be0c523cf, NotBefore: "2022-10-14T19:56:53Z", NotAfter: "2023-10-14T19:57:23Z"
```