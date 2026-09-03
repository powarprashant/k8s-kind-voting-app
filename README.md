# K8s Kind Voting App

A guide for setting up a Kubernetes cluster with Kind, installing and configuring
Argo CD, and deploying the example voting application via Argo CD.

## Overview

This guide covers the steps to:

- Install Docker and Kind.
- Create a Kubernetes cluster using Kind.
- Install and access kubectl.
- Set up the Kubernetes Dashboard.
- Install and configure Argo CD.
- Connect and manage the cluster with Argo CD.

## Architecture

![Architecture diagram](k8s-kind-voting-app.png)

The application consists of:

- A front-end web app in [Python](/vote) which lets you vote between two options
- A [Redis](https://hub.docker.com/_/redis/) instance which collects new votes
- A [.NET](/worker/) worker which consumes votes and stores them in…
- A [Postgres](https://hub.docker.com/_/postgres/) database backed by a volume
- A [Node.js](/result) web app which shows the voting results in real time

## Deploy

```sh
# create the cluster
kind create cluster --config kind-cluster/config.yml

# deploy the app
kubectl apply -f k8s-specifications/
```

## CI

[.github/workflows/ci.yaml](.github/workflows/ci.yaml) scans the source with
Trivy, then builds, scans, and pushes the `vote`, `result`, and `worker` images
to Docker Hub on pushes to `main`.
