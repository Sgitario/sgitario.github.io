---
layout: post
title: Setup Container Registry Action
date: 2023-01-16
tags: [ Containers, GitHub ]
---

I'm pleased to announce my first GitHub action (https://github.com/marketplace/actions/setup-container-registry)[https://github.com/marketplace/actions/setup-container-registry]!

This action setups the [Docker Registry](https://docs.docker.com/registry/) with a single GitHub Action!

```yaml
on: [push]
jobs:
  simple:
    name: Simple Workflow
    runs-on: ubuntu-latest
    steps:
      - id: registry
        uses: Sgitario/setup-container-registry@v1
      - name: Usage
        run: |
          # You can access the container registry via:
          # - Environment Variable: $CONTAINER_REGISTRY_URL
          # - Output: ${{ steps.registry.outputs.container-registry-url }}       
```

## Why do we need this action?

Normally, you first build your container, and then you verify the image by running end-to-end tests. This already works if you run the images locally. However, if what you want is to test the image in a cluster environment like [Kind](https://github.com/marketplace/actions/kind-kubernetes-in-docker-action) or [OpenShift](https://github.com/marketplace/actions/setup-okd-openshift-cluster), you need the containers up and running in a registry and here is when it comes to playing this action :)

## Conclusion

I wanted to take this opportunity to learn how to implement and publish my custom actions in GitHub and I'm pleased to say that the experience was really easy and straightforward. 