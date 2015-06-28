# Kubernetes API Server Configuration using Configuration Containers

This is most likely a really bad idea, but I'm going to try it anyway. Use a docker data-only container to manage configuration files for the Kubernetes API server.

## Build the configuration image

I'm only building the configuration image on the Docker host that will use it for configuration. It's possible to store configuration images on a remote Docker registry, but the security risk is very high, especially if the Docker registry is not behind "your" firewall under your complete control.

```
$ export DOCKER_HOST="tcp://node0.kubestack.io:2376" DOCKER_TLS_VERIFY=1
```

```
$ docker build -t kube-apiserver-conf:0.0.1 .
```

## Create the configuration container

Create the configuration container so future containers can mount in the configs using `volumes-from`. 

```
docker create --name kube-apiserver-conf kube-apiserver-conf:0.0.1
```

## Usage

```
docker run --net=none --volumes-from=kube-apiserver-conf busybox /bin/ls -lh /etc/kubernetes
```

```
total 16
-rw-r--r--    1 root     root          1350 Jun 28 17:22 ca.pem
-rw-r--r--    1 root     root           345 Jun 28 02:53 policy.jsonl
-rw-------    1 root     root          1675 Jun 28 17:22 server-key.pem
-rw-r--r--    1 root     root          1464 Jun 28 17:22 server.pem
```
