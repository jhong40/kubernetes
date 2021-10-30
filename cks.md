Suply Chain Security

kybesec # scac yaml 

```
wget https://github.com/controlplaneio/kubesec/releases/download/v2.11.0/kubesec_linux_amd64.tar.gz
tar -xvf  kubesec_linux_amd64.tar.gz
mv kubesec /usr/bin

kubesec scan /root/node.yaml  > /root/kubesec_report.json
# In node.yaml template change privileged: true to privileged: false under securityContext:
kubesec scan /root/node.yaml  > /root/kubesec_report.json

```

trivy

```
#Add the trivy-repo
apt-get  update
apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list

#Update Repo and Install trivy
apt-get update
apt-get install trivy



trivy nginx

docker pull python:3.10.0a4-alpine
trivy image --output /root/python_alpine.txt python:3.10.0a4-alpine

root@controlplane:~# trivy image --output /root/python_alpine.txt python:3.10.0a4-alpine
2021-10-30T23:05:15.429Z        INFO    Detected OS: alpine
2021-10-30T23:05:15.429Z        INFO    Detecting Alpine vulnerabilities...
2021-10-30T23:05:15.430Z        INFO    Number of language-specific files: 1
2021-10-30T23:05:15.430Z        INFO    Detecting python-pkg vulnerabilities...

python:3.10.0a4-alpine (alpine 3.12.3)
======================================
Total: 21 (UNKNOWN: 0, LOW: 2, MEDIUM: 6, HIGH: 10, CRITICAL: 3)


Python (python-pkg)
===================
Total: 1 (UNKNOWN: 0, LOW: 0, MEDIUM: 1, HIGH: 0, CRITICAL: 0)



trivy image --severity HIGH --output /root/python.txt python:3.10.0a4-alpine

root@controlplane:~# trivy image --severity HIGH --output /root/python.txt python:3.10.0a4-alpine
2021-10-30T23:07:22.930Z        INFO    Detected OS: alpine
2021-10-30T23:07:22.930Z        INFO    Detecting Alpine vulnerabilities...
2021-10-30T23:07:22.933Z        INFO    Number of language-specific files: 1
2021-10-30T23:07:22.933Z        INFO    Detecting python-pkg vulnerabilities...

python:3.10.0a4-alpine (alpine 3.12.3)
======================================
Total: 10 (HIGH: 10)


Python (python-pkg)
===================
Total: 0 (HIGH: 0)


trivy image --input alpine.tar --format json --output /root/alpine.json


```
