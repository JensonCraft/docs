# Autoscale Sample

A demonstration of the autoscaling capabilities of an Knative Serving Revision.

## Prerequisites

1. [Install Knative Serving](https://github.com/knative/docs/blob/master/install/README.md)
1. Install [docker](https://www.docker.com/)

## Setup

Build the autoscale app container and publish it to your registry of choice:

```shell
REPO="gcr.io/<your-project-here>"

# Build and publish the container, run from the root directory.
docker build \
  --tag "${REPO}/serving/samples/autoscale-go" \
  --file=serving/samples/autoscale-go/Dockerfile .
docker push "${REPO}/serving/samples/autoscale-go"

# Replace the image reference with our published image.
perl -pi -e "s@github.com/knative/docs/serving/samples/autoscale-go@${REPO}/serving/samples/autoscale-go@g" serving/samples/autoscale-go/sample.yaml

# Deploy the Knative Serving sample
kubectl apply -f serving/samples/autoscale-go/sample.yaml

```

Export your ingress IP as SERVICE_IP.

```shell
# Put the ingress Host name into an environment variable.
export SERVICE_HOST=`kubectl get route autoscale-route -o jsonpath="{.status.domain}"`

# Put the ingress IP into an environment variable.
export SERVICE_IP=`kubectl get svc knative-ingressgateway -n istio-system -o jsonpath="{.status.loadBalancer.ingress[*].ip}"`
```

Request the largest prime less than 40,000,000 from the autoscale app.  Note that it consumes about 1 cpu/sec.

```shell
time curl --header "Host:$SERVICE_HOST" http://${SERVICE_IP?}/primes/40000000
```

## Running

Ramp up a bunch of traffic on the autoscale app (about 300 QPS).

```shell
kubectl delete namespace hey --ignore-not-found && kubectl create namespace hey
for i in `seq 2 2 60`; do
  kubectl -n hey run hey-$i --image josephburnett/hey --restart Never -- \
    -n 999999 -c $i -z 2m -host $SERVICE_HOST \
    "http://${SERVICE_IP?}/primes/40000000"
  sleep 1
done
watch kubectl get pods -n hey --show-all
```

Watch the Knative Serving deployment pod count increase.

```shell
watch kubectl get deploy
```

Look at the latency, requests/sec and success rate of each pod.

```shell
for i in `seq 4 4 120`; do kubectl -n hey logs hey-$i ; done | less
```

## Cleanup

```shell
kubectl delete namespace hey
kubectl delete -f serving/samples/autoscale-go/sample.yaml
```