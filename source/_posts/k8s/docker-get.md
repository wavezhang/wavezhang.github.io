---
title: docker-get
---

A download docker image pull proxy.

## Setup
```bash
curl -LO http://ss.samblade.ml/docker-get chmod +x docker-get mv docker-get /usr/bin/
```

## Usage

```bash
dockerget IMAGE[:TAG]
```


## Example

```bash
dockerget busybox
dockerget gcr.io/google_containers/kubernetes-dashboard-amd64:v1.8.3
```
