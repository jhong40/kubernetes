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





```
