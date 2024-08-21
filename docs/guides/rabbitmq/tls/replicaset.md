---
title: RabbitMQ ReplicaSet TLS/SSL Encryption
menu:
  docs_{{ .version }}:
    identifier: mg-tls-replicaset
    name: Replicaset
    parent: mg-tls
    weight: 30
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/README.md).

# Run RabbitMQ with TLS/SSL (Transport Encryption)

KubeDB supports providing TLS/SSL encryption (via, `sslMode` and `clusterAuthMode`) for RabbitMQ. This tutorial will show you how to use KubeDB to run a RabbitMQ database with TLS/SSL encryption.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Install [`cert-manger`](https://cert-manager.io/docs/installation/) v1.0.0 or later to your cluster to manage your SSL/TLS certificates.

- Now, install KubeDB cli on your workstation and KubeDB operator in your cluster following the steps [here](/docs/setup/README.md).

- To keep things isolated, this tutorial uses a separate namespace called `demo` throughout this tutorial.

  ```bash
  $ kubectl create ns demo
  namespace/demo created
  ```

> Note: YAML files used in this tutorial are stored in [docs/examples/RabbitMQ](https://github.com/kubedb/docs/tree/{{< param "info.version" >}}/docs/examples/RabbitMQ) folder in GitHub repository [kubedb/docs](https://github.com/kubedb/docs).

## Overview

KubeDB uses following crd fields to enable SSL/TLS encryption in RabbitMQ.

- `spec:`
  - `sslMode`
  - `tls:`
    - `issuerRef`
    - `certificate`
  - `clusterAuthMode`

Read about the fields in details in [RabbitMQ concept](/docs/guides/RabbitMQ/concepts/RabbitMQ.md),

`sslMode`, and `tls` is applicable for all types of RabbitMQ (i.e., `standalone`, `replicaset` and `sharding`), while `clusterAuthMode` provides [ClusterAuthMode](https://docs.RabbitMQ.com/manual/reference/program/mongod/#cmdoption-mongod-clusterauthmode) for RabbitMQ clusters (i.e., `replicaset` and `sharding`).

When, SSLMode is anything other than `disabled`, users must specify the `tls.issuerRef` field. KubeDB uses the `issuer` or `clusterIssuer` referenced in the `tls.issuerRef` field, and the certificate specs provided in `tls.certificate` to generate certificate secrets. These certificate secrets are then used to generate required certificates including `ca.crt`, `mongo.pem` and `client.pem`.

The subject of `client.pem` certificate is added as `root` user in `$external` RabbitMQ database. So, user can use this client certificate for `RabbitMQ-X509` `authenticationMechanism`.

## Create Issuer/ ClusterIssuer

We are going to create an example `Issuer` that will be used throughout the duration of this tutorial to enable SSL/TLS in RabbitMQ. Alternatively, you can follow this [cert-manager tutorial](https://cert-manager.io/docs/configuration/ca/) to create your own `Issuer`.

- Start off by generating you ca certificates using openssl.

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./ca.key -out ./ca.crt -subj "/CN=mongo/O=kubedb"
```

- Now create a ca-secret using the certificate files you have just generated.

```bash
kubectl create secret tls mongo-ca \
     --cert=ca.crt \
     --key=ca.key \
     --namespace=demo
```

Now, create an `Issuer` using the `ca-secret` you have just created. The `YAML` file looks like this:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: mongo-ca-issuer
  namespace: demo
spec:
  ca:
    secretName: mongo-ca
```

Apply the `YAML` file:

```bash
$ kubectl create -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/RabbitMQ/tls/issuer.yaml
issuer.cert-manager.io/mongo-ca-issuer created
```

## TLS/SSL encryption in RabbitMQ Replicaset

Below is the YAML for RabbitMQ Replicaset. Here, [`spec.sslMode`](/docs/guides/RabbitMQ/concepts/RabbitMQ.md#specsslMode) specifies `sslMode` for `replicaset` (which is `requireSSL`) and [`spec.clusterAuthMode`](/docs/guides/RabbitMQ/concepts/RabbitMQ.md#specclusterAuthMode) provides `clusterAuthMode` for RabbitMQ replicaset nodes (which is `x509`).

```yaml
apiVersion: kubedb.com/v1alpha2
kind: RabbitMQ
metadata:
  name: mgo-rs-tls
  namespace: demo
spec:
  version: "4.4.26"
  sslMode: requireSSL
  tls:
    issuerRef:
      apiGroup: "cert-manager.io"
      kind: Issuer
      name: mongo-ca-issuer
  clusterAuthMode: x509
  replicas: 4
  replicaSet:
    name: rs0
  storage:
    storageClassName: "standard"
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
```

### Deploy RabbitMQ Replicaset

```bash
$ kubectl create -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/RabbitMQ/tls/mg-replicaset-ssl.yaml
RabbitMQ.kubedb.com/mgo-rs-tls created
```

Now, wait until `mgo-rs-tls created` has status `Ready`. i.e,

```bash
$ watch kubectl get mg -n demo
Every 2.0s: kubectl get RabbitMQ -n demo
NAME         VERSION     STATUS    AGE
mgo-rs-tls   4.4.26   Ready     4m10s
```

### Verify TLS/SSL in RabbitMQ Replicaset

Now, connect to this database through [mongo-shell](https://docs.RabbitMQ.com/v4.0/mongo/) and verify if `SSLMode` and `ClusterAuthMode` has been set up as intended.

```bash
$ kubectl describe secret -n demo mgo-rs-tls-client-cert
Name:         mgo-rs-tls-client-cert
Namespace:    demo
Labels:       <none>
Annotations:  cert-manager.io/alt-names:
              cert-manager.io/certificate-name: mgo-rs-tls-client-cert
              cert-manager.io/common-name: root
              cert-manager.io/ip-sans:
              cert-manager.io/issuer-group: cert-manager.io
              cert-manager.io/issuer-kind: Issuer
              cert-manager.io/issuer-name: mongo-ca-issuer
              cert-manager.io/uri-sans:

Type:  kubernetes.io/tls

Data
====
ca.crt:   1147 bytes
tls.crt:  1172 bytes
tls.key:  1679 bytes
```

Now, Let's exec into a RabbitMQ container and find out the username to connect in a mongo shell,

```bash
$ kubectl exec -it mgo-rs-tls-0 -n demo bash
root@mgo-rs-tls-0:/$ ls /var/run/RabbitMQ/tls
ca.crt  client.pem  mongo.pem
root@mgo-rs-tls-0:/$ openssl x509 -in /var/run/RabbitMQ/tls/client.pem -inform PEM -subject -nameopt RFC2253 -noout
subject=CN=root,O=kubedb
```

Now, we can connect using `CN=root,O=kubedb` as root to connect to the mongo shell,

```bash
root@mgo-rs-tls-0:/$ mongo --tls --tlsCAFile /var/run/RabbitMQ/tls/ca.crt --tlsCertificateKeyFile /var/run/RabbitMQ/tls/client.pem admin --host localhost --authenticationMechanism RabbitMQ-X509 --authenticationDatabase='$external' -u "CN=root,O=kubedb" --quiet
Welcome to the RabbitMQ shell.
For interactive help, type "help".
For more comprehensive documentation, see
	http://docs.RabbitMQ.org/
Questions? Try the support group
	http://groups.google.com/group/RabbitMQ-user
rs0:PRIMARY>
```

We are connected to the mongo shell. Let's run some command to verify the sslMode and the user,

```bash
rs0:PRIMARY> db.adminCommand({ getParameter:1, sslMode:1 })
{
	"sslMode" : "requireSSL",
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1599490676, 1),
		"signature" : {
			"hash" : BinData(0,"/wQ4pf4HVi1T7SOyaB3pXO56j64="),
			"keyId" : NumberLong("6869759546676477954")
		}
	},
	"operationTime" : Timestamp(1599490676, 1)
}

rs0:PRIMARY> use $external
switched to db $external

rs0:PRIMARY> show users
{
	"_id" : "$external.CN=root,O=kubedb",
	"userId" : UUID("9cebbcf4-74bf-47dd-a485-1604125058da"),
	"user" : "CN=root,O=kubedb",
	"db" : "$external",
	"roles" : [
		{
			"role" : "root",
			"db" : "admin"
		}
	],
	"mechanisms" : [
		"external"
	]
}
> exit
bye
```

You can see here that, `sslMode` is set to `requireSSL` and a user is created in `$external` with name `"CN=root,O=kubedb"`.

## Changing the SSLMode & ClusterAuthMode

User can update `sslMode` & `ClusterAuthMode` if needed. Some changes may be invalid from RabbitMQ end, like using `sslMode: disabled` with `clusterAuthMode: x509`.

The good thing is, **KubeDB operator will throw error for invalid SSL specs while creating/updating the RabbitMQ object.** i.e.,

```bash
$ kubectl patch -n demo mg/mgo-rs-tls -p '{"spec":{"sslMode": "disabled","clusterAuthMode": "x509"}}' --type="merge"
Error from server (Forbidden): admission webhook "RabbitMQ.validators.kubedb.com" denied the request: can't have disabled set to RabbitMQ.spec.sslMode when RabbitMQ.spec.clusterAuthMode is set to x509
```

To **update from Keyfile Authentication to x.509 Authentication**, change the `sslMode` and `clusterAuthMode` in recommended sequence as suggested in [official documentation](https://docs.RabbitMQ.com/manual/tutorial/update-keyfile-to-x509/). Each time after changing the specs, follow the procedure that is described above to verify the changes of `sslMode` and `clusterAuthMode` inside the database.

## Cleaning up

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete RabbitMQ -n demo mgo-rs-tls
kubectl delete issuer -n demo mongo-ca-issuer
kubectl delete ns demo
```

## Next Steps

- Detail concepts of [RabbitMQ object](/docs/guides/RabbitMQ/concepts/RabbitMQ.md).
- [Backup and Restore](/docs/guides/RabbitMQ/backup/overview/index.md) RabbitMQ databases using Stash.
- Initialize [RabbitMQ with Script](/docs/guides/RabbitMQ/initialization/using-script.md).
- Monitor your RabbitMQ database with KubeDB using [out-of-the-box Prometheus operator](/docs/guides/RabbitMQ/monitoring/using-prometheus-operator.md).
- Monitor your RabbitMQ database with KubeDB using [out-of-the-box builtin-Prometheus](/docs/guides/RabbitMQ/monitoring/using-builtin-prometheus.md).
- Use [private Docker registry](/docs/guides/RabbitMQ/private-registry/using-private-registry.md) to deploy RabbitMQ with KubeDB.
- Use [kubedb cli](/docs/guides/RabbitMQ/cli/cli.md) to manage databases like kubectl for Kubernetes.
- Detail concepts of [RabbitMQ object](/docs/guides/RabbitMQ/concepts/RabbitMQ.md).
- Want to hack on KubeDB? Check our [contribution guidelines](/docs/CONTRIBUTING.md).
