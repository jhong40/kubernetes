
# Minimize Microservice Vulnerabilities
## Security Contexts
```
**kubectl exec ubuntu-sleeper -- whoami**   
```
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  securityContext:
    runAsUser: 1001
  containers:
  -  image: ubuntu
     name: web
     command: ["sleep", "5000"]
     securityContext:
       runAsUser: 1002
       capabilities:
         add: ["SYS_TIME"]      
  -  image: ubuntu
     name: sidecar
     command: ["sleep", "5000"]
```
## Validating and Mutating Admission Controllers
webhook function
- Denies all request for pod to run as root in container if no securityContext is provided.
- If no value is set for runAsNonRoot, a default of true is applied, and the user ID defaults to 1234
- Allow to run containers as root if runAsNonRoot set explicitly to false in the securityContext
 
```
kubectl create ns webhook-demo
kubectl -n webhook-demo create secret tls webhook-server-tls \
    --cert "/root/keys/webhook-server-tls.crt" \
    --key "/root/keys/webhook-server-tls.key"
kubectl create -f /root/webhook-deployment.yaml
kubectl create -f /root/webhook-service.yaml

kubectl create -f /root/webhook-configuration.yaml
    
```
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webhook-server
  namespace: webhook-demo
  labels:
    app: webhook-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webhook-server
  template:
    metadata:
      labels:
        app: webhook-server
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1234
      containers:
      - name: server
        image: stackrox/admission-controller-webhook-demo:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8443
          name: webhook-api
        volumeMounts:
        - name: webhook-tls-certs
          mountPath: /run/secrets/tls
          readOnly: true
      volumes:
      - name: webhook-tls-certs
        secret:
          secretName: webhook-server-tls        
```
``` yaml
apiVersion: v1
kind: Service
metadata:
  name: webhook-server
  namespace: webhook-demo
spec:
  selector:
    app: webhook-server
  ports:
    - port: 443
      targetPort: webhook-api
```
```yaml
# /root/webhook-configuration.yaml
#apiVersion: admissionregistration.k8s.io/v1beta1
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: demo-webhook
webhooks:
  - name: webhook-server.webhook-demo.svc
    sideEffects: NoneOnDryRun   ### v1
    admissionReviewVersions: ["v1", "v1beta1"]  ### v1  
    clientConfig:
      service:
        name: webhook-server
        namespace: webhook-demo
        path: "/mutate"
      caBundle: xxxx    
    rules:     
      - operations: [ "CREATE" ]        
        apiGroups: [""]        
        apiVersions: ["v1"]        
        resources: ["pods"]
```
```
None =>   securityContext:
            runAsNonRoot: true
            runAsUser: 1234
           
securityContext:     =>     securityContext:
   runAsNonRoot: false        runAsNonRoot: false  

securityContext:
  runAsNonRoot: true
  runAsUser: 0
root@controlplane:~# kubectl apply -f /root/pod-with-conflict.yaml
Error from server: error when creating "/root/pod-with-conflict.yaml": admission webhook "webhook-server.webhook-demo.svc" denied the request: runAsNonRoot specified, but runAsUser set to 0 (the root user)
```

## Pod Security Policy
## OPA
## Manage Kubernetes secrets
## Using Runtime in kubernetes (gvisor, kata)
## Implement Pod to Pod encryption by mTLS

# Suply Chain Security

## image security

```
  - image: myprivateregistry.com:5000/nginx:alpine


kubectl create secret docker-registry private-reg-cred --docker-username=dock_user --docker-password=dock_password --docker-server=myprivateregistry.com:5000 --docker-email=dock_user@myprivateregistry.com

apiVersion: v1
kind: Pod
metadata:
  name: foo
  namespace: awesomeapps
spec:
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
  imagePullSecrets:
    - name: myregistrykey
```    
    


## white list allowed registry  # enforce image not using latest
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


## kybesec # scan kube manifest yaml 

```
wget https://github.com/controlplaneio/kubesec/releases/download/v2.11.0/kubesec_linux_amd64.tar.gz
tar -xvf  kubesec_linux_amd64.tar.gz
mv kubesec /usr/bin

kubesec scan /root/node.yaml  > /root/kubesec_report.json
# In node.yaml template change privileged: true to privileged: false under securityContext:
kubesec scan /root/node.yaml  > /root/kubesec_report.json

```

## trivy  # image

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

# Monitoring, Logging and Runtime Security

