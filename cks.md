<details>
  <summary>tmux vi</summary>
  
## tmux - vi usage  
 ```
  tmux 
  ctr-b %  - vertical split 
  ctr-b "  - horizontal split
  ctr-b => arrow 
  ctr-b <= arrow
  
  gg - beging of F
  G - end of F
  H - beging of screen
  L - end of screen
  M - middle of screen
  w - => word
  b - <= word
  
  i - insert br
  a - insert after
  o - insert one line below
  s - replace/insert           
  c - followed by w,$ remove those and insert         
  
  I - insert at the begining of l
  A - insert at the end of l
  O - insert one line above
  S - replace the whole line and insert
  C - delete the rest of line and insert
            
  ESC+v - visual mode
  shift+v - visual mode select line
  ctrol+v - visual mode select block
  move arrow to select
  > 
  <
  
  
  k -n project-c13 describe pod | less -p Requests   # highlight Requests
            
 ```
            
</details>
  
  
<details>
  <summary>Note</summary>
  
## Note  

```
k run busybox3 --rm  --image=busybox -it --restart=Never --command --  echo "hello world"
k run busybox3 --rm  --image=busybox -it --restart=Never --  echo "hello world"
k run busybox3 --rm  --image=busybox -it --restart=Never -- /bin/sh -c 'echo hello world'  
hello world
pod "busybox3" deleted     
```    
    
</details>    
  
  
  

<details>
  <summary>Cluster Setup and Hardening</summary>
  
## Run CIS Benchmark Assessment Tool on Ubuntu
  Center for Internet Security
```
sh ./Assessor-CLI.sh -i -rd /var/www/html/ -nts -rp index   # interactive, report dir, no time stamp, report prefix  
```  
## Kube bench
```
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.4.0/kube-bench_0.4.0_linux_amd64.tar.gz -o kube-bench_0.4.0_linux_amd64.tar.gz
tar -xvf kube-bench_0.4.0_linux_amd64.tar.gz

 ./kube-bench --config-dir `pwd`/cfg --config `pwd`/cfg/config.yaml 

  /etc/kubernetes/manifests/kube-controller-manager.yaml
    - --terminated-pod-gc-threshold=10
    - --feature-gates=RotateKubeletServerCertificate=true
  
 /etc/kubernetes/manifests/kube-scheduler.yaml 
    --profiling=false  
  
./kube-bench --config-dir `pwd`/cfg --config `pwd`/cfg/config.yaml  #### run it again to check fixed  
  
```  
## Service Account
  
