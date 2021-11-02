<details>
  <summary>System Hardering</summary>
  
## Limit Node Access
```  
# grep root /etc/passwd
root:x:0:0:root:/root:/bin/bash
# id david
uid=2323(david) gid=2323(david) groups=2323(david) 
# passwd david
Enter new UNIX password: 
# userdel ray
# groupdel devs
# usermod -s /usr/sbin/nologin himanshi
# useradd -d /opt/sam -s /bin/bash -G admin -u 2328 sam  
```  
## SSH Hardening and SUDO 
```
root@controlplane:~# ssh-copy-id -i ~/.ssh/id_rsa.pub jim@node01
jim@node01's password: 
Number of key(s) added: 1
Now try logging into the machine, with:   "ssh 'jim@node01'"
and check to make sure that only the key(s) you wanted were added.

ssh jim@node01  
  
/etc/sudoer
jim     ALL=(ALL:ALL) ALL
jim ALL=(ALL) NOPASSWD:ALL    ############ no password
%admin ALL=(ALL) ALL          ############ user in admin group can sudo  
  
## Create user rob, and make it admin group so that He can sudo   
root@node01:/etc# adduser rob
Adding user `rob' ...
Adding new group `rob' (1002) ...
Adding new user `rob' (1002) with group `rob' ...
Creating home directory `/home/rob' ...
Copying files from `/etc/skel' ...
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
Changing the user information for rob
Enter the new value, or press ENTER for the default
        Full Name []: 
        Room Number []: 
        Work Phone []: 
        Home Phone []: 
        Other []: 
Is the information correct? [Y/n] Y
  
usermod rob -G admin
  
### disable ssh root log, disable pass authentication
#/etc/ssh/sshd_config, systemctl restart sshd
PermitRootLogin no
PasswordAuthentication no   
  
```  
## IDENTIFY OPEN PORTS, REMOVE PACKAGES SERVICES  
```
apt list --installed
apt list --installed | grep python2.7

systemctl list-units -t service  # list only active service
systemctl list-units --all  
lsmod
  
root@controlplane:/etc/systemd/system# systemctl stop nginx
root@controlplane:/etc/systemd/system# systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
rm /lib/systemd/system/nginx.service
  
/etc/modprobe.d/blacklist.conf
#blacklist evbug
blacklist evbug  

apt remove nginx -y

# netstat -tulpn | grep 9090
tcp        0      0 0.0.0.0:9090            0.0.0.0:*               LISTEN      18508/apache2  
systemctl stop apache2
  
root@controlplane:/etc/modprobe.d# apt list --installed | grep wget
wget/now 1.18-5+deb9u3 amd64 [installed,upgradable to: 1.19.4-1ubuntu2.2]  
root@controlplane:/etc/modprobe.d# apt install wget -y   # update to the latest
root@controlplane:/etc/modprobe.d# apt list --installed | grep wget
wget/bionic-updates,bionic-security,now 1.19.4-1ubuntu2.2 amd64 [installed]  
  
```  
## UFW FIREWALL
```
ufw status
ufw status numbered  # show firewall status as numbered list of RULES
ufw allow 1000:2000/tcp
ufw reset  # reset to the default 
root@node01:~# ufw allow 22
Rules updated
Rules updated (v6)
root@node01:~# ufw status
Status: inactive  

root@node01:~# ufw allow from 135.22.65.0/24 to any port 9090 proto tcp
Rules updated
root@node01:~# ufw allow from 135.22.65.0/24 to any port 9091 proto tcp
Rules updated
root@node01:~# ufw enable
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup

systemctl status lighttpd  
netstat -natulp | grep lighttpd
ufw deny 80
  
ufw disable  
```  
## Seccomp
```
strace -c ls /root

docker run --name tracee --rm --privileged -v /lib/modules/:/lib/modules/:ro -v /usr/src:/usr/src:ro -v /tmp/tracee:/tmp/tracee -it aquasec/tracee:0.4.0 --trace container=new

1471.405184    hello            0      pause            1      /14065   1      /14065   0                sched_process_exit   

