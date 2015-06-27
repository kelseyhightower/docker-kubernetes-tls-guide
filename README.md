# Docker and Kubernetes TLS Guide

This guide will walk you through setting up TLS and TLS cert client authentication for Docker and Kubernetes.

## Install CFSSL

https://github.com/cloudflare/cfssl

## Review and customize CSRs

```
$ git clone https://github.com/kelseyhightower/docker-kubernetes-tls-guide.git 
```

## Initialize a CA

```
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

## Docker

### Generate Server and Client Certs

Server Cert

```
$ cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-profile=server \
docker-server-csr.json | cfssljson -bare docker-server
```

Results:

```
docker-server-key.pem
docker-server.csr
docker-server.pem
```

Client Cert

```
$ cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-profile=client \
docker-client-csr.json | cfssljson -bare docker-client
```

Results:

```
docker-client-key.pem
docker-client.csr
docker-client.pem
```

#### Configure Docker

Client

Copy the client certs:

```
$ mkdir -pv ~/.docker
$ cp -v ca.pem ~/.docker/ca.pem
$ cp -v docker-client.pem ~/.docker/cert.pem
$ cp -v docker-client-key.pem ~/.docker/key.pem
```

Fix permissions:

```
$ chmod 0444 ~/.docker/ca.pem
$ chmod 0444 ~/.docker/cert.pem
$ chmod 0400 ~/.docker/key.pem
```

Server

Copy the server certs:

```
$ scp ca.pem docker-server-key.pem docker-server.pem core@docker.kubestack.io:~/
$ ssh core@docker.kubestack.io
$ sudo mv ca.pem /etc/docker/ssl/ca.pem
$ sudo mv docker-server-key.pem /etc/docker/ssl/server-key.pem
$ sudo mv docker-server.pem /etc/docker/ssl/server.pem
```

Fix permissions:

```
$ ssh core@docker.kubestack.io
$ sudo chmod 0444 /etc/docker/ssl/ca.pem
$ sudo chmod 0400 /etc/docker/ssl/server-key.pem
$ sudo chmod 0444 /etc/docker/ssl/server.pem
```

Configure the Docker Engine to use the certs

Edit: /etc/systemd/system/docker.service

```
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
ExecStart=/usr/bin/docker --daemon \
--bip=10.200.0.1/24 \
--host=tcp://0.0.0.0:2376 \
--host=unix:///var/run/docker.sock \
--tlsverify \
--tlscacert=/etc/docker/ssl/ca.pem \
--tlscert=/etc/docker/ssl/server.pem \
--tlskey=/etc/docker/ssl/server-key.pem \
--storage-driver=overlay
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```
$ sudo systemctl start docker
```

## Kubernetes

### Generate Server and Client Certs

#### kube-apiserver

```
$ cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-profile=server \
kube-apiserver-server-csr.json | cfssljson -bare kube-apiserver-server
```

Results:

```
kube-apiserver-server-key.pem
kube-apiserver-server.csr
kube-apiserver-server.pem
```

#### kubectl

```
$ cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-profile=client \
kubernetes-admin-user.csr.json | cfssljson -bare kubernetes-admin-user
```

Results:

```
kubernetes-admin-user-key.pem
kubernetes-admin-user.csr
kubernetes-admin-user.pem
```

### Configure Kubernetes

#### kube-apiserver

Copy the server certs

```
$ scp ca.pem kube-apiserver-server-key.pem kube-apiserver-server.pem core@kube-apiserver.kubestack.io:~/
$ ssh core@kube-apiserver.kubestack.io
$ sudo mkdir -p /etc/kubernetes/kube-apiserver
$ sudo mv ca.pem /etc/kubernetes/kube-apiserver/ca.pem
$ sudo mv kube-apiserver-server-key.pem /etc/kubernetes/kube-apiserver/server-key.pem
$ sudo mv kube-apiserver-server.pem /etc/kubernetes/kube-apiserver/server.pem
```

Fix permissions:

```
$ ssh core@kube-apiserver.kubestack.io
$ sudo chmod 0444 /etc/kubernetes/kube-apiserver/ca.pem
$ sudo chmod 0400 /etc/kubernetes/kube-apiserver/server-key.pem
$ sudo chmod 0444 /etc/kubernetes/kube-apiserver/server.pem
```

Edit: `policy.jsonl`

```
{"user":"admin"}
{"user":"scheduler", "readonly": true, "resource": "pods"}
{"user":"scheduler", "resource": "bindings"}
{"user":"kubelet",  "readonly": true, "resource": "pods"}
{"user":"kubelet",  "readonly": true, "resource": "services"}
{"user":"kubelet",  "readonly": true, "resource": "endpoints"}
{"user":"kubelet", "resource": "events"}
```

```
$ scp policy.jsonl core@kube-apiserver.kubestack.io:~/
$ ssh core@kube-apiserver.kubestack.io
$ sudo mv policy.jsonl /etc/kubernetes/kube-apiserver/policy.jsonl
$ sudo chmod 0644 /etc/kubernetes/kube-apiserver/policy.jsonl
```

Start the Kubernetes Controller containers

```
$ docker-compose -p kubernetes -f compose-controller.yaml up -d
```

#### kubectl

```
$ kubectl config set-cluster secure \
--certificate-authority=ca.pem \
--embed-certs=true \
--server=https://kube-apiserver.kubestack.io:6443
```

```
$ kubectl config set-credentials admin \
--client-key=kubernetes-admin-user-key.pem \
--client-certificate=kubernetes-admin-user.pem \
--embed-certs=true
```

```
$ kubectl config set-context secure \
--cluster=secure \
--user=admin
```

```
$ kubectl config use-context secure
```
