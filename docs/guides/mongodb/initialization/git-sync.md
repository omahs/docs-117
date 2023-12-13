---
title: Initialize MongoDB from a Git repository
menu:
  docs_{{ .version }}:
    identifier: mg-git-sync-initialization
    name: Git Syncer
    parent: mg-initialization-mongodb
    weight: 12
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/README.md).

# Initialize MongoDB from Git repository

This tutorial will show you how to use KubeDB to initialize a MongoDB database with .js and/or .sh script coming from a git repo.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Now, install KubeDB cli on your workstation and KubeDB operator in your cluster following the steps [here](/docs/setup/README.md).

- To keep things isolated, this tutorial uses a separate namespace called `demo` throughout this tutorial.

  ```bash
  $ kubectl create ns demo
  namespace/demo created
  ```

  In this tutorial we will use .js script stored in GitHub repository [kubedb/mongodb-init-scripts](https://github.com/kubedb/mongodb-init-scripts).

> Note: The yaml files used in this tutorial are stored in [docs/examples/mongodb](https://github.com/kubedb/docs/tree/{{< param "info.version" >}}/docs/examples/mongodb) folder in GitHub repository [kubedb/docs](https://github.com/kubedb/docs).

## Prepare Initialization Scripts

MongoDB supports initialization with `.sh` and `.js` files. In this tutorial, we will use `init.js` script from [mongodb-init-scripts](https://github.com/kubedb/mongodb-init-scripts) git repository to insert data inside `kubedb` DB.

We will use a ConfigMap as script source. You can use any Kubernetes supported [volume](https://kubernetes.io/docs/concepts/storage/volumes) as script source.

At first, we will create a ConfigMap from `init.js` file. Then, we will provide this ConfigMap as script source in `init.script` of MongoDB crd spec.

Let's create a ConfigMap with initialization script,

```bash
$ kubectl create configmap -n demo mg-init-script \
--from-literal=init.js="$(curl -fsSL https://github.com/kubedb/mongodb-init-scripts/raw/master/init.js)"
configmap/mg-init-script created
```

## Create a MongoDB database with Init-Script

Below is the `MongoDB` object created in this tutorial.

```yaml
apiVersion: kubedb.com/v1alpha2
kind: MongoDB
metadata:
  name: mgo-init-script
  namespace: demo
spec:
  version: "4.4.26"
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  init:
    script:
      configMap:
        name: mg-init-script
```

```bash
$ kubectl create -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/mongodb/Initialization/replicaset.yaml
mongodb.kubedb.com/mgo-init-script created
```