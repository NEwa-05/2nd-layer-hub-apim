# 2nd-layer-hub-apim
Test possibility to run Hub APIM behind another ingress controller


## Generate Wildcard Cert

```bash
LEGO_DISABLE_CNAME_SUPPORT=true lego -a --email mageekbox@gmail.com --dns gandiv5 -d "${CLUSTERNAME}.${DOMAINNAME}" -d "*.${CLUSTERNAME}.${DOMAINNAME}" run
```

## Deploy OSS

### Create namespace

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

### Deploy Catchall ingress

```bash
envsubst < oss/catch-all.yaml | kubectl apply -f -
```

## Deploy Hub

### Create namespace

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

### Deploy dashboard ingress

```bash
envsubst < hub/ingress.yaml | kubectl apply -f -
```

## Deploy app

### Create apps namespace

```bash
kubectl create ns whoami
```

### Deploy whoami, middleware and ingress

```bash
kubectl apply -f whoami/whoami.yaml
envsubst < whoami/ingress.yaml | kubectl apply -f -
```

## Deploy Keycloak

### Create keycloak namespace

```bash
kubectl create ns keycloak
```

### Create keycloak admin password

```bash
kubectl create secret generic keycloak-admin --from-literal=password="${KEYCLOAK_PASSWORD}" -n keycloak
```

### Create configmap for Traefik realm

```bash
envsubst < keycloak/realm-cm.yaml | kubectl apply -f -
```

### Set ingressRoute for keycloak

```bash
envsubst < keycloak/ingress.yaml | kubectl apply -f -
```

### Install Keycloak with realm configuration

```bash
helm upgrade --install keycloak bitnami/keycloak --create-namespace --namespace keycloak --set keycloakConfigCli.extraEnvVars\[0\].name="KEYCLOAK_URL" --set keycloakConfigCli.extraEnvVars\[0\].value="https://keycloak.${CLUSTERNAME}.${DOMAINNAME}" --values keycloak/keycloak-values.yaml --version 21.7.1
```


```bash
k apply -f api/namespace.yaml
k apply -f api/customers/deployments
envsubst < api/customers/ingresses/api-ingress-v1.yaml | kubectl apply -f -
envsubst < api/customers/ingresses/api-ingress-v2.yaml | kubectl apply -f -
envsubst < api/customers/ingresses/api-ingress-v3.yaml | kubectl apply -f -
k apply -f api/customers/apis
envsubst < api/portal/portal-ingress.yaml | kubectl apply -f -
envsubst < api/portal/portal.yaml | kubectl apply -f -
```