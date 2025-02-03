# 2nd-layer-hub-apim
Test possibility to run Hub APIM behind another ingress controller


## Generate Wildcard Cert

```bash
LEGO_DISABLE_CNAME_SUPPORT=true lego -a --email mageekbox@gmail.com --dns gandiv5 -d "${CLUSTERNAME}.${DOMAINNAME}" -d "*.${CLUSTERNAME}.${DOMAINNAME}" run
```

## Deploy OSS

### Create OSS namespace

```bash
kubectl create ns oss-traefik
```

### Create Wildcard secret

```bash
kubectl create secret tls wildcard-cert --namespace oss-traefik --cert=.lego/certificates/${CLUSTERNAME}.${DOMAINNAME}.crt --key=.lego/certificates/${CLUSTERNAME}.${DOMAINNAME}.key
```

### deploy Traefik

```bash
helm upgrade --install oss-traefik traefik/traefik --create-namespace --namespace oss-traefik --values oss/oss-values.yaml
```

### Set DNS entry

```bash
ADDRECORD='{
  "rrset_type": "CNAME",
  "rrset_name": "*.'$CLUSTERNAME'",
  "rrset_ttl": "1800",
  "rrset_values": [
    "'$(kubectl get svc/oss-traefik -n oss-traefik --no-headers | awk {'print $4'})'."
  ]
}'
curl -s -X POST -d $ADDRECORD \
  -H "Authorization: Apikey $GANDIV5_API_KEY" \
  -H "Content-Type: application/json" \
  https://api.gandi.net/v5/livedns/domains/$DOMAINNAME/records
```

### Deploy dashboard ingress

```bash
envsubst < oss/ingress.yaml | kubectl apply -f -
```

## deploy Redis

### For distributed features

```bash
helm upgrade --install redis bitnami/redis --namespace redis --values redis/values.yaml --create-namespace
```

## Deploy Hub

### Create Hub namespace

```bash
kubectl create ns traefik
```

### Create Hub token secret

```bash
kubectl create secret generic hub-license --from-literal=token="${HUB_TOKEN}" -n traefik
```

### deploy Hub

```bash
helm upgrade --install traefik traefik/traefik --create-namespace --namespace traefik --values hub/hub-values.yaml
```

### Deploy Catchall ingress

```bash
envsubst < oss/catch-all.yaml | kubectl apply -f -
```

### Deploy dashboard ingress

```bash
envsubst < hub/ingress.yaml | kubectl apply -f -
```

## Deploy api

```bash
k apply -f api/namespace.yaml
k apply -f api/customers/deployments
envsubst < api/customers/ingresses/api-ingress-v1.yaml | kubectl apply -f -
k apply -f api/customers/apis
envsubst < api/portal/portal-ingress.yaml | kubectl apply -f -
envsubst < api/portal/portal.yaml | kubectl apply -f -
```