root@controlplane:~# head -20 custom-profile.json 
{
    "defaultAction": "SCMP_ACT_ERRNO",  # need to create white list
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    "syscalls": [
        {
            "names": [
                "accept4",
                "epoll_wait",
                "pselect6",
                "futex",
                "madvise",
  
 
root@controlplane:~# head relaxed-profile.json 
{
    "defaultAction": "SCMP_ACT_ALLOW",  # need to create black list 
    "architectures": [  
  
oot@controlplane:/var/lib/kubelet/seccomp# cd profiles/
root@controlplane:/var/lib/kubelet/seccomp/profiles# ls
audit.json  custom-profile.json  relaxed-profile.json  violation.json
root@controlplane:/var/lib/kubelet/seccomp/profiles#   

```  
```yaml
  
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: audit-nginx
spec:
  securityContext:                              ########
    seccompProfile:                             ########
      type: Localhost                           ########
      localhostProfile: profiles/audit.json     ######## relative /var/lib/kubelet/seccomp
  containers:
  - image: nginx
    name: nginx  
  
```  
## AppArmor    
```  
root@controlplane:/# aa-status
apparmor module is loaded.
56 profiles are loaded.
19 profiles are in enforce mode.
   /sbin/dhclient
   /usr/bin/lxc-start

root@controlplane:/# kubectl describe pod
Name:         nginx
Namespace:    default
Priority:     0
Node:         controlplane/192.168.121.28
Start Time:   Tue, 02 Nov 2021 20:24:08 +0000
Labels:       run=nginx
Annotations:  container.apparmor.security.beta.kubernetes.io/nginx: localhost/custom-nginx   ##############3
Status:       Pending
Reason:       AppArmor
Message:      Cannot enforce AppArmor: profile "custom-nginx" is not loaded    ###############

root@controlplane:/etc/apparmor.d# cat usr.sbin.nginx
#include <tunables/global>

profile custom-nginx flags=(attach_disconnected,mediate_deleted) {    ################
  #include <abstractions/base>

  network inet tcp,
  network inet udp,
  network inet icmp,

  deny network raw,
  
apparmor_parser -q /etc/apparmor.d/usr.sbin.nginx   ######## load profile, pod blocked ==> running
 
cat /etc/apparmor.d/usr.sbin.nginx-updated
profile restricted-nginx flags=(attach_disconnected,mediate_deleted) {
...
  deny /bin/** wl,
  deny /usr/share/nginx/html/restricted/* rw,      ############ deny restricted folder
  
apparmor_parser -q /etc/apparmor.d/usr.sbin.nginx-updated   ######## load above profile
```
```yaml
apiVersion: v1
kind: Pod
metadata:
    creationTimestamp: null
    labels:
        run: nginx
    name: nginx
    annotations:
        container.apparmor.security.beta.kubernetes.io/nginx: localhost/restricted-nginx
spec:
    containers:
        -
            image: 'nginx:alpine'
```
```  
http://vhost/allowed/  
http:/vhost/restricted/  ## denied
  
```  
</details>  











<details>
  <summary>Minimize Microservice Vulnerabilities</summary>
  
## Security Contexts
```
kubectl exec ubuntu-sleeper -- whoami
```
jim ALL=(ALL) NOPASSWD:ALL``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  securityContext:
    runAsUser: 1001    ###
  containers:
  -  image: ubuntu
     name: web
     command: ["sleep", "5000"]
     securityContext:
       runAsUser: 1002   ###
       capabilities:
         add: ["SYS_TIME"]  ###    
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

kubectl create -f /root/webhook-configuration.yaml    #############
```
```yaml
# /root/webhook-configuration.yaml                          ################
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
#############################
None =>   securityContext:       ### add securityContext
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
```
ps -ef | grep api | grep able  # check enable-admission-plugins or disable-admission-plugins
kube-apiserver.yaml
    - --enable-admission-plugins=NodeRestriction,PodSecurityPolicy                 #############

root@controlplane:~# cat psp.yaml 
apiVersion: policy/v1beta1
kind: PodSecurityPolicy     #################
metadata:
  name: example-psp
spec:
  privileged: false
  seLinux:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - configMap
  - secret
  - emptyDir
  - hostPath

kubectl apply -f /root/psp.yaml

root@controlplane:~# kubectl apply -f pod.yaml        
Error from server (Forbidden): error when creating "pod.yaml": pods "example-app" is forbidden: PodSecurityPolicy: unable to admit pod: [spec.containers[0].securityContext.privileged: Invalid value: true: Privileged containers are not allowed spec.containers[0].securityContext.capabilities.add: Invalid value: "CAP_SYS_BOOT": capability may not be added]
 
```
## OPA - Open Policy Agent (port 8181, rego)
```
export VERSION=v0.27.1
curl -L -o opa https://github.com/open-policy-agent/opa/releases/download/${VERSION}/opa_linux_amd64
chmod 755 ./opa
./opa run -s &  

root@controlplane:~# cat example.rego 
package httpapi.authz
import input
default allow =     # default allow = flase  
allow {
 input.path == "home"
 input.user == "Kedar"
 }
  
./opa test example.rego                   ##################
1 error occurred during loading: example.rego:3: rego_parse_error: illegal default rule (value cannot contain var)
        default allow = 
        ^

# load the policy file after fix the: default allow = false
curl -X PUT --data-binary @sample.rego http://localhost:8181/v1/policies/samplepolicy    ###################
 
  
```  
## OPA in Kubernetes
```
kube-mgmt: Replicate kube resource to OPA, Load Policy into OPA via kubernetes
root@controlplane:~# cat untrusted-registry.rego 
package kubernetes.admission

deny[msg] {
  input.request.kind.kind == "Pod"
  image := input.request.object.spec.containers[_].image
  not startswith(image, "hooli.com/")
  msg := sprintf("image '%v' comes from untrusted registry", [image])
}

root@controlplane:~# cat test.yaml 
kind: Pod
apiVersion: v1
metadata:
  name: test
spec:
  containers:
  - image: nginx
    name: nginx-frontend
  - image: hooli.com/mysql
    name: mysql-backend  

kubectl create configmap untrusted-registry --from-file=untrusted-registry.rego

root@controlplane:~# kubectl apply -n dev -f /root/test.yaml
Error from server (image 'nginx' comes from untrusted registry): error when creating "/root/test.yaml": admission webhook "validating-webhook.openpolicyagent.org" denied the request: image 'nginx' comes from untrusted registry
  
# fix it with 
image: hooli.com/nginx  

root@controlplane:~# cat unique-host.rego 
package kubernetes.admission
import data.kubernetes.ingresses

deny[msg] {
    some other_ns, other_ingress
    input.request.kind.kind == "Ingress"
    input.request.operation == "CREATE"
    host := input.request.object.spec.rules[_].host
    ingress := ingresses[other_ns][other_ingress]
    other_ns != input.request.namespace
    ingress.spec.rules[_].host == host
    msg := sprintf("invalid ingress host %q (conflicts with %v/%v)", [host, other_ns, other_ingress])
}  
  
kubectl create configmap unique-host --from-file=/root/unique-host.rego
root@controlplane:~# cat ingress-test-1.yaml 
apiVersion: networking.k8s.io/v1 
kind: Ingress
metadata:
  name: prod
  namespace: test-1
spec:
  rules:
  - host: initech.com
    http:
      paths:
      - path: /finance-1
        pathType: Prefix
        backend:
          service:
            name: banking
            port: 
              number: 443
root@controlplane:~# kubectl apply -f /root/ingress-test-1.yaml
ingress.networking.k8s.io/prod created
  
root@controlplane:~# cat ingress-test-2.yaml 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prod
  namespace: test-2
spec:
  rules:
  - host: initech.com
    http:
      paths:
      - path: /finance-2
        pathType: Prefix
        backend:
          service:
            name: banking
            port: 
              number: 443
root@controlplane:~# kubectl apply -f /root/ingress-test-2.yaml
Error from server (invalid ingress host "initech.com" (conflicts with test-1/prod)): error when creating "/root/ingress-test-2.yaml": admission webhook "validating-webhook.openpolicyagent.org" denied the request: invalid ingress host "initech.com" (conflicts with test-1/prod)
root@controlplane:~# 
  
  
  
 
  
```  
## Manage Kubernetes secrets
```
root@controlplane:~# k describe secrets default-token-26hjx 
Name:         default-token-26hjx
Namespace:    default
...
Type:  kubernetes.io/service-account-token
Data
====
ca.crt:     1066 bytes
namespace:  7 bytes
token:    eyJhbGciOiJSUzI..
  
kubectl create secret generic db-secret --from-literal=DB_Host=sql01 --from-literal=DB_User=root --from-literal=DB_Password=password123  ########

apiVersion: v1 
kind: Pod 
metadata:
  labels:
    name: webapp-pod
  name: webapp-pod
  namespace: default 
spec:
  containers:
  - image: kodekloud/simple-webapp-mysql
    imagePullPolicy: Always
    name: webapp
    envFrom:
    - secretRef:
        name: db-secret     #############  
  
  
```  
## Using Runtime in kubernetes (gvisor, kata)
```
root@controlplane:~# docker info | grep -i runtime   ################
WARNING: No swap limit support
 Runtimes: runc
 Default Runtime: runc
  
root@controlplane:~# k get runtimeclasses   #####################3
NAME              HANDLER        AGE
gvisor            runsc          5m15s
kata-containers   kata-runtime   5m14s
  
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
    name: secure-runtime
handler: runsc                      ##########################
 
root@controlplane:~# k get runtimeclasses.node.k8s.io 
NAME              HANDLER        AGE
gvisor            runsc          7m23s
kata-containers   kata-runtime   7m22s
secure-runtime    runsc          4s    #########################
  
root@controlplane:~# cat 6.yml 
apiVersion: v1
kind: Pod
metadata:
    name: simple-webapp-1
    labels:
        name: simple-webapp
spec:
   runtimeClassName: secure-runtime          ############################
   containers:
     - name: simple-webapp
       image: kodekloud/webapp-delayed-start
       ports:
        - containerPort: 8080
  
  
  
  
  
```  
  
## Implement Pod to Pod encryption by mTLS
</details>



<details>
  <summary>Suply Chain Security</summary>

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
</details>  
  
<details>
  <summary>Monitoring, Logging and Runtime Security</summary>
  
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
</details>