```
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2021-11-03T23:53:12Z"
  name: default
  namespace: default
  resourceVersion: "412"
  uid: 206064b9-1f41-49a1-b232-35b8d9cd4e3a
secrets:
- name: default-token-c47kx
  
kubectl create sa dashboard-sa
```  
```yaml
#root@controlplane:/var/rbac# cat pod-reader-role.yaml 
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups:
  - ''
  resources:
  - pods
  verbs:
  - get
  - watch
  - list
  
#root@controlplane:/var/rbac# cat dashboard-sa-role-binding.yaml 
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: dashboard-sa # Name is case sensitive
  namespace: default
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io  
```
```  
      serviceAccountName: dashboard-sa    
```  
  
  
## View Certificate 
```  
root@controlplane:~# k -n kube-system describe pod kube-apiserver-controlplane  | grep crt
      --client-ca-file=/etc/kubernetes/pki/ca.crt
      --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
      --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
      --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
      --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
      --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
root@controlplane:~# k -n kube-system describe pod kube-apiserver-controlplane  | grep key 
      --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
      --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
      --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
      --service-account-key-file=/etc/kubernetes/pki/sa.pub
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
      --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
  

root@controlplane:~# k -n kube-system describe pod etcd-controlplane | grep crt           
      --cert-file=/etc/kubernetes/pki/etcd/server.crt
      --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
      --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
      --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
root@controlplane:~# k -n kube-system describe pod etcd-controlplane | grep key
      --key-file=/etc/kubernetes/pki/etcd/server.key
      --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
  
openssl x509  -noout -text -in /etc/kubernetes/pki/apiserver.crt
  
root@controlplane:/etc/kubernetes/manifests# docker ps | grep api
8499a1973635        ca9843d3b545           "kube-apiserver --ad…"   23 seconds ago      Up 21 seconds  k8s_kube-apiserver_kube-apiserver-controlplane_kube-system_438f60ae442d63c542063736081f2ce9_5
  
docker logs 8499
W1104 01:23:51.568113       1 clientconn.go:1223] grpc: addrConn.createTransport failed to connect to {https://127.0.0.1:2379  <nil> 0 <nil>}. Err :connection error: desc = "transport: Error while dialing dial tcp 127.0.0.1:2379: connect: connection refused". Reconnecting...
  
 
root@controlplane:/etc/kubernetes/manifests# docker ps -a | grep etcd      ############### add -a if docker instance not found.
03a283b32ddd        0369cf4303ff           "etcd --advertise-cl…"   About a minute ago   Up About a minute  k8s_etcd_etcd-controlplane_kube system_39d6dffeffbc33b9c948fe9f59ee7bbb_0
 
docker logs 03a
2021-11-04 01:24:53.703321 I | embed: ready to serve client requests
2021-11-04 01:24:53.703404 I | embed: ready to serve client requests
2021-11-04 01:24:53.703645 C | etcdmain: open /etc/kubernetes/pki/etcd/server-certificate.crt: no such file or directory    ################  
  
## docker ps -a | grep api, docker logs xxx
root@controlplane:~# docker ps -a | grep kube-apiserver
8af74bd23540        ca9843d3b545           "kube-apiserver --ad…"   39 seconds ago      Exited (1) 17 seconds ago                          k8s_kube-apiserver_kube-apiserver-controlplane_kube-system_f320fbaff7813586592d245912262076_4
c9dc4df82f9d        k8s.gcr.io/pause:3.2   "/pause"                 3 minutes ago       Up 3 minutes                                       k8s_POD_kube-apiserve-controlplane_kube-system_f320fbaff7813586592d245912262076_1
  
  
root@controlplane:~# docker logs 8af74bd23540  --tail=2
W0520 01:57:23.333002       1 clientconn.go:1223] grpc: addrConn.createTransport failed to connect to {https://127.0.0.1:2379  <nil> 0 <nil>}. Err :connection error: desc = "transport: authentication handshake failed: x509: certificate signed by unknown authority". Reconnecting...
Error: context deadline exceeded
root@controlplane:~#   
  
grep crt kube-apiserver.yaml 
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --etcd-cafile=/etc/kubernetes/pki/ca.crt      ##################### => /etc/kubernetes/pki/etcd/ca.crt fix
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
root@controlplane:/etc/kubernetes/manifests# !70
grep crt etcd.yaml 
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt      ########################
   
```  
## KubeConfig
```
~/.kube/config
kubectl config view  #count clusters, count users,contexts
kubectl config view --kubeconfig my-kube-config  #  

k --kubeconfig=my-kube-config config use-context research
Switched to context "research".
k --kubeconfig=my-kube-config config current-context
research
  
- name: dev-user
  user:
    client-certificate: /etc/kubernetes/pki/users/dev-user/developer-user.crt
    client-key: /etc/kubernetes/pki/users/dev-user/dev-user.key  
 
  k get pods
error: unable to read client-cert /etc/kubernetes/pki/users/dev-user/developer-user.crt for dev-user due to open /etc/kubernetes/pki/users/dev-user/developer-user.crt: no such file or directory

devloper-user.crt => dev-user.crt  # fix  
  
  
  
  
```  
## RBAC
```
 --authorization-mode=Node,RBAC
  
root@controlplane:~# k -n kube-system describe role kube-proxy
Name:         kube-proxy
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources   Non-Resource URLs  Resource Names  Verbs
  ---------   -----------------  --------------  -----
  configmaps  []                 [kube-proxy]    [get]
  
root@controlplane:~# kubectl describe rolebinding kube-proxy -n kube-system
Name:         kube-proxy
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  Role
  Name:  kube-proxy
Subjects:
  Kind   Name                                             Namespace
  ----   ----                                             ---------
  Group  system:bootstrappers:kubeadm:default-node-token  
  
kubectl create role developer --verb=list,create,delete --resource=pods
kubectl create rolebinding dev-user-binding --role=developer --user=dev-user  
k get pods --as dev-user
  
  
kubectl edit role developer -n blue
  rules:
- apiGroups:
  - ""
  resourceNames:
  - blue-app           ###### => dark-blue-app fix
  resources:
  - pods
  verbs:
  - get
    
kubectl create role foo --verb=get,list,watch --resource=replicasets.apps   ##### => deploments,apps  

- apiGroups:
  - apps
  - extensions
  resources:
  - deployments
  verbs:
  - create
  
  k -n blue auth can-i create deployment --as dev-user
  k -n blue create deployment blah --image=nginx  --as dev-user
   
  
k create role mydeployrole99 --verb=* --resource=deployments
role.rbac.authorization.k8s.io/mydeployrole99 created
root@controlplane:~# k describe role mydeployrole99 
Name:         mydeployrole99
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources         Non-Resource URLs  Resource Names  Verbs
  ---------         -----------------  --------------  -----
  deployments.apps  []                 []              [*]    
  
```   
## Cluster Role and Role Bindins
```
kubectl get clusterroles --no-headers | wc -l   the same as  kubectl get clusterroles --no-headers -o json | jq '.items | length'
kubectl get clusterrolebindings --no-headers | wc -l    the same as  kubectl get clusterrolebindings --no-headers -o json | jq '.items | length'
kubectl describe clusterrole cluster-admin

k create clusterrole mrole --verb=* --resource=nodes
k create clusterrolebinding michelle-mrole --clusterrole=mrole --user=michelle
  
  
root@controlplane:~# k api-resources | grep -i storage
csidrivers                                     storage.k8s.io/v1                      false        CSIDriver
csinodes                                       storage.k8s.io/v1                      false        CSINode
storageclasses                    sc           storage.k8s.io/v1                      false        StorageClass
volumeattachments                              storage.k8s.io/v1                      false        VolumeAttachment
oot@controlplane:~# k api-resources | grep pers
persistentvolumeclaims            pvc          v1                                     true         PersistentVolumeClaim
persistentvolumes                 pv           v1                                     false        PersistentVolume  
  
  
k create clusterrole storage-admin --verb=* --resource=storageclasses --verb=* --resource=persistentvolumes
clusterrole.rbac.authorization.k8s.io/storage-admin created
root@controlplane:~# k describe clusterrole storage-admin 
Name:         storage-admin
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources                      Non-Resource URLs  Resource Names  Verbs
  ---------                      -----------------  --------------  -----
  persistentvolumes              []                 []              [*]
  storageclasses.storage.k8s.io  []                 []              [*]
  
k create clusterrolebinding michelle-storage-admin --clusterrole=storage-admin --user=michelle  
  
kubectl auth can-i list storageclasses --as michelle
Warning: resource 'storageclasses' is not namespace scoped in group 'storage.k8s.io'
yes  
```  

