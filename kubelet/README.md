# Kubernetes API Server Configuration using Configuration Containers

```
$ export DOCKER_HOST="tcp://node1.kubestack.io:2376" DOCKER_TLS_VERIFY=1
```

```
$ docker build -t kubelet-conf:0.0.1 .
```

## Create the configuration container

Create the configuration container so future containers can mount in the configs using `volumes-from`. 

```
docker create --name kubelet-conf kube-apiserver-conf:0.0.1
```

## Usage

```
docker run --net=none --volumes-from=kubelet-conf busybox /bin/ls -lh /etc/kubernetes
```

```
total 16
-rw-r--r--    1 root     root          1350 Jun 28 17:22 ca.pem
-rw-r--r--    1 root     root           345 Jun 28 02:53 policy.jsonl
-rw-------    1 root     root          1675 Jun 28 17:22 server-key.pem
-rw-r--r--    1 root     root          1464 Jun 28 17:22 server.pem
```
