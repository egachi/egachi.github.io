---
layout: post
title: How to recover a docker image from Kubernetes Node
---

When you've lost a Docker image from your remote repository but it's still cached on a Kubernetes Node, you can recover it using containerd's command-line tool, ctr. This tool is typically bundled with containerd installations.

## Using ctr

ctr, often included in containerd installations, can be used in conjunction with crictl, a standalone kubernetes-sigs project. Here's how to interact with ctr and containerd:


1. Set up the environment: To interact with ctr define this environment variable:

    `export CONTAINER_RUNTIME_ENDPOINT="unix:///run/containerd/containerd.sock"`

2. For basic authentication (e.g., with AWS ECR):
   
    ```bash
    ECR_REGION=us-east-1
    ECR_PASSWORD=$(aws ecr get-login-password --region $ECR_REGION)
    ```

3. Tagging and pushing images: 

    ```bash
    ctr -n k8s.io image tag <IMAGE> <ECR:IMAGE:TAG>
    ctr -n k8s.io image push --user "AWS:$ECR_PASSWORD" <ECR:IMAGE:TAG>
    ```

    Note: The 'k8s.io' namespace is required for image interactions.

## Other Helpful Tools

- [nerdctrl](https://github.com/containerd/nerdctl): Docker-compatible CLI for containerd
- [crictl](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md): crictl provides a CLI for CRI-compatible container runtimes.