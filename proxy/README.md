# Kubernetes Proxy Configuration using Configuration Containers

```
$ export DOCKER_HOST="tcp://node1.kubestack.io:2376" DOCKER_TLS_VERIFY=1
```

```
$ docker build -t kube-proxy-conf:0.0.1 .
```

## Create the configuration container

Create the configuration container so future containers can mount in the configs using `volumes-from`. 

```
docker create --name kube-proxy-conf-0.0.1 kube-proxy-conf:0.0.1
```

## Usage

```
docker run --net=none --volumes-from=kube-proxy-conf-0.0.1 busybox /bin/ls -lh /etc/kubernetes
```

```
total 16
-rw-r--r--    1 root     root        1.3K Jun 29 01:03 ca.pem
-rw-------    1 root     root        1.6K Jun 29 01:03 client-key.pem
-rw-r--r--    1 root     root        1.4K Jun 29 01:03 client.pem
-rw-r--r--    1 root     root         447 Jun 29 01:05 kubeconfig
```
