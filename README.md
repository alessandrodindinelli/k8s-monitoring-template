# Kubernetes Monitoring Template

> **Disclaimer**: This repository was written and tested to be used in the future, as a possible reference, if needed. I can't ensure that the code will continue to work as intended with future updates to the tools used. Enjoy 🤓

## Introduction

This repository provides a basic setup for a local development Kubernetes cluster using Kind.
The idea is to have a reusable template to quickly startup a local cluster to utilize in demos or various development.

## Requirements

The following are the versions of the various tools used:

- Helm - 3.19.0
  - kube-prometheus-stack - 78.2.3
  - ingress-nginx-controller - 1.13.3
- Kind - 0.30.0
- Kubectl - 1.32.2

## Structure

The project contains a `kind` folder with the cluster configuration, to have a simple setup of two worker nodes, with port-mapping that enables us to expose services on localhost.
The values file contained in the `loki` folder contains the configuration for both Grafana and Prometheus, to enable a basic monitoring setup.

## Usage

### Kind cluster creation

To test this setup we'll use a [Kind](https://kind.sigs.k8s.io/) cluster.
With the following command we'll deploy this local configuration: [kind/cluster.yaml](./kind/cluster.yaml).
The idea is to have a single controller node, with mappings for ports 80 and 443 to make it easier to run local tests.

```bash
kind create cluster --name kind-cluster --config ./kind/cluster.yaml
```

### NGINX controller setup

To make our setup reachable by a browser we use nginx in our cluster.
We require minimal configuation, so the default setup for Kind can be used, we'll patch it with a second command to make it run on the controller node so that the ports 80 and 443 are reachable.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

kubectl -n ingress-nginx patch deploy ingress-nginx-controller --type='merge' -p '{"spec":{"template":{"spec":{"nodeSelector":{"ingress-ready":"true"}}}}}'

# NB: you can double check that the nginx pod is now on the right node
kubectl -n ingress-nginx get pods -o wide
```

### Prometheus and Grafana installation

Here we deploy both Prometheus and Grafana, with a basic setup.
In particular we want to expose the endpoints locally to facilitate testing, so by using the `*.localhost` domains most operating systems automatically redirects the requests on `127.0.0.1`.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace -f ./prometheus/values.yaml
```

You should be able to open these links on your browser after the setup is done:

- [https://grafana.demo.localhost](https://grafana.demo.localhost)
- [https://prometheus.demo.localhost](https://prometheus.demo.localhost)

Of course this is just a local demo so we don't bother too much with security, so you can find the credentials to enter for Grafana on the values file.
In a real world scenario I'd recommend encrypting the file itself or using a Secrets Manager of some sort.

## Cleanup

After you're done with your testing, here's how you can delete the environment.
Here is a small checklist you can go through:

```bash
# See what you have
kind get clusters

# Delete a specific cluster (the name we used here is "kind-cluster")
kind delete cluster --name kind-cluster

# Remove the Docker network created by kind
docker network ls | grep kind
docker network rm kind
```
