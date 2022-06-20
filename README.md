# Kong Rate Limiting Advanced bug example

## Install dependencies

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami

# install redis
helm install redis bitnami/redis --namespace default --values redis/values.yaml 

# install postgres
helm install postgres bitnami/postgresql --namespace default --values postgres/values.yaml
```

## Add kong namespace and secrets

```shell
# create kong namespace
kubectl create namespace kong

# create secrets with license and tls cert for hybrid mode
# https://github.com/Kong/charts/blob/main/charts/kong/README.md#certificates

# update secrets.yaml with your own enterprise license 
kubectl apply -f control-plane-secrets/secrets.yaml
```

## Deploy control and data planes

```shell
helm repo add bitnami https://charts.konghq.com

# install control plane
helm install control-plane kong/kong --namespace kong --values control-plane/values.yaml

# install data plane
helm install data-plane kong/kong --namespace kong --values data-plane/values.yaml
```

## Deploy echo service

```shell
kubectl apply -f echo-service/
```

## Expose admin API and add consumer to consumer group

```shell
# expose admin api locally
kubectl port-forward svc/control-plane-kong-admin -n kong 8444:8444

# create consumer group called premium
curl -k -X POST https://localhost:8444/consumer_groups --data name="premium"

# and override its settings for rate limiting plugin
curl -sk -X PUT https://localhost:8444/consumer_groups/premium/overrides/plugins/rate-limiting-advanced \
  --data config.limit=20 \
  --data config.window_size=60 \
  --data config.retry_after_jitter_max=0

# create test consumer
CONSUMER_ID=$(curl -sk -X POST https://localhost:8444/consumers --data "username=test" | jq -r '.id')

# and generate api key 
API_KEY=$(curl -sk -X POST "https://localhost:8444/consumers/${CONSUMER_ID}/key-auth" | jq -r '.key')

# add test consumer to premium consumer group
curl -k -X POST "https://localhost:8444/consumers/${CONSUMER_ID}/consumer_groups" \
  --data group="premium"

# get the port number for kong data plane node port for HTTP endpoint
KONG_PROXY_PORT=$(kubectl get svc -n kong data-plane-kong-proxy -o json | jq -r '.spec.ports[] | select(.port==80).nodePort')

# test rate limiting plugin and observe
for _ in $(seq 1 100); do
  curl -i "http://localhost:${KONG_PROXY_PORT}" -H "Host: test.example.com" -H "authorization: ${API_KEY}"
done
```

Observe following headers since their value will be growing with each retry

```text
RateLimit-Reset: 273
Retry-After: 273
```