## Kubelet Security
```
/usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.2
  

  grep rotateCertificates /var/lib/kubelet/config.yaml
rotateCertificates: true
  
  10250: full access
  10255: read only access
  
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: true                          ####################
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: AlwaysAllow                        #####################
  
curl -sk https://localhost:10250/pods   # can see pod
  
authorization:
  mode: Webhoook                            #########
curl -sk https://localhost:10250/pods
Forbidden (user=system:anonymous, verb=get, resource=nodes, subresource=proxy)
 
authentication:
  anonymous:
    enabled: false                           ########   
curl -sk https://localhost:10250/pods
Unauthorized  
  
curl -sk http://localhost:10255/metrics   # still work
  
syncFrequency: 0s
volumeStatsAggPeriod: 0s
readOnlyPort: 10255     #########################  => 0
curl -sk http://localhost:10255/metrics   # show nothing  
```  
##  KUBECTL PROXY & PORT FORWARD  
```  
kubectl proxy - Opens proxy port to API server
kubectl port-forward - Opens port to target deployment pods  
  
root@controlplane:~# curl -k https://localhost:6443
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",

kubectl proxy                                         ##########
Starting to serve on 127.0.0.1:8001
curl http://localhost:8001/
curl 127.0.0.1:8001/version
  
kubectl proxy --port 8002 &
curl 127.0.0.1:8002
  
  
kubectl port-forward pods/{POD_NAME} 8005:80 &
kubectl port-forward deployment/{DEPLOYMENT_NAME} 8005:80 &
kubectl port-forward service/{SERVICE_NAME} 8005:80 &
kubectl port-forward replicaset/{REPLICASET_NAME} 8005:80 &

k port-forward deployment/nginx 8005:80 &   ########################
[3] 18975
root@controlplane:~# Forwarding from 127.0.0.1:8005 -> 80
curl localhost:8005
Handling connection for 8005
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>  
 
```

  
## Secure kubernetes Dashboard

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
kubectl proxy --address=0.0.0.0 --disable-filter &
https://8001-port-516697f26a37488c.labs.kodekloud.com/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login

