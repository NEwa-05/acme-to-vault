# acme-to-vault

This repo describe the acme.json file migration to vault storage with Traefik Hub

## Deploy Traefik OSS

### deploy Traefik OSS n1

```bash
helm upgrade --install traefik-1 traefik/traefik --create-namespace --namespace traefik-1 --values oss/oss-values-1.yaml
```

### Set DNS entry

```bash
ADDRECORD='{
  "rrset_type": "CNAME",
  "rrset_name": "*.1.'$CLUSTERNAME'",
  "rrset_ttl": "1800",
  "rrset_values": [
    "'$(kubectl get svc/traefik-1 -n traefik-1 --no-headers | awk {'print $4'})'."
  ]
}'
curl -s -X POST -d $ADDRECORD \
  -H "Authorization: Apikey $GANDIV5_API_KEY" \
  -H "Content-Type: application/json" \
  https://api.gandi.net/v5/livedns/domains/$DOMAINNAME/records
```

### Deploy OSS dashboard ingress

```bash
envsubst < oss/traefik-1-dashboard.yaml | kubectl apply -f -
```

## Deploy Traefik OSS n2

### deploy Traefik

```bash
helm upgrade --install traefik-2 traefik/traefik --create-namespace --namespace traefik-2 --values oss/oss-values-2.yaml
```

### Set DNS entry

```bash
ADDRECORD='{
  "rrset_type": "CNAME",
  "rrset_name": "*.2.'$CLUSTERNAME'",
  "rrset_ttl": "1800",
  "rrset_values": [
    "'$(kubectl get svc/traefik-2 -n traefik-2 --no-headers | awk {'print $4'})'."
  ]
}'
curl -s -X POST -d $ADDRECORD \
  -H "Authorization: Apikey $GANDIV5_API_KEY" \
  -H "Content-Type: application/json" \
  https://api.gandi.net/v5/livedns/domains/$DOMAINNAME/records
```

### Deploy OSS dashboard ingress

```bash
envsubst < oss/traefik-2-dashboard.yaml | kubectl apply -f -
```

## Deploy app

### Create apps namespace

```bash
kubectl create ns whoami-1
kubectl create ns whoami-2
```

### Deploy whoami and ingress

```bash
kubectl apply -f whoami/whoami-1.yaml
envsubst < whoami/ingress-1.yaml | kubectl apply -f -
kubectl apply -f whoami/whoami-2.yaml
envsubst < whoami/ingress-2.yaml | kubectl apply -f -
```

## Vault deployment

```bash
helm upgrade --install vault hashicorp/vault -f vault/values.yaml --namespace vault --create-namespace
```

```bash
kubectl -n vault exec -it vault-0 -- vault status -tls-skip-verify
```

```bash
envsubst < vault/ingress.yaml | k apply -f -
```

##### unseal from the CLI

```bash
export VAULT_ADDR=https://vault.1.${CLUSTERNAME}.${DOMAINNAME}
export VAULT_SKIP_VERIFY='true'
vault login
```

```bash
vault secrets enable -path=hubcerts kv-v2
```

Then create a Vault policy to store LE certs:

```bash
cat <<EOF | vault policy write traefik-write-certs -
path "hubcerts/le/config" {
  capabilities = ["read"]
}
path "hubcerts/le/data/*" {
  capabilities = ["create", "read", "update", "list"]
}
path "hubcerts/le/metadata/*" {
  capabilities = ["read", "delete", "list"]
}
EOF
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

### Set DNS entry

```bash
ADDRECORD='{
  "rrset_type": "CNAME",
  "rrset_name": "*.'$CLUSTERNAME'",
  "rrset_ttl": "1800",
  "rrset_values": [
    "'$(kubectl get svc/traefik -n traefik --no-headers | awk {'print $4'})'."
  ]
}'
curl -s -X POST -d $ADDRECORD \
  -H "Authorization: Apikey $GANDIV5_API_KEY" \
  -H "Content-Type: application/json" \
  https://api.gandi.net/v5/livedns/domains/$DOMAINNAME/records
```

### migrate acme.json to vault secret storage

#### Get acme.json from traefik-1

```bash
kubectl -n traefik-1 exec -it $(k get po -n traefik-1 --no-headers -o custom-columns=":metadata.name") -- cat /data/acme.json >> data/acme-1.json
```

#### Get acme.json from traefik-2

```bash
kubectl -n traefik-2 exec -it $(k get po -n traefik-2 --no-headers -o custom-columns=":metadata.name") -- cat /data/acme.json >> data/acme-2.json
```

#### create global acme.json

```bash
jq ".le.Certificates += $(jq .le.Certificates data/acme-2.json)" data/acme-1.json > data/acme.json
chmod 600 data/acme.json
```

### Push certs to Vaults

```bash
docker run --net=host --volume $(pwd)/data:/data europe-west9-docker.pkg.dev/traefiklabs/traefik-hub/traefik-hub:pr-390 migrate-acme --source.acme.storage=/data/acme.json --destination.distributedAcme.storage.vault.url=https://vault.1.${CLUSTERNAME}.${DOMAINNAME} --destination.distributedAcme.storage.vault.enginePath=hubcerts --destination.distributedAcme.storage.vault.auth.token=root --resolverName=le --force
```

### Deploy dashboard ingress

```bash
envsubst < hub/ingress.yaml | kubectl apply -f -
```

### Set DNS alias for migrated app

```bash
ADDRECORD='{
  "rrset_type": "CNAME",
  "rrset_name": "whoami.1.'$CLUSTERNAME'",
  "rrset_ttl": "1800",
  "rrset_values": [
    "'$(kubectl get svc/traefik -n traefik --no-headers | awk {'print $4'})'."
  ]
}'
curl -s -X POST -d $ADDRECORD \
  -H "Authorization: Apikey $GANDIV5_API_KEY" \
  -H "Content-Type: application/json" \
  https://api.gandi.net/v5/livedns/domains/$DOMAINNAME/records
```

```bash
ADDRECORD='{
  "rrset_type": "CNAME",
  "rrset_name": "whoami.2.'$CLUSTERNAME'",
  "rrset_ttl": "1800",
  "rrset_values": [
    "'$(kubectl get svc/traefik -n traefik --no-headers | awk {'print $4'})'."
  ]
}'
curl -s -X POST -d $ADDRECORD \
  -H "Authorization: Apikey $GANDIV5_API_KEY" \
  -H "Content-Type: application/json" \
  https://api.gandi.net/v5/livedns/domains/$DOMAINNAME/records
```

### change app gateway DNS

```bash
envsubst < whoami/ingresses-hub.yaml | kubectl apply -f -
```
