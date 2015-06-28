FROM scratch
MAINTAINER Kelsey Hightower <kelsey.hightower@gmail.com>
COPY kube-apiserver-server-key.pem /etc/kubernetes/server-key.pem
COPY kube-apiserver-server.pem /etc/kubernetes/server.pem
COPY ca.pem /etc/kubernetes/ca.pem
COPY policy.jsonl /etc/kubernetes/policy.jsonl
VOLUME /etc/kubernetes
CMD ["/bin/true"]
