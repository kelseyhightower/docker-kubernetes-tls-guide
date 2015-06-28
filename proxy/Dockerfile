FROM scratch
MAINTAINER Kelsey Hightower <kelsey.hightower@gmail.com>
COPY kube-proxy-client-key.pem /etc/kubernetes/client-key.pem
COPY kube-proxy-client.pem /etc/kubernetes/client.pem
COPY ca.pem /etc/kubernetes/ca.pem
COPY proxy.kubeconfig /etc/kubernetes/kubeconfig
VOLUME /etc/kubernetes
CMD ["/bin/true"]
