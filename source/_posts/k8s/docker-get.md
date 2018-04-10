---
title: docker-get
date: 2018-03-14 21:19:47
tags: 
- docker-get
- docker
- proxy
---

A docker image download proxy.

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
