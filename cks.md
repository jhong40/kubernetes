# Suply Chain Security

## white list allowed registry
```
kubectl apply -f image-policy-webhook.yaml
   image-bouncer-webhook:30080

#     --registry-whitelist=docker.io,k8s.gcr.io

/etc/kubernetes/pki/admission_configuration.yaml  # refer to kubeconfigfile
  kubeConfigFile: /etc/kubernetes/pki/admission_kube_config.yaml
  
root@controlplane:/etc/kubernetes/pki# cat admission_configuration.yaml 
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/pki/admission_kube_config.yaml 
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false  

kubeConfigFile: /etc/kubernetes/pki/admission_kube_config.yaml  # https://image-bouncer-webhook:30080
  server: https://image-bouncer-webhook:30080/image_policy


/etc/kubernetes/manifests/kube-apiserver.yaml
    - --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
    - --admission-control-config-file=/etc/kubernetes/pki/admission_configuration.yaml


kubectl apply -f /root/nginx-latest.yml
replicaset.apps/nginx-latest created

kubectl describe replicaset nginx-latest
Error from server (Forbidden): pods "nginx" is forbidden: image policy webhook backend denied one or more images: Images using latest tag are not allowed

 image: nginx:1.19
 kubectl apply -f /root/nginx-latest.yml
 
```


## kybesec # scac yaml 

```
wget https://github.com/controlplaneio/kubesec/releases/download/v2.11.0/kubesec_linux_amd64.tar.gz
tar -xvf  kubesec_linux_amd64.tar.gz
mv kubesec /usr/bin

kubesec scan /root/node.yaml  > /root/kubesec_report.json
# In node.yaml template change privileged: true to privileged: false under securityContext:
kubesec scan /root/node.yaml  > /root/kubesec_report.json

```

## trivy

```
#Add the trivy-repo
apt-get  update
apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list

#Update Repo and Install trivy
apt-get update
apt-get install trivy


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
