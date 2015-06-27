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

```
$ mkdir -pv ~/.docker
$ cp -v ca.pem ~/.docker/ca.pem
$ cp -v docker-client.pem ~/.docker/cert.pem
$ cp -v docker-client-key.pem ~/.docker/key.pem
```

Server

```
$ scp ca.pem docker-server-key.pem docker-server.pem core@docker.kubestack.io:~/
$ ssh core@docker.kubestack.io
$ sudo mv ca.pem /etc/docker/ssl/ca.pem
$ sudo mv docker-server-key.pem /etc/docker/ssl/server-key.pem
$ sudo mv docker-server.pem /etc/docker/ssl/server.pem
```

```
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
