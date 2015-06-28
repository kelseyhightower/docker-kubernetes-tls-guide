FROM scratch
MAINTAINER Kelsey Hightower <kelsey.hightower@gmail.com>
COPY kubelet-client-key.pem /etc/kubernetes/client-key.pem
COPY kubelet-client.pem /etc/kubernetes/client.pem
COPY ca.pem /etc/kubernetes/ca.pem
COPY kubelet.kubeconfig /etc/kubernetes/kubeconfig
VOLUME /etc/kubernetes