## Falco
```
systemctl status falco
journalctl -u falco

/etc/falco/falco.yaml

Oct 31 03:25:50 controlplane falco[632]: 03:25:50.514987067: Error Package management process launched in container (user=root user_loginuid=-1 command=apt update container_id=316f59cd09a1 container_name=k8s_simple-webapp-1_simple-webapp-1_critical-apps_37a12c57-f0a5-4768-8016-8f60dd6af7d3_0 image=nginx:latest)

Oct 31 03:28:29 controlplane falco[632]: 03:28:29.117105566: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=bash parent=bash file=/root/compromised_pods.txt program=bash container_id=host image=<NA>)

root@controlplane:/# grep -ir 'Package management process launched in container' /etc/falco/ 
/etc/falco/falco_rules.yaml:    Package management process launched in container (user=%user.name user_loginuid=%user.loginuid
root@controlplane:/# 

/etc/falco/falco_rules.yaml
# Container is supposed to be immutable. Package management should be done in building the image.
- rule: Launch Package Management Process in Container
  desc: Package management process ran inside container
  condition: >
    spawned_process
    and container
    and user.name != "_apt"
    and package_mgmt_procs
    and not package_mgmt_ancestor_procs
    and not user_known_package_manager_in_container
  output: >
    Package management process launched in container (user=%user.name user_loginuid=%user.loginuid
    command=%proc.cmdline container_id=%container.id container_name=%container.name image=%container.image.repository:%container.image.tag)
  priority: ERROR
  tags: [process, mitre_persistence]
  
  
  
  /etc/falco/falco_rules.local.yaml
  - rule: Launch Package Management Process in Container
  desc: Package management process ran inside container
  condition: >
    spawned_process
    and container
    and user.name != "_apt"
    and package_mgmt_procs
    and not package_mgmt_ancestor_procs
    and not user_known_package_manager_in_container
  output: >
    Package Management Tools Executed (user=%user.name command=%proc.cmdline container_id=%container.id)
  priority: ERROR
  tags: [process, mitre_persistence]
  
  kill -1 $(cat /var/run/falco.pid)
```
## ENSURE IMMUTABILITY OF CONTAINERS AT RUNTIME
```
    securityContext:
      privileged: true               #need: not true    withtout: container can be priviliged
      readOnlyRootFilesystem: true   #need: true    without it: pod can write to root file system
      

apiVersion: v1
kind: Pod
metadata:
  labels:
    name: triton
  name: triton
  namespace: alpha
spec:
  containers:
  - image: httpd
    name: triton
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - mountPath: /usr/local/apache2/logs
      name: log-volume
  volumes:
  - name: log-volume
    emptyDir: {}      
      
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
```
## Audit
```
Create /etc/kubernetes/prod-audit.yaml as below:

apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  namespaces: ["prod"]
  verbs: ["delete"]
  resources:
  - group: ""
    resources: ["secrets"]
    
Next, make sure to enable logging in api-server:

 - --audit-policy-file=/etc/kubernetes/prod-audit.yaml
 - --audit-log-path=/var/log/prod-secrets.log
 - --audit-log-maxage=30
 
Then, add volumes and volume mounts as shown in the below snippets.
volumes:

  - name: audit
    hostPath:
      path: /etc/kubernetes/prod-audit.yaml
      type: File

  - name: audit-log
    hostPath:
      path: /var/log/prod-secrets.log
      type: FileOrCreate
volumeMounts:

  - mountPath: /etc/kubernetes/prod-audit.yaml
    name: audit
    readOnly: true
  - mountPath: /var/log/prod-secrets.log
    name: audit-log
    readOnly: false

then save the file and make sure that kube-apiserver restarts.


{"kind":"Event","apiVersion":"audit.k8s.io/v1","level":"Metadata","auditID":"9787902b-0abe-4fb3-9c6e-4a090caa26c7","stage":"RequestReceived","requestURI":"/api/v1/namespaces/prod/secrets/blah","verb":"delete","user":{"username":"kubernetes-admin","groups":["system:masters","system:authenticated"]},"sourceIPs":["10.41.5.3"],"userAgent":"kubectl/v1.20.0 (linux/amd64) kubernetes/af46c47","objectRef":{"resource":"secrets","namespace":"prod","name":"blah","apiVersion":"v1"},"requestReceivedTimestamp":"2021-11-01T17:33:36.892980Z","stageTimestamp":"2021-11-01T17:33:36.892980Z"}
{"kind":"Event","apiVersion":"audit.k8s.io/v1","level":"Metadata","auditID":"9787902b-0abe-4fb3-9c6e-4a090caa26c7","stage":"ResponseComplete","requestURI":"/api/v1/namespaces/prod/secrets/blah","verb":"delete","user":{"username":"kubernetes-admin","groups":["system:masters","system:authenticated"]},"sourceIPs":["10.41.5.3"],"userAgent":"kubectl/v1.20.0 (linux/amd64) kubernetes/af46c47","objectRef":{"resource":"secrets","namespace":"prod","name":"blah","apiVersion":"v1"},"responseStatus":{"metadata":{},"status":"Success","code":200},"requestReceivedTimestamp":"2021-11-01T17:33:36.892980Z","stageTimestamp":"2021-11-01T17:33:36.900224Z","annotations":{"authorization.k8s.io/decision":"allow","authorization.k8s.io/reason":""}}

```


