## Shared storage with S3 backend
The storage is definitely the most complex and important part of an application setup, once this part is 
completed, 80% of the tasks are completed.

Mounting an S3 bucket into a pod using FUSE allows you to access the data as if it were on the local disk. The 
mount is a pointer to an S3 location, so the data is never synced locally. Once mounted, any pod can read or even write
from that directory without the need for explicit keys.

**But as mentioned above - the data is not mirrored locally. As a result, read/write operations always go over the 
network and are correspondingly slow.**

However, it can be used to import and parse large amounts of data into a database.

## Overview

![s3-mount](/images/s3-mount.png)

## Before you Begin
You need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with 
your cluster. If you do not already have a cluster, you can create one by using the [Gardener](https://gardener.kubernetes.sap.corp/login).

Ensure that you have create the "imagePullSecret" in your cluster.
```sh 
kubectl create secret docker-registry artifactory --docker-server=<YOUR-REGISTRY>.docker.repositories.sap.ondemand.com --docker-username=<USERNAME> --docker-password=<PASSWORD> --docker-email=<EMAIL> -n <NAMESPACE>
```

## Setup
The first step is to clone this repository. Next is the Secret for the AWS API credentials of the user that has 
full access to our S3 bucket. Copy the `configmap_secrets_template.yaml` to `configmap_secrets.yaml` and place 
your secretes at the right place

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: s3-config
data:
  S3_BUCKET: <YOUR-S3-BUCKET-NAME>
  AWS_KEY: <YOUR-AWS-TECH-USER-ACCESS-KEY>
  AWS_SECRET_KEY: <YOUR-AWS-TECH-USER-SECRET>
```

## Build and deploy
Change the settings in the `build.sh` file with your docker registry settings. 

```sh
#!/usr/bin/env bash

########################################################################################################################
# PREREQUISTITS
########################################################################################################################
#
# - ensure that you have a valid Artifactory or other Docker registry account
# - Create your image pull secret in your namespace
#   kubectl create secret docker-registry artifactory --docker-server=<YOUR-REGISTRY>.docker.repositories.sap.ondemand.com --docker-username=<USERNAME> --docker-password=<PASSWORD> --docker-email=<EMAIL> -n <NAMESPACE>
# - change the settings below arcording your settings
#
# usage:
# Call this script with the version to build and push to the registry. After build/push the
# yaml/* files are deployed into your cluster
#
#  ./build.sh 1.0
#
VERSION=$1
PROJECT=kube-s3
REPOSITORY=cp-enablement.docker.repositories.sap.ondemand.com


# causes the shell to exit if any subcommand or pipeline returns a non-zero status.
set -e
# set debug mode
#set -x

.
.
.
.

```
Create the S3Fuse Pod and check the status:

```sh
# build and push the image to your docker registry
./build.sh 1.0 

# check that the pods are up and running
kubectl get pods

```

## Check success
Create a demo Pod and check the status:
```sh 
kubectl apply -f ./yaml/example_pod.yaml

# wait some second to get the od up and running...
kubectl get pods

# go into the pd and check that the /var/s3 is mounted with your S3 bucket content inside
kubectl exec -ti test-pd  sh

# inside the pod
ls -la /var/s3

```

## Why does this work?
Docker engine 1.10 added a new feature which allows containers to share the host mount namespace. This feature makes 
it possible to mount a s3fs container file system to a host file system through a shared mount, providing a persistent
network storage with S3 backend.

The key part is mountPath: `/var/s3:shared` which enables the volume to be mounted as shared inside the pod. When the 
container starts it will mount the S3 bucket onto `/vars3` and consequently the data will be available under 
`/mnt/data-s3fs` on the host and thus to any other container/pod running on it (and has `/mnt/data-s3fs` mounted too). 