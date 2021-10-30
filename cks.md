Suply Chain Security

kybesec # scac kube cluster

```
wget https://github.com/controlplaneio/kubesec/releases/download/v2.11.0/kubesec_linux_amd64.tar.gz

tar -xvf  kubesec_linux_amd64.tar.gz

mv kubesec /usr/bin

kubesec scan /root/node.yaml  > /root/kubesec_report.json

In node.yaml template change privileged: true to privileged: false under securityContext:

kubesec scan /root/node.yaml  > /root/kubesec_report.json

```
