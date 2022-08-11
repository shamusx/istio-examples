# [Istio] How to enable Plug-In CA Rotation

A common topic when setting up Istio is leveraging existing PKI and the intermediate CA for Istio to sign workload certificates.  But the question comes for short-lived intermediate CA's Istio will need be aware of the change, also fast forward to multiple Istio clusters this will become a handful to manage.
<br>
<br>
We will make run through how to make Istio aware of changes to Intermediate CA used to sign workload certificates.  As well as have `cert-manager` renew the Intermediate CA before it expires.

## Prepare
What will be needed:
- Kubernetes Cluster (minikube is also an option)
- istioctl (version >=1.14.2)
- cert-manager (https://cert-manager.io/docs/)

## Setup Cert Manager
1. Install `cert-manager`

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.2/cert-manager.yaml
```

2. For simplicity we will setup a selfsigned CA, but with `cert-manager` we could have used an alternate Issuer leveraging existing PKI.
```
cat << EOF | kubectl apply -f -
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned
  namespace: cert-manager
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-ca
  namespace: cert-manager
spec:
  isCA: true
  duration: 21600h # 900d
  secretName: selfsigned-ca
  commonName: certmanager-ca
  subject:
    organizations:
      - cert-manager
  issuerRef:
    name: selfsigned
    kind: Issuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-ca
spec:
  ca:
    secretName: selfsigned-ca
EOF
```

3. Setup the Intermediate CA istio will use to sign workload certificates - with rotation set to occur every 60days(1440h) and renew before 15days(360h) of expiry
```
kubectl create namespace istio-system
cat << EOF | kubectl apply -f -
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cacerts
  namespace: istio-system
spec:
  secretName: cacerts
  duration: 1440h
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
    name: selfsigned-ca
    kind: ClusterIssuer
    group: cert-manager.io
EOF
```

Note: In Istio Release 1.4.2(https://istio.io/latest/news/releases/1.14.x/announcing-1.14.2/#changes) Istiod digest kubernetes.io/tls type secrets

## Setup Istio
Install Istio and set the environment variable `AUTO_RELOAD_PLUGIN_CERTS=true`

```
istioctl operator init

cat << EOF | kubectl apply -f -
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

## Intermediate Certificate Rotation
1. Lets say requirements have changed, previously we set `duration` to 60days(1440h) and it now must rotated every 30days(720h)

```
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
    name: selfsigned-ca
    kind: ClusterIssuer
    group: cert-manager.io
EOF
```

2. Watching the logs should reveal CA change
```
k logs -l app=istiod -n istio-system -f
```

Log to look for:
```
2022-08-11T20:18:41.493247Z	info	Update Istiod cacerts
2022-08-11T20:18:41.493483Z	info	Using kubernetes.io/tls secret type for signing ca files
2022-08-11T20:18:41.716843Z	info	Istiod has detected the newly added intermediate CA and updated its key and certs accordingly
2022-08-11T20:18:41.717170Z	info	x509 cert - Issuer: "CN=istiod.istio-system.svc", Subject: "", SN: 1c43c1686425ee2e63f2db90bd3cf17f, NotBefore: "2022-08-11T20:16:41Z", NotAfter: "2032-08-08T20:18:41Z"
2022-08-11T20:18:41.717220Z	info	x509 cert - Issuer: "CN=certmanager-ca,O=cert-manager", Subject: "CN=istiod.istio-system.svc", SN: c172b51eeb4a2891fe66f30babb42bb0, NotBefore: "2022-08-11T20:17:25Z", NotAfter: "2022-08-13T20:17:25Z"
2022-08-11T20:18:41.717254Z	info	x509 cert - Issuer: "CN=certmanager-ca,O=cert-manager", Subject: "CN=certmanager-ca,O=cert-manager", SN: ea1760f2dcf9806a8c997c4bc4b2fb30, NotBefore: "2022-08-11T20:13:33Z", NotAfter: "2025-01-27T20:13:33Z"
2022-08-11T20:18:41.717256Z	info	Istiod certificates are reloaded
```

## Summary

We now can have peace of mind that our Istio will not only detect that the Intermediate CA has changed, but the re-issuing of a new CA is handled `cert-manager` who can configure other Issuers.
