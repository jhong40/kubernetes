- [katacoda](https://www.katacoda.com/learn)
- [k8splayground](https://labs.play-with-k8s.com)
- [killercode](https://killercoda.com/)

- https://krew.sigs.k8s.io/docs/user-guide/setup/install/ 
- https://artifacthub.io/packages/krew/krew-index/neat?modal=install
- https://github.com/kodekloudhub/certified-kubernetes-administrator-course
- https://www.cncf.io/wp-content/uploads/2020/08/CNCF-Presentation-Template-K8s-Deployment.pdf

### Command and Args
```
apiVersion: v1
kind: Pod
metadata:
  name: command-demo
  labels:
    purpose: demonstrate-command
spec:
  containers:
  - name: command-demo-container
    image: debian
    # command: ["printenv"]
    # args: ["HOSTNAME", "KUBERNETES_PORT"]
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
```
```
apiVersion: v1
kind: Pod
metadata:
  name: command-demo
  labels:
    purpose: demonstrate-command
spec:
  containers:
  - name: command-demo-container
    image: debian
    command: ["/bin/sh", "-c"]
    args:
      - echo "aaa";
        echo "bbb";
        echo "ccc";
        sleep 3600
  restartPolicy: OnFailure
```

# 7 Security
```
user/pass, user/token, cert, ldap(3rd party)
cat user-details.csv
   password123,user1,u0001,group1
   password123,user2,u0002,group2
  --basic-auth-file=user-details.csv in kube-apiserver.service or /etc/kubernetes/manifests/kube-apiserver.yaml (create volume attached)
  curl -v -k https://apiserverurl/api/v1/pods -u "user1:password123"
cat user-token-details.csv
   dkkkdfllflf1dkd,user1,u0001,group1
   2kkkdfll2lf1dkd,user2,u0002,group2
  --token-auth-file=user-token-details.csv in kube-apiserver.service or /etc/kubernetes/manifests/kube-apiserver.yaml (create volume attached)
    curl -v -k https://apiserverurl/api/v1/pods --header "Authorization: Bearer dkkkdfllflf1d"
```
### TLS
```
  ssh-keygen => id_rsa, id_rsa.pub
  user1 => host1  (~/.ssh/authorized_keys: ssh-rsa id_rsa.pub) 
  ssh -i id_rsa user1@host1
  
  PKI (CA, Cert/Key, server, browser)
  public key: .crt .pem
  private key: .key, -key.pem
   
openssl genrsa -out ca.key 2048  # generate key => ca.key
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr  # Cert signing request => ca.csr
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt  # Sign cert => ca.crt

openssl genrsa -out admin.key 2048 
openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr
openssl x509 -req -in admin.csr –CA ca.crt -CAkey ca.key -out admin.crt  # api server key,crt

### api server key,crt 
openssl genrsa -out apiserver.key 2048 
openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr -config openssl.cnf  # ->apiserver.csr
openssl.cnf:
  [req]
  req_extensions = v3_req
  [ v3_req ]
  basicConstraints = CA:FALSE
  keyUsage = nonRepudiation,
  subjectAltName = @alt_names
  [alt_names]
  DNS.1 = kubernetes
  DNS.2 = kubernetes.default
  DNS.3 = kubernetes.default.svc
  DNS.4 = kubernetes.default.svc.cluster.local
  IP.1 = 10.96.0.1
  IP.2 = 172.17.0.87

openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -out apiserver.crt  # ->apiserver.crt
```
### view crt
```
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout

### new user jane
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr # -> jane.csr
cat jane.csr | base64  # -> {result}

  apiVersion: certificates.k8s.io/v1beta1
  kind: CertificateSigningRequest
    metadata:
  name: jane
  spec:
  groups:
  - system:authenticated
  usages:
  - digital signature
  - key encipherment
  - server auth
  request:
    {result}
 
 kubectl get csr
  NAME AGE REQUESTOR CONDITION
  jane 10m admin@example.com Pending
 kubectl certificate approve jane
  jane approved!
 kubectl get csr jane -o yaml # => retrive the certificate, then decode with base64 --decode
 
 ### Kubeconf
 
 curl https://my-kube-playground:6443/api/v1/pods \
--key admin.key
--cert admin.crt
--cacert ca.crt
{
"kind": "PodList", 
"apiVersion": "v1",
"metadata": {
"selfLink": "/api/v1/pods",
},
"items": []
}

kubectl get pods
--server myapiserver:6443
--client-key admin.key
--client-certificate admin.crt
--certificate-authority ca.crt

kubectl get pods --kubeconfig config
kubectl get pods # ~/.kube/config

kubectl config view
  apiVersion: v1
  kind: Config
  current-context: kubernetes-admin@kubernetes
  clusters:
  - cluster:
  certificate-authority-data: REDACTED
  server: https://172.17.0.5:6443
  name: kubernetes
  contexts:
  - context:
  cluster: kubernetes
  user: kubernetes-admin
  name: kubernetes-admin@kubernetes
  users:
  - name: kubernetes-admin
  user:
  client-certificate-data: REDACTED
  client-key-data: REDACTED
```
### API Group
/metrics /healthz /version /api /apis /logs
```
core group /api /api/V1
named group /apis
  groups: /apis: /apps, /extensions/networking.k8s.io, /storage.k8s.io /authencations.k8s.io
  resources: (list, get, create, delete, update, watch)
  /apis/v1/deployments
          /replicases
          /statefulsets
  /apis/extensions
  /apis/networking.k8s.io/v1/networkingpolicies
  /apis/certificates.k8s.io/certificatesigningrequests
  
  1. 
   curl https://my-kube-playground:6443/api/v1/pods \
--key admin.key
--cert admin.crt
--cacert ca.crt
  2. kubectl proxy 
     Starting to server on 127.0.0.01:8080
     curl http://localhost:8080 -k  #without key,cert,cacert
### Authorization 
Mechanism: Node, ABAC, RBAC, Webhook, AlwaysAllow, AlwaysDeny
Node authorizor (kubelet)
ABAC: dev-user (view pod,create,delete pod) for every user
RBAC: developerrole(view/create/delete pod), developer in developerrole
Webhook: Open Policy Agent

--authorization-mode=AlwaysAllow  # default
--authorization-mode=Node,RBAC,WebHook  # denied then foward to the next oneRBAC
```
### RBAC
```
Role
RoleBinding
kubectl get roles
kubectl get rolebindings
kubectl auth can-i create deployments  # yes or no
kubectl auth can-i delete nodes
kubectl auth can-i create deployments --as dev-user
```

### Cluster Roles and Rolebinding
```
Cluster wide resources:
nodes, pv, clusterroles, clusterrolebindings, certficatesigingrequests, namespaces
ClusterAdmin: view/create/delete Node
StorageAdmin: view/create/delete PV
ClusterRole
ClusterRoleBinding

```
### Image Security
```
image: dockerio/nginx/nginx  (registry/user/imagerepository)

docker login private-registry.io  # supply id/pass
docker run private-registry.io/apps/myapp  

kubectl create secret docker-registry regcred \
--docker-server= private-r istry.io --docker-username=user1 --docker-password=pass1 --docker-email=abc@org.com
imagePullSecrets: regcred

```
### Security Context
```
docker run --user=1001 ubuntu sleep 3600
docker run --cap-add MAC_ADMIN ubuntu

securityContext:   # Pod: for all container. or specific container
  runAsUser: 1000
  capabilities:
    add: ["MAC_ADMIN"]  # capabilities can only for container, not pod
    
```
### Network Policy
```
#API Pod -> DB Pod -> BackUpserver 
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLables:
      role: db
  policyTypes:
  - Ingress   
  - Egress
  ingress:
  - from:
    - podSelectore:
        matchLabels:
          name: api-pod
      (namespaceSelector:
        matchLabels:
          name: prod)
    (- ipBlock: 
          cidr: 192.168.5.10/32)
    ports:
    - protocol: TCP
      port: 3306
  - to:
    - ipBlock:
        cidr: 192.168.5.11/32
      ports:
      - protocol: TCP
        port: 80
```
