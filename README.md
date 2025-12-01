

## Prerequisites

### harbor-cli

```bash
curl -L https://github.com/goharbor/harbor-cli/releases/download/v0.0.15/harbor-cli_0.0.15_linux_amd64.tar.gz | tar xvzf - harbor-cli
chmod u+x harbor-cli
```

### skopeo

https://github.com/containers/skopeo/blob/main/install.md

```bash
sudo apt install skopeo
```

## Reproduction instructions
### Install harbor
```bash
minikube start

kubectl create namespace harbor
kubectl apply -f harbor-tls.yml

helm repo add harbor https://helm.goharbor.io
helm upgrade --install harbor harbor/harbor \
    --namespace harbor \
    --version=1.18.0 \
    --set expose.type=nodePort \
    --set expose.tls.certSource=secret \
    --set expose.tls.secret.secretName=harbor-tls \
    --wait

# This can take a few minutes
```

### Expose harbor

```bash
minikube service harbor --url --https=true --namespace=harbor

# Once running set a bash variable to the exposed 2nd port
# e.g.
#
# minikube service harbor --url --https=true --namespace=harbor
# https://127.0.0.1:39095
# https://127.0.0.1:43831

# HARBOR_TUNNEL_PORT=43831
```

### Configure Harbor

```bash
./harbor-cli login https://127.0.0.1:${HARBOR_TUNNEL_PORT} -u admin -p Harbor12345

./harbor-cli registry create --name dockerhub --type docker-hub --url https://hub.docker.com --insecure=true -v

./harbor-cli project create dockerhub --public --proxy-cache --storage-limit=-1 --registry-id 1 -v
```

### Check if blobs exist

```bash
curl -v https://127.0.0.1:${HARBOR_TUNNEL_PORT}/v2/

IMAGE=dockerhub/library/redis
TAG=latest

HARBOR_TOKEN=$(curl --insecure --silent \
    -G \
    --data-urlencode "scope=repository:$IMAGE:pull" \
    --data-urlencode "service=harbor-registry" \
    https://127.0.0.1:${HARBOR_TUNNEL_PORT}/service/token | \
    jq -r '.token'
)

MANIFEST_SHA=$(curl --insecure --silent \
    -H "Authorization: Bearer ${HARBOR_TOKEN}" \
    https://127.0.0.1:${HARBOR_TUNNEL_PORT}/v2/${IMAGE}/manifests/${TAG} | \
    jq -r '.manifests[] | select(.platform.architecture == "amd64") | select(.platform.os == "linux") | .digest' \
)

BLOB_LIST=$(curl --insecure --silent \
    -H "Authorization: Bearer ${HARBOR_TOKEN}" \
    -H "Accept: application/vnd.oci.image.manifest.v1+json" \
    https://127.0.0.1:${HARBOR_TUNNEL_PORT}/v2/${IMAGE}/manifests/${MANIFEST_SHA} | jq -r '.layers[].digest' \
)

for BLOB_SHA in $BLOB_LIST; do
    echo "HEAD https://127.0.0.1:${HARBOR_TUNNEL_PORT}/v2/${IMAGE}/blobs/${BLOB_SHA}"
    curl --insecure --silent \
        --head \
        -H "Authorization: Bearer ${HARBOR_TOKEN}" \
        -H "Connection: close" \
        https://127.0.0.1:${HARBOR_TUNNEL_PORT}/v2/${IMAGE}/blobs/${BLOB_SHA} | \
        grep -E "HTTP|Content-Length"
done
```

### Pull image

```bash
skopeo copy --src-tls-verify=false docker://127.0.0.1:${HARBOR_TUNNEL_PORT}/${IMAGE}:${TAG} docker-archive:test.tar:test:latest && rm test.tar
```

### Check if blobs exist again

```bash
curl -v https://127.0.0.1:${HARBOR_TUNNEL_PORT}/v2/

IMAGE=dockerhub/library/redis
TAG=latest

HARBOR_TOKEN=$(curl --insecure --silent \
    -G \
    --data-urlencode "scope=repository:$IMAGE:pull" \
    --data-urlencode "service=harbor-registry" \
    https://127.0.0.1:${HARBOR_TUNNEL_PORT}/service/token | \
    jq -r '.token'
)

MANIFEST_SHA=$(curl --insecure --silent \
    -H "Authorization: Bearer ${HARBOR_TOKEN}" \
    https://127.0.0.1:${HARBOR_TUNNEL_PORT}/v2/${IMAGE}/manifests/${TAG} | \
    jq -r '.manifests[] | select(.platform.architecture == "amd64") | select(.platform.os == "linux") | .digest' \
)

BLOB_LIST=$(curl --insecure --silent \
    -H "Authorization: Bearer ${HARBOR_TOKEN}" \
    -H "Accept: application/vnd.oci.image.manifest.v1+json" \
    https://127.0.0.1:${HARBOR_TUNNEL_PORT}/v2/${IMAGE}/manifests/${MANIFEST_SHA} | jq -r '.layers[].digest' \
)

for BLOB_SHA in $BLOB_LIST; do
    echo "HEAD https://127.0.0.1:${HARBOR_TUNNEL_PORT}/v2/${IMAGE}/blobs/${BLOB_SHA}"
    curl --insecure --silent \
        --head \
        -H "Authorization: Bearer ${HARBOR_TOKEN}" \
        -H "Connection: close" \
        https://127.0.0.1:${HARBOR_TUNNEL_PORT}/v2/${IMAGE}/blobs/${BLOB_SHA} | \
        grep -E "HTTP|Content-Length"
done
```