root@controlplane:~# k get sa -A | grep admin-user
kubernetes-dashboard   admin-user  

root@controlplane:~# k -n kubernetes-dashboard describe clusterrolebinding admin-user-binding 
Name:         admin-user-binding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  cluster-admin
Subjects:
  Kind            Name        Namespace
  ----            ----        ---------
  ServiceAccount  admin-user  kubernetes-dashboard  
  
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
eyJhbGciOiJSUzI1NiIsImtpZCI6InNxdHhYUmQtNXpXTy1sdWdMeGt2amt5d2RWQUFrMVJ3ZEJBaEpLZW8tQXMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3Nlcn....
  

#copy the above to login with token on the dashboard UI
# admin-user is too powerful
  
https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles  
  
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: readonly-user
  namespace: kubernetes-dashboard
EOF

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: readonly-user-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view                                                  ##########
subjects:
- kind: ServiceAccount
  name: readonly-user
  namespace: kubernetes-dashboard
EOF
  
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/readonly-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"

## readonly user can only read, let create dashboard-admin to only full access to kubernetes-dashboard ns

 ?????????????????? 
  
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kubernetes-dashboard
EOF


# admin RoleBinding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dashboard-admin-binding
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kubernetes-dashboard
EOF


## list-namespace ClusterRoleBinding

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin-list-namespace-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: list-namespace
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kubernetes-dashboard
EOF  
  
```  
## Verify Platform binary
```
wget -O /opt/kubernetes.tar.gz https://dl.k8s.io/v1.20.0/kubernetes.tar.gz
shasum -a512 /opt/kubernetes.tar.gz       ####################
sha512sum kubernetes.tar.gz     
```  
## Cluster Upgrade
```
kubectl get nodes   # check version 
  
# check which node can host workload  
root@controlplane:~# k describe nodes controlplane | grep -i taint
Taints:             <none>
root@controlplane:~# k describe nodes node01 | grep -i taint
Taints:             <none>
  
k get deploy   # check how many deployment

##### controlplane  
kubeadm upgrade plan
k drain controlplane --ignore-daemonsets
  
apt install kubeadm=1.20.0-00 
kubeadm version 
kubeadm upgrade plan
kubeadm upgrade apply v1.20.0
apt install kubelet=1.20.0-00
kubectl uncordon controlplane
  
#### node
kubectl drain node01 --ignore-daemonsets (--force)
apt update; apt install kubeadm=1.20.0-00
kubeadm upgrade node
apt install kubelet=1.20.0-00
systemctl restart kubelet
k uncordon node01
```  
## Network Security Policy
```
k get netpol
NAME             POD-SELECTOR   AGE
payroll-policy   name=payroll   34s
  
root@controlplane:~# k describe networkpolicies.networking.k8s.io 
Name:         payroll-policy
Namespace:    default
Created on:   2021-11-04 21:29:49 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     name=payroll  #####
  Allowing ingress traffic:
    To Port: 8080/TCP
    From:
      PodSelector: name=internal
  Not affecting egress traffic
  Policy Types: Ingress  
  
 
 ```
 ```yaml
 
 apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
spec:
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          name: mysql
    ports:
    - protocol: TCP
      port: 3306

  - to:
    - podSelector:
        matchLabels:
          name: payroll
    ports:
    - protocol: TCP
      port: 8080

  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
  
kubectl get svc -n kube-system 
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   93m
 
  
```  
## Ingress 1
```  
k get deployment -A | grep -v kube-system
k get ingress -A
root@controlplane:~# k -n app-space describe ingress ingress-wear-watch 
Name:             ingress-wear-watch
Namespace:        app-space
Address:          
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           
              /wear    wear-service:8080 (10.244.0.4:8080)
              /watch   video-service:8080 (10.244.0.7:8080)
Annotations:  nginx.ingress.kubernetes.io/rewrite-target: /
              nginx.ingress.kubernetes.io/ssl-redirect: false
Events:       <none>
  
root@controlplane:~# cat 22.yml 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
  name: pay-ingress 
  namespace: critical-space 
spec:
  rules:
  - http:
      paths:
      - backend:
          service:
            name: pay-service
            port:
              number: 8282
        path: /pay
        pathType: Prefix  
  
```  
## Ingress 2 
```
  
```  
  
</details>  

  
  
  
  
  
  
  
  
  
  
  
  
  
  
  

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
    annotations:                                                                             ########
        container.apparmor.security.beta.kubernetes.io/nginx: localhost/restricted-nginx     ########
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

