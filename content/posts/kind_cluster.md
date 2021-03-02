---
title: "Kind_cluster"
date: 2021-02-19T17:17:26Z
author: "Carlos Castro"
tags: ["Docker", "kind", "e2e"]
categories: ["Docker"]
description: "Quick kind demo"
draft: false
showToc: false
TocOpen: false
hidemeta: false
comments: false
disableHLJS: true 
disableShare: true
disableHLJS: true
searchHidden: true
---

Create new cluster:
```sh
kind create cluster {name}
```

```sh
kind load docker-image {image} --name {cluster_name}
```

See if image is uploaded
```sh
$ kubectl get nodes
$ docker exec -it {node_name} bash
$ crictl images
```

If `ImagePullBackOff` still persists and its a custom image, check if `imagePullPolicy` is `IfNotPresent` instead of `Always`.

```sh
$ kind delete cluster --name {name} 
```
