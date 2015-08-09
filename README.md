# Docker and Kubernetes TLS Guide

This guide will walk you through setting up TLS and TLS cert client authentication for Docker and Kubernetes.

## Install CFSSL

The first step in securing Docker and Kubernetes is to set up a PKI infrastructure for managing TLS certificates.

https://github.com/cloudflare/cfssl

## Review and customize CSRs

The CFSSL tool takes various JSON configuration files to initial a CA and produce certificates. Clone this repo and review the current set of configs and adjust them for you environment.

```
$ git clone https://github.com/kelseyhightower/docker-kubernetes-tls-guide.git 
```

## Initialize a CA

Before we can generate any certs we need to initialize a CA.

```
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

## Docker

The Docker daemon can be [protected using TLS certificates](https://docs.docker.com/articles/https), but instead of using the openssl tools we are going to leverage our PKI from above.

### Generate Server and Client Certs

#### Docker Engine

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

#### Docker Client

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

### Configure Docker

#### Docker Daemon

Copy the server certs to the Docker host.

```
$ scp ca.pem docker-server-key.pem docker-server.pem core@docker.kubestack.io:~/
```

Move the server certs into place and fix permissions.

```
$ ssh core@docker.kubestack.io
$ sudo mv ca.pem /etc/docker/ca.pem
$ sudo mv docker-server-key.pem /etc/docker/server-key.pem
$ sudo mv docker-server.pem /etc/docker/server.pem
$ sudo chmod 0444 /etc/docker/ca.pem
$ sudo chmod 0400 /etc/docker/server-key.pem
$ sudo chmod 0444 /etc/docker/server.pem
```

Configure the Docker daemon to use the certs.

```
cat > /etc/systemd/system/docker.service <<EOF
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
ExecStart=/usr/bin/docker --daemon \
--bip=10.200.0.1/24 \
--host=tcp://0.0.0.0:2376 \
--host=unix:///var/run/docker.sock \
--tlsverify \
--tlscacert=/etc/docker/ca.pem \
--tlscert=/etc/docker/server.pem \
--tlskey=/etc/docker/server-key.pem \
--storage-driver=overlay
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Start or restart the Docker daemon

```
$ sudo systemctl start docker
```

```
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

#### Client

Copy the client certs to the Docker client config dir.

```
$ mkdir -pv ~/.docker
$ cp -v ca.pem ~/.docker/ca.pem
$ cp -v docker-client.pem ~/.docker/cert.pem
$ cp -v docker-client-key.pem ~/.docker/key.pem
$ chmod 0444 ~/.docker/ca.pem
$ chmod 0444 ~/.docker/cert.pem
$ chmod 0400 ~/.docker/key.pem
```

```
$ export DOCKER_HOST="tcp://docker.kubestack.io:2376" DOCKER_TLS_VERIFY=1
$ docker ps
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

#### kubelet

```
$ cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-profile=client \
kubelet-client-csr.json | cfssljson -bare kubelet-client
```

Results:

```
kubelet-client-key.pem
kubelet-client.csr
kubelet-client.pem
```

#### kube-proxy

```
$ cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-profile=client \
kube-proxy-client-csr.json | cfssljson -bare kube-proxy-client
```

Results

```
kube-proxy-client-key.pem 
kube-proxy-client.csr
kube-proxy-client.pem
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

#### Controllers

A Kubernetes controller includes the API server, Controller Manager, and the Scheduler.

Copy the server certs to the Kubernetes API server.

```
$ scp ca.pem kube-apiserver-server-key.pem kube-apiserver-server.pem core@kube-apiserver.kubestack.io:~/
$ ssh core@kube-apiserver.kubestack.io
$ sudo mkdir -p /etc/kubernetes/kube-apiserver
$ sudo mv ca.pem /etc/kubernetes/kube-apiserver/ca.pem
$ sudo mv kube-apiserver-server-key.pem /etc/kubernetes/kube-apiserver/server-key.pem
$ sudo mv kube-apiserver-server.pem /etc/kubernetes/kube-apiserver/server.pem
$ sudo chmod 0444 /etc/kubernetes/kube-apiserver/ca.pem
$ sudo chmod 0400 /etc/kubernetes/kube-apiserver/server-key.pem
$ sudo chmod 0444 /etc/kubernetes/kube-apiserver/server.pem
```

```
cat > policy.jsonl <<EOF
{"user":"admin"}
{"user":"scheduler", "readonly": true, "resource": "pods"}
{"user":"scheduler", "resource": "bindings"}
{"user":"kubelet",  "readonly": true, "resource": "pods"}
{"user":"kubelet",  "readonly": true, "resource": "services"}
{"user":"kubelet",  "readonly": true, "resource": "endpoints"}
{"user":"kubelet", "resource": "events"}
EOF
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

#### Workers

A Kubernetes worker includes the kubelet and the proxy.

Copy the client certs to the worker node.

```
$ scp ca.pem core@node0.kubestack.io:~/
$ scp kube-proxy-client-key.pem kube-proxy-client.pem core@node0.kubestack.io:~/
$ scp kubelet-client-key.pem kubelet-client.pem core@node0.kubestack.io:~/
```

```
$ ssh core@node0.kubestack.io
$ sudo mkdir -p /etc/kubernetes
$ sudo mv ca.pem /etc/kubernetes/ca.pem
$ sudo mv kube-proxy-client-key.pem /etc/kubernetes/kube-proxy/client-key.pem
$ sudo mv kube-proxy-client.pem /etc/kubernetes/kube-proxy/client.pem
$ sudo mv kubelet-client-key.pem /etc/kubernetes/kubelet/client-key.pem
$ sudo mv kubelet-client.pem /etc/kubernetes/kubelet/client.pem
$ sudo chmod 0444 /etc/kubernetes/ca.pem
$ sudo chmod 0400 /etc/kubernetes/kube-proxy/client-key.pem
$ sudo chmod 0444 /etc/kubernetes/kube-proxy/client.pem
$ sudo chmod 0400 /etc/kubernetes/kubelet/client-key.pem
$ sudo chmod 0444 /etc/kubernetes/kubelet/client.pem
```

Copy the kubeconfigs for the kubelet and proxy services.

```
$ scp kubeconfigs/kubelet.kubeconfig core@node0.kubestack.io:~/
$ scp kubeconfigs/proxy.kubeconfig core@node0.kubestack.io:~/
```

```
$ ssh core@node0.kubestack.io
$ sudo mv kubelet.kubeconfig /etc/kubernetes/kubelet/
$ sudo mv proxy.kubeconfig /etc/kubernetes/kube-proxy/
$ sudo chmod 0444 /etc/kubernetes/kubelet/kubelet.kubeconfig
$ sudo chmod 0444 /etc/kubernetes/kube-proxy/proxy.kubeconfig
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

```
$ kubectl cluster-info
$ kubectl get cs
```
